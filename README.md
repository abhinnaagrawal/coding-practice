# Knowledge Base

Live site: **https://abhinnaagrawal.github.io/coding-practice/**

Personal reference site — one docsify deployment, multiple content sections. Pick a section from the sidebar.

## Sections

- **[Coding Practice](/coding-practice/README.md)** — senior/staff FAANG+ coding interview pattern reference: 19 categories, each with recognition signals, gotchas, a company-frequency-ranked problem list, and commented Python solutions.
- **[Python Tutorial](/python-tutorial/README.md)** — Python in one day for C/Java/Bash engineers: 9 chapters framed as diffs against what you already know, interview-first ordering, gotcha checklists, and micro-exercises per chapter.
- **[Numerical Methods](/numerical-methods/README.md)** — numerical methods intuition in one week: floating point, root finding, interpolation, numerical linear algebra, differentiation/integration, ODEs, and optimization, each with runnable Python and open-access deep dives.
- **[Book Summaries](/book-summaries/README.md)** — lesson-style notes working through technical books.

## How this repo is structured

```
.
├── README.md                 this file — top-level landing page
├── index.html                 docsify loader (theme, search, copy-code, Python syntax highlighting)
├── _sidebar.md                left-nav links, grouped by section
├── .nojekyll                  tells GitHub Pages to serve files as-is (skip Jekyll processing)
├── coding-practice/            section: interview prep (see coding-practice/README.md)
│   └── 01-arrays-hashing.md … 19-design-data-structures.md
├── python-tutorial/            section: Python in one day (see python-tutorial/README.md)
│   └── 01-setup-and-running-code.md … 09-advanced-decorators-context-managers-typing.md
├── numerical-methods/          section: numerical methods intuition (see numerical-methods/README.md)
│   └── 01-number-representation.md … 07-numerical-optimization.md
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
