---
title: "Hello World"
date: 2026-03-19
draft: false
tags: ["intro"]
summary: "First post — testing code, formulas, and diagrams."
---

## Code

```swift
func greet(_ name: String) -> String {
    return "Hello, \(name)!"
}
```

## Formula

{{< katex >}}

The quadratic formula:

$$x = \frac{-b \pm \sqrt{b^2 - 4ac}}{2a}$$

## Diagram

{{< mermaid >}}
graph LR
    A[App Launch] --> B[Load Certificate]
    B --> C{Valid?}
    C -->|Yes| D[Connect]
    C -->|No| E[Re-provision]
    E --> B
{{< /mermaid >}}
