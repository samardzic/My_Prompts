## Terminology search

Act within this constraints:
### Acronyms generation

```text
# Hardware/SoC Search Delivery Format

Use the following structure for all future hardware, semiconductor, networking, SoC, and interface-related topics.

---

## <Term Name>

<br>

Short technical definition (2-4 paragraphs max).

Explain:
- What it is
- Where it sits in the architecture
- Why it is needed
- Its primary responsibilities

<br>

Typical use cases:
- Use case 1
- Use case 2
- Use case 3
- Use case 4

<br>

**_Graph 1: <System Placement / Architecture Overview>_**

```text
ASCII Diagram
```

<br>

**_Graph 2: <Operation / Data Flow>_**

```text
ASCII Diagram
```

<br>

**_Graph 3: <Detailed Example>_**

```text
ASCII Diagram
```

<br>

**_Graph 4: <Optional Additional View>_**

```text
ASCII Diagram
```
<br>

**_Key Functions:_**
- Function 1
- Function 2
- Function 3
- Function 4

<br>

**_Key Takeaway:_**
One concise paragraph summarizing the concept.

---

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

Every diagram must have:

```markdown
**_Title_**

```text
diagram
```

<br>


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

Use for:
- Interface connections
- Protocol layering
- Bus connectivity
- Network architecture

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

Use for:
- TX paths
- RX paths
- Packet processing
- Protocol stacks

---

## 7. Use Up To Four Diagrams

Target:

- Minimum: 2 diagrams
- Typical: 3 diagrams
- Maximum: 4 diagrams

Only add more if the topic genuinely requires them.

---

## 8. Prefer Practical Hardware Examples

Examples should show:

- Real SoC blocks
- CPUs
- DMA
- Interconnects
- PHYs
- MACs
- PCIe
- DDR
- SerDes
- Switches
- Sensors
- Accelerators

Avoid overly abstract diagrams when a real hardware example exists.

---

## 9. Output Format

Deliver the entire response as:

```markdown
Normal markdown text

**_**aph 1***
```text
diagram
```

<br>

**_Graph 2_**
```text
diagram
```



The entire answer should be returned inside a **single raw markdown block** (````markdown ... ````) so it can be copied directly into documentation, Confluence, Markdown files, Git repositories, or internal hardware glossaries.

---

# Quality Target

The style should resemble:

- Semiconductor architecture documentation
- Hardware IP datasheets
- SoC design wiki pages
- Engineering onboarding materials
- Technical reference manuals

Focus on:
- Concise definitions
- Accurate terminology
- Clean ASCII diagrams
- High readability
- Copy/paste-friendly Markdown
- Minimal fluff








