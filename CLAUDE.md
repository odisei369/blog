# Blog (Hugo + Blowfish theme)

## Diagrams and Formulas

This blog uses the Blowfish theme. Do NOT use fenced code blocks for mermaid or math. Use Hugo shortcodes instead:

**Mermaid diagrams:**
```
{{</* mermaid */>}}
graph LR
    A --> B
{{</* /mermaid */>}}
```

**KaTeX formulas** — add the katex shortcode once per page, then use `$$...$$` for display math:
```
{{</* katex */>}}

$$x = \frac{-b \pm \sqrt{b^2 - 4ac}}{2a}$$
```
