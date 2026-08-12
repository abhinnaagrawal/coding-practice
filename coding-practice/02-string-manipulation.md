[← back to index](/coding-practice/README.md)

# String Manipulation

## When to recognize it
Problem is explicitly about characters, substrings, or parsing — palindromes, anagrams, pattern matching. Usually combines with hashing (character counts) or two pointers (symmetric checks).

## Core idea
Strings are just arrays of characters — most string problems are array problems with an extra constraint (order matters, or you need character frequency). Palindrome checks are two-pointer problems in disguise; anagram checks are frequency-count problems in disguise.

## Gotchas
- Case sensitivity and non-alphanumeric characters — clarify whether to normalize (`lower()`, strip punctuation) before comparing.
- Immutability: Python strings can't be mutated in place — building a new string via list + `''.join()` is usually faster than repeated concatenation.
- Palindrome substrings: expanding-from-center handles both odd and even length palindromes — must run the expansion twice per center (once assuming odd, once assuming even).
- Anagram check via sorting is O(n log n); via frequency count is O(n) — know both, offer the O(n) one when asked to optimize.

## Problems

| Problem | Difficulty | Freq | What it asks | Intuition |
|---|---|---|---|---|
| [Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/) | Medium | 160 | Find the length of the longest substring with no repeated characters. | Sliding window with a map of last-seen index per character; when you hit a repeat, jump `left` to just past its previous occurrence. |
| [Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring/) | Medium | 93 | Find the longest substring that reads the same forwards and backwards. | Expand outward from every possible center (once treating it as odd-length, once as even-length), track the longest expansion that stays a palindrome. |
| [Valid Parentheses](https://leetcode.com/problems/valid-parentheses/) | Easy | 87 | Given a string of brackets, determine if every bracket is properly closed and nested. | Push opening brackets onto a stack; on a closing bracket, pop and check it matches — mismatch or empty-stack-pop means invalid. |
| [Valid Anagram](https://leetcode.com/problems/valid-anagram/) | Easy | 50 | Determine if two strings are anagrams of each other. | Count character frequencies of both strings and compare. |
| [Valid Palindrome](https://leetcode.com/problems/valid-palindrome/) | Easy | 50 | Determine if a string is a palindrome, ignoring non-alphanumeric characters and case. | Two pointers from both ends moving inward, skipping non-alphanumeric characters, comparing lowercased characters. |
| [Group Anagrams](https://leetcode.com/problems/group-anagrams/) | Medium | 105 | Group a list of strings into sets of anagrams. | Sorted string (or char-count tuple) as the hashmap key — anagrams collide to the same key. |
| [Palindromic Substrings](https://leetcode.com/problems/palindromic-substrings/) | Medium | — | Count how many substrings of a given string are palindromes. | Same expand-from-center technique as Longest Palindromic Substring, but increment a counter on every valid expansion instead of tracking the max. |

## Solutions

### Longest Substring Without Repeating Characters
```python
def length_of_longest_substring(s):
    seen = {}  # char -> last index seen
    left = 0
    longest = 0
    for right, ch in enumerate(s):
        if ch in seen and seen[ch] >= left:
            left = seen[ch] + 1  # jump left past the previous occurrence
        seen[ch] = right
        longest = max(longest, right - left + 1)
    return longest
```

### Longest Palindromic Substring
```python
def longest_palindrome(s):
    def expand(l, r):
        while l >= 0 and r < len(s) and s[l] == s[r]:
            l -= 1
            r += 1
        return s[l + 1:r]  # l, r overshot by one on mismatch

    result = ""
    for i in range(len(s)):
        odd = expand(i, i)       # center is a single character
        even = expand(i, i + 1)  # center is between two characters
        result = max(result, odd, even, key=len)
    return result
```

### Valid Parentheses
```python
def is_valid_parentheses(s):
    pairs = {')': '(', ']': '[', '}': '{'}
    stack = []
    for ch in s:
        if ch in pairs:
            if not stack or stack.pop() != pairs[ch]:
                return False  # mismatch or nothing to close
        else:
            stack.append(ch)
    return not stack  # every opener must have been closed
```

### Valid Anagram
```python
from collections import Counter

def is_anagram(s, t):
    return Counter(s) == Counter(t)
```

### Valid Palindrome
```python
def is_palindrome(s):
    l, r = 0, len(s) - 1
    while l < r:
        while l < r and not s[l].isalnum():
            l += 1
        while l < r and not s[r].isalnum():
            r -= 1
        if s[l].lower() != s[r].lower():
            return False
        l += 1
        r -= 1
    return True
```

### Group Anagrams
```python
def group_anagrams(strs):
    groups = {}
    for s in strs:
        key = tuple(sorted(s))
        groups.setdefault(key, []).append(s)
    return list(groups.values())
```

### Palindromic Substrings
```python
def count_palindromic_substrings(s):
    def expand(l, r):
        count = 0
        while l >= 0 and r < len(s) and s[l] == s[r]:
            count += 1
            l -= 1
            r += 1
        return count

    total = 0
    for i in range(len(s)):
        total += expand(i, i)       # odd-length palindromes centered at i
        total += expand(i, i + 1)   # even-length palindromes centered between i, i+1
    return total
```
