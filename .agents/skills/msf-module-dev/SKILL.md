---
name: msf-module-dev
description: "Develop and review Metasploit Framework modules using current Rapid7 conventions. Use when creating or fixing remote, local, auxiliary, scanner, or post modules; responding to PR feedback; validating metadata, AutoCheck, payloads, reporting, cleanup, documentation, msftidy, RuboCop, or manual QA."
---

# Metasploit Framework module development

Use the current checkout as the source of truth. Treat this file as a workflow and routing guide, not a frozen copy of repository policy.

## Establish authority

Before inspecting examples or editing:

1. Read every `AGENTS.md` governing the files in scope.
2. Apply evidence in this order:
   - the user's requested outcome and applicable repository instructions;
   - current framework source, mixins, configuration, custom cops, specs, templates, and official documentation;
   - recent Rapid7 review feedback and recently merged code;
   - existing modules, only after verifying that their APIs and patterns remain current.
3. Follow the higher-authority evidence when this package disagrees with the checkout, flag stale package guidance, and surface a direct conflict between the request and repository instructions.
4. Separate observations from conclusions. Do not claim success, repeatability, test coverage, or a root cause without evidence.
5. Treat public analyses, proof-of-concept code, and output from other agents or AI tools as leads rather than proof. Reproduce material claims against the current checkout and the target's real integration path before relying on them.

## Load only relevant detail

Read every row matching the task; do not preload unrelated references.

| Task or code in scope | Read |
| --- | --- |
| PR review, reviewer feedback, rebase, regression investigation, manual testing, or final QA | [review-qa.md](./references/review-qa.md) |
| New module, substantial restructuring, state-changing flow, or need for annotated patterns | [module-template.md](./references/module-template.md) |
| `check`, CheckCode, AutoCheck, ForceExploit, or state reuse between check and execution | [module-template.md](./references/module-template.md); use [review-qa.md](./references/review-qa.md) for the test matrix |
| Metadata, Notes, Rank, options, actions, targets, architecture, payloads, or DefangedMode | [metadata.md](./references/metadata.md) |
| `HttpClient`, URIs, requests, cookies, redirects, multipart data, parsing, or timeouts | [http-client.md](./references/http-client.md) |
| Services, vulnerabilities, credentials, logins, loot, tables, or database-disabled behavior | [reporting.md](./references/reporting.md) |
| Scanner, raw TCP, LoginScanner, protocol mixins, local/post modules, remote files, process execution, or generated fetch commands | [non-http-modules.md](./references/non-http-modules.md) |
| New or changed module documentation, vulnerable-lab setup, Scenarios, or docs validation | [documentation-template.md](./references/documentation-template.md) |

## Workflow

1. Define the requested outcome and classify the work as remote exploit, local exploit, auxiliary, post, or framework library code. Check for an existing module or open pull request before creating one.
2. Inspect every included or prepended mixin and the current implementation/specs of each API whose behavior matters. Confirm inherited options, lifecycle hooks, reporting ownership, and return contracts.
3. Trace the complete lifecycle: identification, `check`, exploitation or gathering, reporting, each mutation and failure boundary, cleanup, immediate rerun, and documentation.
4. Define the evidence required for every `CheckCode`, successful mutation, reported finding, and claimed exploit result before implementing the path.
5. Make the smallest coherent change, preserve unrelated work, and keep code, metadata, tests, and documentation synchronized. Do not commit, push, publish, or alter a pull request unless explicitly authorized.
6. Validate from the source checkout, then report exactly what was and was not tested.

## Cross-cutting invariants

- Do not re-register inherited options, duplicate mixin-owned reporting, or reimplement an existing helper.
- Use `Msf::Exploit::SQLi` for SQL injection rather than implementing the primitive from scratch.
- Do not submit opaque binaries or blobs. Include source and reproducible build instructions for every generated binary artifact.
- Make every expected path through `check`, including handled failures from known helpers, return a human-readable `CheckCode`; never use `fail_with` there. Rescue only exceptions the called helper or parser is documented to raise. Exploit subclasses may use bare `CheckCode`; auxiliary, post, and evasion modules must use `Exploit::CheckCode` or `Msf::Exploit::CheckCode`.
- Use `Safe` only when evidence disproves applicability, `Unknown` when assessment was prevented, `Detected` for product identification without exploitability evidence, `Appears` for a verified vulnerable range, and `Vulnerable` only when safely exercising the vulnerability produced direct, vulnerability-specific evidence. Reserve `Unsupported` for cases where automated checking itself is not implemented or supported. Do not use it as a catch-all for an implemented check's configuration, data, or flow failures; return `Unknown` when assessment was prevented and `Safe` only when evidence disproves applicability.
- Normally prepend AutoCheck. Do not duplicate its vulnerability gate inside `exploit`, depend on state from a prior console `check`, or defeat `AutoCheck false` and `ForceExploit`. Cached AutoCheck state must remain an optional optimization.
- In `run` and `exploit`, use a specific `fail_with(Failure::..., reason)` for fatal target, response, or configuration failures. Rescue only exceptions the invoked operation can genuinely raise; do not conceal programming errors with a blanket rescue.
- In library code, raise a domain-appropriate exception class and preserve useful context. When several custom exception classes divide expected failure modes, document what each represents and verify that every raise/rescue maps only the intended operational failures. Never `rescue Exception`; it catches signals and exits.
- Before writing bespoke parser or decoder code, inspect current declared or locked dependencies, their public APIs, and available security notes. For untrusted input, understand resource behavior and enforce format-appropriate structural and resource limits before or around delegation; translate only expected documented exceptions, link the governing format for nontrivial binary formats, and test applicable malformed, truncated, trailing, and resource-exhaustion boundaries.
- When compatibility depends on framework or library configuration, trace how the application actually constructs both producer and consumer, including wrappers, fallbacks, and version-dependent defaults, and reproduce that path end to end. A standalone probe of a class or dependency proves only the configuration it directly instantiated.
- Validate product-specific evidence after every state-changing operation. Track ownership and restoration state before the next fallible step, and before an operation that can partially mutate despite failing. Delete only artifacts created by the module, preserve captured original state, call `super` from cleanup, and verify an immediate rerun.
- Report an identified application or protocol service as soon as evidence supports it. Associate vulnerabilities, credential logins, and loot with that exact service/resource, avoid duplicates, support `AutoCheck false`, and tolerate a disconnected database.
- Let the framework choose compatible payloads when possible. Retest every target and payload-selection path after target, architecture, staging, or default changes; do not infer success from session counts or call the handler without a demonstrated lifecycle reason.
- Keep documentation operationally accurate. A human must provide and refresh Scenarios from real testing; never invent output, sessions, versions, credentials, or target details, and redact secrets from recorded output.
- Treat a rebase as a possible behavioral change because mixins, payload generation, parsers, handlers, and reporting can change without a module diff.

## Validation and handoff

Run validators required by the current checkout, including Ruby syntax, repository RuboCop, `msftidy`, documentation tidy, `git diff --check`, focused RSpec for library changes, and any specialized validator named by repository instructions.

Test the source checkout rather than a copied or packaged module. Isolate the console configuration to avoid stale user modules and settings:

```bash
MSF_CFGROOT_CONFIG="$(mktemp -d /tmp/msf-module-test.XXXXXX)" ./msfconsole -q
```

Use the task-appropriate manual matrix in [review-qa.md](./references/review-qa.md). Exercise relevant CheckCode branches, AutoCheck overrides, targets/actions, payload selection, malformed and unavailable responses, partial-failure cleanup, database-disabled behavior, and an immediate rerun. Retest after a rebase or payload/default change.

In the handoff, distinguish static validators, manually tested environments and paths, cleanup/repeatability evidence, and anything not tested or not proven.
