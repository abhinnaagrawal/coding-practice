[← back to index](README.md)

# Trie

## When to recognize it
"Prefix," "autocomplete," "dictionary of words," or word search combined with a fixed word list — anything where you repeatedly check "does any word start with this prefix."

## Core idea
A tree where each path from root to a marked node spells a word, and shared prefixes share the same path. Lookup/insert is O(word length), independent of how many words are stored — much better than scanning a word list.

## Gotchas
- Use a `dict` of children, not a fixed 26-array, unless the alphabet is guaranteed lowercase a-z.
- Mark "end of word" explicitly — a node existing isn't the same as a complete word ending there (e.g. "car" existing as a path doesn't mean "car" was inserted, vs. being a prefix of "cart").

## Problems

| Problem | Difficulty | Freq | What it asks | Intuition |
|---|---|---|---|---|
| [Implement Trie (Prefix Tree)](https://leetcode.com/problems/implement-trie-prefix-tree/) | Medium | — | Design a trie supporting insert, search (exact word), and startsWith (prefix). | Each node holds a dict of children keyed by character plus an "is end of word" flag; insert walks/creates nodes character by character, search/startsWith walk and check existence (search additionally checks the end-of-word flag). |
| [Design Add and Search Words Data Structure](https://leetcode.com/problems/design-add-and-search-words-data-structure/) | Medium | — | Same as Trie, but search supports a wildcard `.` matching any character. | Same trie structure as above; search becomes a DFS that, on encountering `.`, branches into *all* children instead of just one. |
| [Word Search II](https://leetcode.com/problems/word-search-ii/) | Hard | — | Given a grid and a list of words, find all words present in the grid via adjacent letters (Trie + backtracking). | Build a trie of all target words, then DFS from every grid cell following the trie's structure — prune a branch immediately once the current path no longer matches any trie node. |
| [Word Break](https://leetcode.com/problems/word-break/) | Medium | — | Determine if a string can be segmented into dictionary words (trie speeds up the dictionary lookup). | Same DP as in the DP section, but store the dictionary as a trie so each substring-membership check during the DP transition is O(word length) instead of a hash lookup with string slicing overhead. |
| [Design Search Autocomplete System](https://leetcode.com/problems/design-search-autocomplete-system/) | Hard | — | Design an autocomplete system that returns top-3 historical sentences matching the current input prefix. | Store past sentences with frequency counts in a trie (or hashmap keyed by prefix); as the user types, narrow down candidates matching the current prefix and rank by frequency then lexicographic order. |

## Solutions

```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False
```

### Implement Trie (Prefix Tree)
```python
class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word):
        node = self.root
        for ch in word:
            node = node.children.setdefault(ch, TrieNode())
        node.is_end = True

    def search(self, word):
        node = self._find(word)
        return node is not None and node.is_end

    def startsWith(self, prefix):
        return self._find(prefix) is not None

    def _find(self, word):
        node = self.root
        for ch in word:
            if ch not in node.children:
                return None
            node = node.children[ch]
        return node
```

### Design Add and Search Words Data Structure
```python
class WordDictionary:
    def __init__(self):
        self.root = TrieNode()

    def addWord(self, word):
        node = self.root
        for ch in word:
            node = node.children.setdefault(ch, TrieNode())
        node.is_end = True

    def search(self, word):
        def dfs(node, i):
            if i == len(word):
                return node.is_end
            ch = word[i]
            if ch == '.':
                return any(dfs(child, i + 1) for child in node.children.values())  # try all children
            if ch not in node.children:
                return False
            return dfs(node.children[ch], i + 1)

        return dfs(self.root, 0)
```

### Word Search II
```python
def find_words(board, words):
    root = TrieNode()
    for w in words:
        node = root
        for ch in w:
            node = node.children.setdefault(ch, TrieNode())
        node.is_end = True
        node.word = w  # stash the full word at its terminal node for easy retrieval

    rows, cols = len(board), len(board[0])
    res = []

    def dfs(r, c, node):
        ch = board[r][c]
        if ch not in node.children:
            return  # prune: no word in the trie continues this way
        nxt = node.children[ch]
        if nxt.is_end:
            res.append(nxt.word)
            nxt.is_end = False  # avoid adding the same word twice

        board[r][c] = '#'  # mark visited
        for dr, dc in ((1, 0), (-1, 0), (0, 1), (0, -1)):
            nr, nc = r + dr, c + dc
            if 0 <= nr < rows and 0 <= nc < cols:
                dfs(nr, nc, nxt)
        board[r][c] = ch  # backtrack

    for r in range(rows):
        for c in range(cols):
            dfs(r, c, root)
    return res
```

### Word Break
See [15-dynamic-programming.md](15-dynamic-programming.md) — the trie speeds up the dictionary-membership check in the DP transition.

### Design Search Autocomplete System
```python
from collections import Counter

class AutocompleteSystem:
    def __init__(self, sentences, times):
        self.counts = Counter(dict(zip(sentences, times)))
        self.current = ""

    def input(self, c):
        if c == '#':
            self.counts[self.current] += 1  # commit the typed sentence, bump its frequency
            self.current = ""
            return []
        self.current += c
        matches = [s for s in self.counts if s.startswith(self.current)]
        matches.sort(key=lambda s: (-self.counts[s], s))  # frequency desc, then lexicographic
        return matches[:3]
```
