---
title: 'Hello, World'
date: 2026-05-01T00:00:00.000Z
summary: A first Quarto musing — Python code running on the site.
tags: []
jupyter: volcan
---


This page is rendered from a [Quarto](https://quarto.org) document.
The Python code below runs at build time; the output is baked into the page.

``` python
msg = "Hello, World!"
print(msg)
```

    Hello, World!

``` python
import math

for n in range(1, 6):
    print(f"  {n}! = {math.factorial(n)}")
```

      1! = 1
      2! = 2
      3! = 6
      4! = 24
      5! = 120

Simple as that.
