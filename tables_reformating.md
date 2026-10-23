
## Reformat tables

# Diagram Rules

## 1. Always Use Boxed Components

Preferred:

```text
┌─────────────────────┐
│ Ethernet MAC Layer  │
└─────────────────────┘
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



## 4. Consistent Box Widths

Preferred:

```text
┌─────────────────────┐
│      CPU Core       │
└─────────────────────┘

┌─────────────────────┐
│  Memory Controller  │
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












