# AWS IAM

## 30-Second Intuition

IAM is AWS's authorization system for "which AWS API calls can this identity make" — every request to any AWS service (an S3 `GetObject`, an EC2 `RunInstances`, a Lambda `Invoke`) is checked against a set of JSON policy documents attached to the caller, the target resource, and (if applicable) the caller's account's Organization, before AWS decides whether to execute it. The one fact that matters most operationally: **the evaluation is default-deny, and an explicit `Deny` anywhere in any applicable policy always wins, no matter how many other policies say `Allow`.** Unlike a ReBAC system (Zanzibar/SpiceDB, already in this KB), IAM does not traverse a relationship graph — it pattern-matches a request's action/resource/condition against a pile of static JSON documents, per request, with no persistent "access graph" to query.

---

## Resource-Layer Map

IAM is not a resource-consuming system in the CPU/memory/disk/network sense this KB usually maps — it has no data plane of its own, no storage engine, no query executor. It sits entirely in the **control plane**: every AWS service call passes through it as a policy check before the service's own resource layer (S3's disk, EC2's hypervisor, DynamoDB's partitions) ever gets touched. So instead of CPU/memory/disk/network, the right decomposition for IAM is the axes that actually vary from one IAM concept to the next:

| Axis | Role IAM plays | What it's optimizing for |
|---|---|---|
| **Evaluation surface** | Where a policy is attached and checked: identity-based (on the user/role/group making the call) vs. resource-based (on the S3 bucket, KMS key, SQS queue, etc. being called) | Letting the resource owner and the identity owner each independently express intent, without requiring one central authority to encode both sides |
| **Trust boundary** | A role's trust policy — itself a resource-based policy on the role — decides who is allowed to call `sts:AssumeRole` on it | Decoupling "who can become this identity" from "what this identity can do once assumed" |
| **Credential lifetime** | STS-issued temporary session credentials (`AccessKeyId`/`SecretAccessKey`/`SessionToken`, with an `Expiration`) vs. static long-lived IAM user access keys (no expiry, valid until manually rotated/deleted) | Minimizing blast radius — a leaked temporary credential is only useful until it expires (minutes to at most 12 hours); a leaked static key is useful indefinitely |
| **Organizational ceiling** | Permission boundaries (per-identity) and Service Control Policies (per-account/OU, via AWS Organizations) — both cap what an identity-based policy can ever grant, regardless of how permissive that policy is written | Giving a central security team a backstop that no developer-authored policy, however broad, can exceed |

This table *is* the differentiator for IAM relative to a ReBAC system like Zanzibar: Zanzibar's resource layer is Spanner storage and Check-latency fan-out because it's answering "does a graph path exist between these two nodes." IAM's "resource layer" is these four axes because it's answering "does this static JSON document set, evaluated once per request, produce an Allow" — no graph, no persistent relationship data, no traversal cost that scales with nesting depth.

---

## The Signature Mechanism

### The Policy Evaluation Algorithm: Explicit Deny Wins, Then Union of Allows, Else Default Deny

Every IAM authorization decision — for every service, every action, every resource — runs the same fixed algorithm (per current AWS documentation, [Policy evaluation logic](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html)):

1. **Default is deny.** Absent any matching statement anywhere, the request is denied. This is the *implicit* deny — there's no "everything is open unless blocked" mode in IAM.
2. **Gather every applicable policy**: identity-based policies (attached to the user/role/group), resource-based policies (attached to the resource being called, if the service supports them — S3 bucket policies, KMS key policies, SQS queue policies, Lambda resource policies, IAM role trust policies), permission boundaries (if attached to the calling identity), Organizations SCPs (if the account is in an Organization), and session policies (if passed at `AssumeRole` time).
3. **Check for an explicit `Deny`** across all of those. If even one matches, the final decision is `Deny` — full stop, no further evaluation matters.
4. **Absent any explicit deny, check for an explicit `Allow`** across identity-based and resource-based policies. This is a **union**, not a conjunction, *within a single account*: if either the identity-based policy or the resource-based policy allows the action, that's sufficient.
5. **If nothing explicitly allows it, the implicit default (deny) applies.**

Permission boundaries and SCPs don't grant anything by themselves — they only intersect against what step 4 already allowed, narrowing it. This is why the doc frames them as a "ceiling" concept, not a "grant" concept — see Deep Internals below.

This is the reason the resource-layer table above says IAM's organizational-ceiling row is about *capping*, not *granting*: a permission boundary or SCP can only ever remove permissions that an identity-based/resource-based Allow already produced, never add new ones.

---

## High-to-Low Walkthrough: Creating and Assuming a Role

This traces one concrete operation end to end: standing up a cross-account-assumable role via the CLI, then a caller actually assuming it and getting temporary credentials.

```
aws iam create-role (trust policy: who can assume this)
        │
        ▼
aws iam create-policy (permissions policy: what this role can do)
        │
        ▼
aws iam attach-role-policy (bind permissions policy to the role)
        │
        ▼
aws sts assume-role (caller in the trust policy calls this)
        │
        ▼
STS validates caller identity against the role's trust policy
        │
        ▼
STS mints temporary credentials: AccessKeyId / SecretAccessKey /
SessionToken / Expiration (default 1hr, max up to the role's
MaxSessionDuration, up to 12hr)
        │
        ▼
Caller uses those temporary creds for subsequent AWS API calls,
each of which re-runs the full policy-evaluation algorithm above
using the ROLE's identity-based policy (not the caller's original one)
```

### 1. Create the role with a trust policy

The trust policy answers "who can call `sts:AssumeRole` on this role" — it is itself a **resource-based policy attached to the role**, distinct from the permissions policy that answers "what can this role do."

```bash
cat > trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "partner-integration-2026"
        }
      }
    }
  ]
}
EOF

aws iam create-role \
  --role-name CrossAccountS3Reader \
  --assume-role-policy-document file://trust-policy.json \
  --max-session-duration 3600
```

`Principal.AWS: "arn:aws:iam::111111111111:root"` means "any identity in account `111111111111`, subject to that account's own policies also allowing the assume call" — using the account root as principal is a common cross-account pattern that delegates the actual identity-level restriction to the calling account's own IAM policies. `sts:ExternalId` is a standard hardening condition against the "confused deputy" problem when a third party is assuming the role on your behalf.

### 2. Create the permissions policy — what the role can actually do

```bash
cat > permissions-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::shared-reports-bucket",
        "arn:aws:s3:::shared-reports-bucket/*"
      ]
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name CrossAccountS3ReadOnly \
  --policy-document file://permissions-policy.json
# -> returns Policy.Arn, e.g. arn:aws:iam::222222222222:policy/CrossAccountS3ReadOnly

aws iam attach-role-policy \
  --role-name CrossAccountS3Reader \
  --policy-arn arn:aws:iam::222222222222:policy/CrossAccountS3ReadOnly
```

### 3. Assume the role and get temporary credentials

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::222222222222:role/CrossAccountS3Reader \
  --role-session-name partner-batch-job \
  --external-id partner-integration-2026 \
  --duration-seconds 3600
```

Annotated response:

```json
{
  "Credentials": {
    "AccessKeyId": "ASIAJEXAMPLEXXXXXXXX",
    "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
    "SessionToken": "IQoJb3JpZ2luX2VjE...==",
    "Expiration": "2026-09-21T18:30:00Z"
  },
  "AssumedRoleUser": {
    "AssumedRoleId": "AROAEXAMPLEID:partner-batch-job",
    "Arn": "arn:aws:sts::222222222222:assumed-role/CrossAccountS3Reader/partner-batch-job"
  }
}
```

Note the `AccessKeyId` prefix `ASIA` (temporary) vs. `AKIA` (a long-lived IAM user key) — this prefix is a real, checkable signal in credential-scanning tools. The `SessionToken` must accompany every subsequent request (it's not optional the way it would be for a static key pair), and `Expiration` is enforced server-side by STS/every service — there is no way to keep using these credentials past that timestamp.

### 4. Using the temporary credentials

```bash
export AWS_ACCESS_KEY_ID=ASIAJEXAMPLEXXXXXXXX
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
export AWS_SESSION_TOKEN=IQoJb3JpZ2luX2VjE...==

aws s3api get-object --bucket shared-reports-bucket --key q3.csv /tmp/q3.csv
```

### How EC2, Lambda, and EKS get credentials without ever calling AssumeRole

This is the mechanism that replaces static access keys in a well-run AWS environment — none of these compute services embed a key at all:

```
EC2 instance                          Lambda function                    EKS pod
     │                                       │                                │
     ▼                                       ▼                                ▼
Instance Profile                     Execution Role (set at         ServiceAccount annotated
(attached to the instance,           function config time)          with an IAM role ARN
contains an IAM role)                       │                       (IRSA) or a Pod Identity
     │                                       ▼                       association (newer)
     ▼                              Lambda service itself                    │
Instance Metadata Service            calls sts:AssumeRole                    ▼
(IMDSv2, 169.254.169.254)             on your behalf before          Pod's projected service-
returns temp creds, auto-             invoking your code,             account OIDC token is
rotated well before expiry            injects temp creds as           exchanged (via EKS's
     │                                env vars                        OIDC provider, IRSA) or
     ▼                                       │                        via the EKS Pod Identity
SDK inside the instance                       ▼                        Agent (newer, simpler)
picks these up automatically          SDK inside the function          for temp STS creds
via the default credential            picks up AWS_ACCESS_KEY_ID/            │
provider chain — no                   SECRET/SESSION_TOKEN env vars          ▼
AssumeRole call in app code           automatically                   SDK inside the pod
                                                                       picks up creds via
                                                                       AWS_WEB_IDENTITY_TOKEN_FILE
                                                                       (IRSA) or the Pod Identity
                                                                       Agent's local endpoint
```

Check what an EC2 instance actually sees (from inside the instance, IMDSv2):

```bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/

# then, with the role name returned above:
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/CrossAccountS3Reader
# -> JSON with AccessKeyId/SecretAccessKey/Token/Expiration, auto-rotated by AWS
```

---

## Deep Internals

### Identity-based vs. resource-based: the same-account vs. cross-account asymmetry

Within a **single account**, step 4 of the evaluation algorithm is a union: either the identity-based policy on the caller *or* the resource-based policy on the resource allowing the action is sufficient.

Across **two accounts**, this changes — it becomes a requirement for **both**:

```
Same-account S3 access:
  User's identity policy: Allow s3:GetObject         }
  Bucket policy: (silent / no statement)              }-> effective: ALLOWED (identity policy alone suffices)

Cross-account S3 access:
  Account A user's identity policy: Allow s3:GetObject on arn:aws:s3:::bucket-in-account-b/*
  Account B bucket policy: (silent / no statement granting Account A)
                                                       }-> effective: DENIED
                                                          (resource owner never opted in)

Cross-account S3 access, done correctly:
  Account A user's identity policy: Allow s3:GetObject on arn:aws:s3:::bucket-in-account-b/*
  Account B bucket policy: Allow Principal arn:aws:iam::ACCOUNT-A:root, s3:GetObject
                                                       }-> effective: ALLOWED (both sides explicitly opted in)
```

Worked bucket policy for the correct cross-account case:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::111111111111:root" },
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::bucket-in-account-b",
        "arn:aws:s3:::bucket-in-account-b/*"
      ]
    }
  ]
}
```

The reasoning: within one account, AWS trusts the account owner to have set both policies consistently, so either side opting in is enough — a single owner can't meaningfully "attack" their own resources this way. Across accounts, the resource owner (Account B) has no visibility into Account A's identity policies, so AWS requires the *resource owner's own policy* to explicitly name the foreign account/principal — otherwise Account A could grant itself access to any bucket in the world just by writing a permissive identity policy naming someone else's ARN.

### Worked evaluation trace #1: explicit deny wins regardless of an identity allow

```
Identity policy on user Alice:
  Allow s3:GetObject on arn:aws:s3:::reports-bucket/*

Bucket policy on reports-bucket:
  Deny s3:GetObject unless aws:SourceVpce == "vpce-0123456789abcdef0"
  (i.e. Deny s3:GetObject when NOT StringEquals aws:SourceVpce vpce-0123...)
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyOutsideVPC",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::reports-bucket/*",
      "Condition": {
        "StringNotEquals": { "aws:SourceVpce": "vpce-0123456789abcdef0" }
      }
    }
  ]
}
```

Trace for Alice calling `GetObject` from outside that VPC endpoint:

1. Gather applicable policies: Alice's identity policy (Allow), the bucket policy (Deny when condition matches).
2. Check for explicit deny first: the bucket policy's `Deny` statement's condition evaluates true (Alice is not coming from `vpce-0123...`) → **explicit deny found**.
3. Evaluation stops here. Final decision: **Deny.** Alice's identity-policy `Allow` never even gets to matter — it is not "outvoted," it is simply irrelevant once an explicit deny fires.

### Worked evaluation trace #2: two independent allows are additive, not conjunctive

```
Identity policy on user Bob:
  Allow s3:GetObject on arn:aws:s3:::reports-bucket/*

Bucket policy on reports-bucket:
  Allow s3:GetObject to Principal arn:aws:iam::ACCOUNT:user/Bob
```

Trace:

1. No explicit deny anywhere.
2. Check for explicit allow: identity policy allows it — **already sufficient**. The bucket policy also allowing it is redundant, not required. Either one alone would have produced Allow.
3. Final decision: **Allow.**

This is the core asymmetry to hold in your head: **deny is a veto (any one is enough to kill the request); allow is a union (any one is enough to grant it) — but only within a single account.** Cross-account, allow requires both sides, as shown above.

### Permission boundaries: the intersection, not the grant

A permission boundary is a **managed policy attached to a user or role** that sets the ceiling on what that identity's *own* attached policies can grant — it is never itself a source of permissions.

```bash
cat > admin-permissions.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": "iam:*", "Resource": "*" }
  ]
}
EOF

cat > s3-only-boundary.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": "s3:*", "Resource": "*" }
  ]
}
EOF

aws iam create-policy --policy-name AdminPermissions --policy-document file://admin-permissions.json
aws iam create-policy --policy-name S3OnlyBoundary --policy-document file://s3-only-boundary.json

aws iam create-role --role-name BoundedAdmin --assume-role-policy-document file://trust-policy.json
aws iam attach-role-policy --role-name BoundedAdmin --policy-arn arn:aws:iam::ACCOUNT:policy/AdminPermissions
aws iam put-role-permissions-boundary --role-name BoundedAdmin --permissions-boundary arn:aws:iam::ACCOUNT:policy/S3OnlyBoundary
```

`BoundedAdmin`'s attached identity policy says `iam:*` (looks like full IAM admin). Its permission boundary says `s3:*` only. Effective permission = **intersection** = only S3 actions. A call to `iam:CreateUser` fails even though the attached policy explicitly allows it, because the boundary never granted `iam:` actions in the first place — the boundary can only cap what's already allowed, it can't be exceeded by a broader identity policy, and it can't grant anything the identity policy didn't already allow either.

### SCPs: the same ceiling concept, at the Organization level

Service Control Policies apply to entire AWS accounts or OUs via AWS Organizations — same "ceiling, not grant" logic as permission boundaries, but organization-wide instead of per-identity. Per AWS's own account-level evaluation: when an account is a member of an Organization, "the resulting permissions are the intersection of the user's policies and the SCP... an action must be allowed by both the identity-based policy and the SCP. An explicit deny in either overrides the allow."

```bash
cat > deny-outside-approved-regions.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyOutsideApprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "iam:*", "sts:*", "support:*", "organizations:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["us-east-1", "us-west-2"]
        }
      }
    }
  ]
}
EOF

aws organizations create-policy \
  --name DenyOutsideApprovedRegions \
  --type SERVICE_CONTROL_POLICY \
  --content file://deny-outside-approved-regions.json

aws organizations attach-policy \
  --policy-id p-exampleid \
  --target-id ou-root-exampleouid
```

No identity or role in any account under `ou-root-exampleouid` can ever act outside `us-east-1`/`us-west-2`, regardless of what any account's own IAM admin grants — this is precisely why SCPs are the tool for org-wide guardrails that individual account admins cannot override. (AWS Organizations has also added **Resource Control Policies (RCPs)** as a newer, parallel org-wide ceiling that constrains resource-based policies rather than identity-based ones — RCPs are the resource-side analog to SCPs; check current AWS Organizations docs before relying on RCP-specific claims, since this is a comparatively newer feature relative to the rest of this doc's grounding.)

### IRSA vs. EKS Pod Identity: two ways a pod gets credentials without a static key

**IRSA (IAM Roles for Service Accounts)**, introduced 2019, uses OIDC federation:

1. Each EKS cluster has an associated OIDC identity provider registered in IAM.
2. A Kubernetes ServiceAccount is annotated with an IAM role ARN.
3. The EKS control plane injects a projected, short-lived OIDC token into the pod (mounted file, `AWS_WEB_IDENTITY_TOKEN_FILE`).
4. The AWS SDK inside the pod calls `sts:AssumeRoleWithWebIdentity`, presenting that token; STS validates it against the role's trust policy, which conditions on the OIDC provider's `sub` claim (`system:serviceaccount:<namespace>:<serviceaccount>`).

```bash
eksctl utils associate-iam-oidc-provider --cluster my-cluster --approve

cat > irsa-trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:sub": "system:serviceaccount:default:my-app-sa",
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
EOF

aws iam create-role --role-name my-app-irsa-role --assume-role-policy-document file://irsa-trust-policy.json

kubectl annotate serviceaccount my-app-sa \
  -n default \
  eks.amazonaws.com/role-arn=arn:aws:iam::ACCOUNT:role/my-app-irsa-role
```

**EKS Pod Identity**, shipped November 2023, replaces the OIDC-token-exchange dance with an AWS-managed component:

1. An **EKS Pod Identity Agent** (a DaemonSet) runs on every node.
2. You create a Pod Identity **association** directly linking a namespace/ServiceAccount to an IAM role — no OIDC provider setup, no per-cluster trust-policy `sub`-claim string to get right.
3. The agent handles the credential exchange locally; the pod's SDK calls a local endpoint instead of doing `AssumeRoleWithWebIdentity` itself.

```bash
aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace default \
  --service-account my-app-sa \
  --role-arn arn:aws:iam::ACCOUNT:role/my-app-pod-identity-role
```

The Pod Identity role's trust policy is simpler and standardized (trusts the EKS Pod Identity service principal, `pods.eks.amazonaws.com`) rather than a per-cluster OIDC-provider ARN with a hand-written `sub` condition:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "pods.eks.amazonaws.com" },
      "Action": ["sts:AssumeRole", "sts:TagSession"]
    }
  ]
}
```

As of current (2026) AWS guidance, EKS Pod Identity is the recommended default for new standard EC2-backed clusters — it removes the OIDC-provider bootstrapping step and the most common IRSA misconfiguration (a loosely scoped `sub` condition). IRSA remains fully supported with no deprecation date, and is still the right choice for Fargate profiles, Windows nodes, and EKS Anywhere/hybrid topologies where the Pod Identity Agent daemonset model doesn't apply the same way. Migrate because a specific requirement (ABAC via session tags, cross-account role assumption ergonomics, simpler Terraform) calls for it — not by default.

---

## Comparative: IAM vs. Zanzibar/SpiceDB ReBAC

This KB already covers Zanzibar and SpiceDB (`/systems-engineering/authz/zanzibar.md`, `/systems-engineering/authz/spicedb.md`) in depth — the short version for cross-reference rather than re-explanation:

| | IAM | Zanzibar/SpiceDB (ReBAC) |
|---|---|---|
| Unit of data | Static JSON policy documents (identity-based, resource-based, boundary, SCP) | Relationship tuples (`object#relation@subject`) in a live, queryable store |
| Evaluation model | Pattern-match action/resource/condition against all applicable documents, per request, no persistent graph | Recursive graph traversal (`Check`) over unions/intersections/arrow-hops through stored relationships |
| What it answers well | "Which AWS API calls can this identity make" — coarse-to-medium grained, resource-pattern-based (ARNs, wildcards, conditions on request context) | "Does user X have access to this specific document via this specific sharing chain" — fine-grained, per-object, inheritance-aware |
| Consistency concern | None analogous to the "new enemy" problem — policies are evaluated fresh per request against the current document set, not against a replicated tuple store with propagation delay | Central concern (zookies/ZedTokens exist specifically to prevent a stale revocation from being missed) |
| Ceiling mechanism | Permission boundaries (per-identity) and SCPs (per-org) as intersecting caps | No direct analog — Zanzibar/SpiceDB have no built-in concept of "org-wide permission ceiling" separate from the schema/relationship data itself |

The genuinely different fit: IAM is right for "can this service principal call this API," where the space of resources is comparatively small (accounts, buckets, tables, functions) and access is granted at the level of AWS-defined actions and ARN patterns. ReBAC is right for "can this end-user see this specific document," where the resource count is unbounded (millions of user-created documents) and access follows an object-ownership/sharing graph that AWS's own IAM model was never built to represent — you would not model "Alice shared this specific Google-Doc-style document with Bob" as an IAM policy, and you would not model "which S3 buckets can this Lambda read" as a Zanzibar relationship graph. Both are called "authorization"; they solve different-shaped problems.

---

## Key Gotchas

- **Cross-account access requires BOTH sides to allow — the single most common cross-account debugging trap.** "I gave the role `s3:*` and it still can't read the bucket in the other account" almost always means the *other account's* bucket policy never granted the calling account/role as a principal. Fix by checking the resource-based policy on the far side, not by widening the identity policy further.
- **A broad `Deny` written to restrict one specific thing can silently blackhole unrelated access.** A statement meant to say "deny writes from outside the VPC" but written with `"Action": "s3:*"` instead of `"Action": "s3:PutObject"` also denies reads, and because explicit deny always wins, no other policy can override it — audit the `Action`/`Resource`/`Condition` scoping on every `Deny` statement, not just its intent.
- **Permission boundaries are invisible in the normal console view of a role's attached policies.** Someone reviewing `BoundedAdmin`'s attached policy sees `iam:*` and reasonably assumes full IAM admin — the boundary that silently caps it to `s3:*` lives in a separate tab/API call (`get-role` → `PermissionsBoundary`) that's easy to forget to check.
- **Long-lived IAM user access keys are still fully issuable and remain a real liability** if IAM Identity Center / role-based access isn't enforced org-wide — a static `AKIA...` key pair has no expiry and, once leaked (committed to a repo, logged, cached in a CI artifact), is valid until someone notices and manually deactivates it. Current AWS guidance treats eliminating them for human users as a 2026 baseline, not an aspirational goal.
- **IRSA trust policies scoped too loosely let any pod in the cluster assume the role.** The condition must pin `oidc-provider:sub` to `system:serviceaccount:<namespace>:<serviceaccount>` exactly; a common misconfiguration uses only the `aud` condition (or a wildcard-y `sub`), which lets any workload with any service account in the cluster — not just the intended one — successfully call `AssumeRoleWithWebIdentity` and inherit the role's permissions.

---

## Open items to confirm

- **RCPs (Resource Control Policies)**: mentioned above as a newer, resource-side analog to SCPs. This is a comparatively recent AWS Organizations feature relative to the rest of this doc's grounding — verify current scope/GA status against AWS's own Organizations documentation before treating any RCP-specific claim as settled.
- **EKS Pod Identity vs. IRSA "recommended default" framing**: search results consistently describe Pod Identity as the recommended default for *new, standard EC2-backed* clusters as of 2026, with IRSA remaining fully supported (no deprecation date) for Fargate/Windows/hybrid topologies — this doc states it that way, but it's worth a periodic recheck since AWS could tighten this guidance further.
