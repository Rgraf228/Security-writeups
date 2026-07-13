# npm Audit Batch Remediation — brace-expansion, glob, minimatch, yaml

**Date:** 2026-07-13
**Detection Method:** `npm audit`
**Status:** ✅ Remediated — All findings resolved via `npm audit fix --legacy-peer-deps`

---

## Overview

During the same `npm audit` run that surfaced the node-tar CVE cluster, four additional vulnerable packages were identified: `brace-expansion`, `glob`, `minimatch`, and `yaml`. All were resolved in the same `npm audit fix --legacy-peer-deps` command. This write-up documents all four as a batch, as each represents a lower-complexity finding relative to the node-tar cluster and the Vite CVE series — none required manual version verification or exposure assessment beyond confirming the audit output.

**Final audit result:** `found 0 vulnerabilities`

---

## Findings

### 1. brace-expansion — Zero-Step Sequence DoS

**Advisory:** GHSA-f886-m6hf-6m8v
**Severity:** Moderate
**Affected Range:** `2.0.0 - 2.0.2`
**CWE:** CWE-1333 (Inefficient Regular Expression Complexity)

`brace-expansion` is a JavaScript library that expands brace expressions like `{a,b,c}` and `{1..5}` into arrays of strings — the same syntax used in Bash and glob patterns. The vulnerability involves a zero-step numeric sequence such as `{0..0..0}`, which causes the library to enter an infinite or near-infinite expansion loop, hanging the process and exhausting memory.

Unlike a ReDoS which exploits regex backtracking, this is a direct algorithmic DoS: the library attempts to generate an expansion of zero step size, producing a degenerate loop condition. Exploitability requires untrusted user input to be passed as a brace expression — the same condition as picomatch ReDoS. In this environment, `brace-expansion` is consumed by internal build tooling processing developer-controlled patterns.

**Resolution:** Updated via `npm audit fix --legacy-peer-deps`.

---

### 2. glob — Command Injection via `-c/--cmd` Flag

**Advisory:** GHSA-5j98-mcp5-4vw2
**Severity:** High
**Affected Range:** `10.2.0 - 10.4.5`
**CWE:** CWE-78 (OS Command Injection)

`glob` is one of the most widely used file globbing libraries in the Node.js ecosystem. The vulnerability affects `glob`'s CLI interface: when invoked with the `-c` or `--cmd` flag, glob executes matched filenames as shell commands with `shell: true`. If an attacker can control the filenames being matched (e.g. by placing a maliciously named file in a directory that glob is called against), they can achieve OS command injection.

This is a CLI-specific vulnerability — it only affects applications that invoke `glob` via its command-line interface with the `-c`/`--cmd` flag, not applications using it as a library via `require('glob')`. In this environment, glob is a transitive build tooling dependency used for file discovery, not CLI command execution.

**Resolution:** Updated via `npm audit fix --legacy-peer-deps`.

---

### 3. minimatch — ReDoS (3 Advisories)

**Advisories:** GHSA-3ppc-4f35-3m26, GHSA-7r86-cg39-jmmj, GHSA-23c5-xmqv-rm74
**Severity:** High
**Affected Range:** `9.0.0 - 9.0.6`
**CWE:** CWE-1333 (Inefficient Regular Expression Complexity)

`minimatch` is a glob matching library and one of the most downloaded npm packages. Three separate ReDoS vulnerabilities were identified in the `9.0.x` branch, each exploiting a different pattern construction that leads to catastrophic backtracking:

- **GHSA-3ppc-4f35-3m26** — Repeated wildcards combined with a non-matching literal suffix in the pattern trigger exponential backtracking
- **GHSA-7r86-cg39-jmmj** — Multiple non-adjacent `**` (GLOBSTAR) segments in `matchOne()` cause combinatorial backtracking
- **GHSA-23c5-xmqv-rm74** — Nested `*()` extglobs generate catastrophically backtracking regular expressions (same extglob class as CVE-2026-33671 in picomatch, which minimatch depends on)

The third advisory (GHSA-23c5-xmqv-rm74) is directly related to CVE-2026-33671 — minimatch uses picomatch for extglob processing, and the patched picomatch `2.3.2` addresses the underlying extglob ReDoS. The other two are minimatch-native issues in its own regex generation logic.

All three share the same exploitability condition: untrusted user input passed as glob patterns. In this environment, minimatch is a build tooling transitive dependency processing developer-controlled patterns.

**Resolution:** Updated via `npm audit fix --legacy-peer-deps`.

---

### 4. yaml — Stack Overflow via Deeply Nested Collections

**Advisory:** GHSA-48c2-rrv3-qjmp
**Severity:** Moderate
**Affected Range:** `2.0.0 - 2.8.2`
**CWE:** CWE-674 (Uncontrolled Recursion)

`yaml` is a widely used YAML parsing and serialization library for Node.js. The vulnerability involves deeply nested YAML collections — objects or arrays nested to extreme depth — which cause the parser's recursive processing functions to overflow the call stack, crashing the Node.js process.

This is an uncontrolled recursion DoS: the parser does not enforce a maximum nesting depth before recursing, so a sufficiently deep structure exhausts the call stack. Exploitability requires untrusted YAML input to be parsed — a common condition in applications that accept YAML configuration files from users, API endpoints that parse YAML payloads, or CI/CD pipelines processing YAML from untrusted sources.

In this environment, `yaml` is a transitive dependency of build tooling. However, YAML parsing of untrusted input is a realistic scenario in many Node.js applications, making this worth patching regardless of current exposure.

**Resolution:** Updated via `npm audit fix --legacy-peer-deps`.

---

## Remediation

All four findings were resolved alongside the node-tar cluster in a single command:

```cmd
npm audit fix --legacy-peer-deps
```

**Result:**
```
removed 1 package, changed 6 packages, and audited 148 packages in 1s
found 0 vulnerabilities
```

The `--legacy-peer-deps` flag was required due to the pre-existing peer conflict between `@tailwindcss/vite@4.1.12` and Vite `8.1.3`.

### Final Verification

```cmd
npm audit
```

**Result:**
```
found 0 vulnerabilities
```

---

## Pattern Analysis

Across all four packages in this batch, two vulnerability classes dominate:

**ReDoS (brace-expansion, minimatch × 3):** Three of the four packages contained Regular Expression Denial of Service vulnerabilities. All share the same exploitability condition — untrusted user input as glob or brace patterns — and the same root cause class: regex patterns generated from user-supplied input without adequate safeguards against exponential backtracking. This is the same class as CVE-2026-33671 (picomatch) documented separately. The prevalence of ReDoS across the Node.js glob matching ecosystem (`picomatch`, `minimatch`, `brace-expansion`, `micromatch`) reflects a systemic issue: these libraries were historically designed for developer-controlled patterns, not adversarial input, and their regex generation logic was never hardened against crafted inputs.

**Uncontrolled Recursion (yaml):** Stack overflow via deeply nested input is a close cousin of ReDoS — both are resource exhaustion attacks exploiting algorithmic complexity. The fix pattern is the same: enforce an input complexity limit before processing.

**Command Injection (glob CLI):** The glob CLI injection is the outlier — it's a different class (OS command injection, CWE-78) and the only finding in this batch that could lead to code execution rather than denial of service. It's also the most narrowly scoped, requiring CLI invocation with a specific flag.

---

## Lessons Learned

**1. ReDoS is endemic to the Node.js glob matching ecosystem.**
Within a single `npm audit` session, ReDoS vulnerabilities were identified in `picomatch`, `minimatch`, and `brace-expansion` — three separate libraries, all in the same functional category (glob/pattern matching), all with the same root cause class. This is not coincidence: glob matching libraries were built for developer ergonomics, not security, and the extglob/quantifier patterns that make them expressive are exactly the patterns that produce catastrophic regex backtracking. Any Node.js application that exposes glob pattern input to users should be treated as a ReDoS risk surface by default.

**2. `npm audit fix` is powerful but not unconditional.**
`npm audit fix` resolved 11 findings across 5 packages in a single command. However, it required `--legacy-peer-deps` to bypass the `@tailwindcss/vite` peer conflict, and the command output included a deprecation warning about `glob@10.5.0`. Automated fixes should always be followed by functional testing — a dependency update that breaks a build tool or introduces a new incompatibility creates a different kind of problem. In this case the fix went cleanly, but the habit of verifying after automated remediation is important.

**3. Moderate severity findings still warrant remediation.**
`brace-expansion` and `yaml` were flagged as Moderate rather than High. It would be easy to deprioritize these in favor of the High severity findings. However, both have available patches and both represent realistic DoS vectors in applications that process untrusted input. When a patch is available and the upgrade is non-breaking, severity triage should inform prioritization order, not a decision to skip remediation entirely.
