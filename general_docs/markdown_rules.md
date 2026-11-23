# Markdown formatting rules

Deliver the COMPLETE contents of the input content or given target as RAW Markdown source.

STRICT OUTPUT REQUIREMENTS:

- Return the ENTIRE file from the first character to the last character.
- Do NOT summarize, shorten, omit, rewrite, interpret, or reorganize anything.
- Preserve ALL original Markdown syntax and content exactly as it appears in the file.
- This includes:
  - headings
  - paragraphs
  - lists
  - tables
  - links
  - images
  - Mermaid diagrams
  - ASCII/Unicode diagrams
  - HTML
  - block-quotes
  - horizontal rules
  - YAML/front matter
  - inline code
  - fenced code blocks
  - comments
  - whitespace and indentation where significant
- Do NOT render Markdown elements. Show their raw Markdown source.
- Do NOT extract tables, diagrams, code, or other elements into separate rendered sections.
- Do NOT add explanations, introductions, summaries, notes, or commentary.

OUTPUT FORMAT:
Return everything as ONE single copyable code block.

The ONE code block must contain the COMPLETE raw contents of input content.

IMPORTANT:
The file itself may contain fenced code blocks using ```.

To prevent those internal fences from terminating the outer code block, wrap the COMPLETE file using an outer fence that is longer
than any fence contained in the file, for example four back-ticks:

```text
<complete raw contents of visa_scpi_ivi_guide.md, including any internal ``` fences>




# Diagram Rules

## 1. Always Use Boxed Components

Preferred:

```text
┌──────────────────────┐
│  Ethernet MAC Layer  │
└──────────────────────┘
```

Avoid:

```text
Ethernet MAC Layer
```

---

## 2. Use Unicode Box Drawing Characters

Use:

```text
┌ ┐ └ ┘ │ ─ ├ ┤ ┬ ┴ ┼ ▼ ▲ ◄ ►
```

Avoid:

```text
+----+
|    |
+----+
```

---

## 3. Separate Every Diagram

Never place multiple unrelated diagrams inside the same code block.

---



## 4. Consistent Box Widths

Preferred:

```text
┌─────────────────────┐
│       CPU Core      │
└─────────────────────┘

┌─────────────────────┐
│ Memory Controller   │
└─────────────────────┘
```

Avoid:

```text
┌───┐
│CPU│
└───┘

┌────────────────────────────┐
│ Memory Controller          │
└────────────────────────────┘
```

---

## 5. Horizontal Layout for Connectivity

Preferred:

```text
┌──────────────────┐                           ┌──────────────────┐
│       MAC        │ ───────────────────────►  │       PHY        │
└──────────────────┘                           └──────────────────┘
```

---



## 6. Vertical Layout for Processing Flows

Preferred:

```text
┌──────────────────┐
│    Software      │
└──────────────────┘
          │
          ▼
┌──────────────────┐
│       MAC        │
└──────────────────┘
          │
          ▼
┌──────────────────┐
│       PHY        │
└──────────────────┘
```

