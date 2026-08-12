# Knowledge Base

Live site: **https://abhinnaagrawal.github.io/coding-practice/**

Personal reference site — one docsify deployment, multiple content sections. Pick a section from the sidebar.

## Sections

- **[Coding Practice](coding-practice/README.md)** — senior/staff FAANG+ coding interview pattern reference: 19 categories, each with recognition signals, gotchas, a company-frequency-ranked problem list, and commented Python solutions.
- **[Book Summaries](book-summaries/README.md)** — lesson-style notes working through technical books.

## How this repo is structured

```
.
├── README.md                 this file — top-level landing page
├── index.html                 docsify loader (theme, search, copy-code, Python syntax highlighting)
├── _sidebar.md                left-nav links, grouped by section
├── .nojekyll                  tells GitHub Pages to serve files as-is (skip Jekyll processing)
├── coding-practice/            section: interview prep (see coding-practice/README.md)
│   └── 01-arrays-hashing.md … 19-design-data-structures.md
└── book-summaries/             section: book notes (see book-summaries/README.md)
    └── deep-learning-book/
        └── 01-introduction.md … 06-machine-learning-basics-part2.md
```

Adding a new section means: a new top-level folder, its own `README.md` landing page, an entry in `_sidebar.md`, and a link from this file.

## How it's hosted

Static site, **GitHub Pages**, free tier:
- No build step — [docsify](https://docsify.js.org/) renders the `.md` files client-side via CDN-loaded JS (`index.html` is the only "code" in this repo).
- `_sidebar.md` drives the left-nav; `loadSidebar: true` in `index.html` wires it up.
- Pages is configured to deploy from the `main` branch, root folder (Settings → Pages).
- To update: edit a `.md` file, commit, push to `main` — Pages redeploys automatically within ~1 minute, no action needed.
