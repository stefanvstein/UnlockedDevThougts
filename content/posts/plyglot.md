---
title: "Hello World Polyglot in Brainf*ck and Whitespace"
date: 2013-04-09
tags: ["Art"]
---
A Hello World Polyglot in Brainf*ck and Whitespace

Just to follow tradition—and use obscure languages—here’s a “Hello World!” implemented in both Brainf*ck and Whitespace at the same time.

Personally, I don’t find either BF or WS particularly obscure. They’re just languages with annoyingly difficult syntax, but very simple semantics. Still, it gets a bit obscure when code for two languages is mixed. In this case though, it’s barely noticeable—since all alphanumeric characters are treated as comments, and only the rest (tabs, spaces, and line feeds) are actual code.

Whitespace is a surprisingly expressive language, especially compared to Brainf*ck—even though it only consists of three characters. Its instructions are made up of specific sequences of those characters.

```bash
> wspace hai.ws.bf
Hello World!
> bf hai.ws.bf
Hello World!
```

[![Link to hai.ws.bf](/images/polyglot.jpg)](/downloads/hai.ws.bf)
