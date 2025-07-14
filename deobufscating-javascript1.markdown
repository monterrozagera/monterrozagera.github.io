---
title: Identifying Malicious Behavior
permalink: /deobufscating-javascript1.html
---

# Understanding JavaScript Deobfuscation

## What is JavaScript Deobfuscation?

JavaScript **deobfuscation** is the process of **reverse-engineering obfuscated (intentionally scrambled or disguised) JavaScript code** to understand its true functionality. It’s a critical skill in malware analysis, reverse engineering, and web application security.

Many threat actors use obfuscation to:

* Hide malicious behavior (e.g. data theft, XSS).
* Avoid detection by antivirus or firewalls.
* Slow down reverse engineering.

---

## Why is JavaScript Obfuscated?

Here’s an example of **obfuscated code**:

```js
(function(_0xabc1){var _0x1c2d=["log","Hello\x20World!"];console[_0x1c2d[0]](_0x1c2d[1]);})();
```

This is **confusing on purpose**, but all it does is:

```js
console.log("Hello World!");
```

---

## Tools & Techniques for Deobfuscation

Deobfuscation is both **art and science**. Here are common methods and tools used:

### 1. Manual Analysis

Use your brain + `console.log()` to observe behavior step-by-step.

### 2. Pretty-Printing (Beautifying)

Use tools like:

* [Prettier](https://prettier.io/)
* [JSBeautifier](https://beautifier.io/)
* VS Code Format Document

Before:

```js
(function(a){a();})(function(){alert('hi')});
```

After:

```js
(function(a) {
  a();
})(function() {
  alert('hi');
});
```

### 3. Variable Renaming

Rename misleading variables like `_0xabc1`, `_x12`, etc., into something readable.

```js
// Original
var _0x1c2d = ["log", "Data stolen"];
console[_0x1c2d[0]](_0x1c2d[1]);

// After rename
var actions = ["log", "Data stolen"];
console[actions[0]](actions[1]);
```

### 4. Evaluate and Rewrite

Use the browser console or Node.js to evaluate:

```js
eval("console.log('Secret');")
```

Replace `eval()` with a direct value when possible:

```js
// Instead of:
eval("var x = 1 + 2;");
// Deobfuscate to:
var x = 3;
```

---

## Real-World Example

### Obfuscated JavaScript (typical web skimmer):

```js
var _0xabc1 = ["fromCharCode", "charCodeAt"];
var secret = "hftu".split('').map(c => String[_0xabc1[0]](c[_0xabc1[1]]() - 1)).join('');
console.log(secret);
```

### Step-by-step:

1. `fromCharCode` and `charCodeAt` are used dynamically.
2. `"hftu"` is `'test'`, but obfuscated by incrementing each char code by 1.
3. Deobfuscated:

```js
var secret = "test";
console.log(secret);
```

---

## Deobfuscation Code Snippet

Here’s a simple script to help deobfuscate hex-encoded strings:

```js
function hexDecode(str) {
  return str.replace(/\\x([0-9A-Fa-f]{2})/g, (_, hex) =>
    String.fromCharCode(parseInt(hex, 16))
  );
}

const input = "\\x61\\x6C\\x65\\x72\\x74\\x28\\x27\\x68\\x69\\x27\\x29";
console.log(hexDecode(input)); // outputs: alert('hi')
```

---

## Use Cases in Security

* Malware deobfuscation (malicious scripts hidden on websites).
* Understanding web skimmers (Magecart).
* Reverse engineering browser exploits.
* Auditing third-party scripts.
* Detecting crypto miners or iframe injections.

---

## Dangers of `eval()` and `Function()`

Obfuscated code often uses:

```js
eval("code here");
new Function("code");
```

Always **replace these dynamically** with static equivalents or neutralize them:

```js
// BAD
eval("console.log('hidden');");

// GOOD
console.log("hidden");
```

---

## Recommended Tools

| Tool                                              | Purpose                                       |
| ------------------------------------------------- | --------------------------------------------- |
| [JSNice](http://www.jsnice.org/)                  | Rename variables intelligently                |
| [Beautifier.io](https://beautifier.io/)           | Format ugly JavaScript                        |
| Chrome DevTools                                   | Debug and step through code                   |
| [AST Explorer](https://astexplorer.net/)          | Analyze and transform JavaScript syntax trees |
| [Decompiler](https://lelinhtinh.github.io/de4js/) | Decode `eval()`, `String.fromCharCode`, etc.  |

---

## Final Thoughts

JavaScript deobfuscation is an essential skill for anyone in:

* Malware analysis
* Application security
* Threat hunting
* Bug bounty

It requires **patience, curiosity, and tooling**, but over time you’ll develop an instinct for spotting patterns and unraveling even the nastiest code.

---