# Markdown formatting rules


IMPORTANT:
Use the accompanying **`markdown_rules.md`** file as the authoritative reference for processing and formatting Markdown documents.


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

---




## 7. Header Spacing

Apply the following additional spacing conventions consistently.

**H2 headings (`##`):**

Insert two HTML line breaks before every H2 heading.

Required format:
```

<br/><br/>





## H2 Heading
```

**H3 headings (`###`):**

Insert one HTML line break before every H3 heading.

Required format:

```

<br/>



### H3 Heading
```

Do not insert these spacers before the first H1 heading.

Do not add unnecessary consecutive spacers.

---



## 8. Markdown Table Formatting and Alignment Rules

Apply the following rules whenever creating or editing Markdown tables.
The goal is to make tables readable in both rendered Markdown and raw `.md` source.


### 8.1 Consistent column width and raw-source alignment

- Align all vertical pipe characters (`|`) in the raw Markdown source.
- Within each column, make every cell the same character width by padding shorter values with spaces.
- Determine each column's width from its longest cell, including the header and separator row.
- Make the separator row (`|---|`) match the width of its corresponding column.
- Apply alignment to the actual Markdown source, not only to the rendered table.
- Use spaces for padding; do not use tabs.

Prefered:
```markdown
| Component        | Description               |
|:-----------------|:--------------------------|
| Clock Controller | Controls the system clock |
| State            | Manages clock sources     |
```


Avoid:
```markdown
| Component | Description |
|------|----------|
| Clock Controller | Controls the system clock |
| State  | Manages clock sources |
```


### 8.2 Handle long cell content

- Avoid unnecessarily long cells, particularly when they contain lengthy descriptions, explanations, or several independent points.
- When practical, split a long description across multiple rows instead of creating an excessively wide table.
- If a cell contains several independent items, give each item its own row where that improves clarity.
- When continuing a value across rows, leave the repeated identifying cell blank only when the relationship remains unambiguous.
- Preserve the full meaning and all important details. Do not truncate or remove information merely to shorten a table.


Prefer splitting the description into shorter rows:

```markdown
| Component        | Description                     |
|------------------|---------------------------------|
| Clock Controller | Manages system clock sources    |
|                  | Controls clock gating           |
|                  | Supports frequency scaling      |
|                  | Coordinates clock distribution  |
```

Avoid putting several separate responsibilities into one very long cell:

```markdown
| Component        | Description |
|------------------|-------------|
| Clock Controller | Controls the system clock, manages clock sources, handles clock gating, supports frequency scaling, and coordinates clock distribution across the entire SoC. |
```


### 8.3 Preserve content and relationships

- Do not sacrifice technical accuracy or completeness for visual compactness.
- Keep related values together and make the meaning of each row clear.
- If splitting a table, repeat relevant context in the new table headings or columns so that each table can be understood independently.
- Do not split a logical record in a way that makes it unclear which values belong together.

