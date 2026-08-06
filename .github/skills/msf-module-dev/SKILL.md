---
name: msf-module-dev
description: "Develop and review Metasploit Framework modules using current Rapid7 conventions. Use when creating or fixing remote, local, auxiliary, scanner, or post modules; responding to PR feedback; validating metadata, AutoCheck, payloads, reporting, cleanup, documentation, msftidy, RuboCop, or manual QA."
---

# Metasploit Framework module development

Use this skill for module implementation and review. It is a workflow and routing guide; the current checkout remains the source of truth.

## Establish the rules first

1. Find and read every applicable `AGENTS.md` before inspecting examples or editing files.
2. Prefer evidence in this order:
   - the user's request and applicable repository instructions;
   - current framework source, mixins, specs, and official developer documentation;
   - recent Rapid7 review feedback and recently merged code;
   - existing modules, after checking that their APIs and patterns are current.
3. Never copy a pattern from one old module without verifying it. Conventions around service reporting, payload selection, timeouts, parsers, and AutoCheck evolve.
4. Separate observations from conclusions. Reproduce regressions and compare relevant variables before claiming a root cause.

Read [review-qa.md](./references/review-qa.md) when reviewing a PR, rebasing a module, or preparing manual tests.

## Classify the module

| Outcome | Base class and usual path |
| --- | --- |
| Remote code execution/session | `Msf::Exploit::Remote`, `modules/exploits/...` |
| Local privilege escalation | `Msf::Exploit::Local`, `modules/exploits/<platform>/local/...` |
| Scanning, gathering, administration | `Msf::Auxiliary`, `modules/auxiliary/...` |
| Work through an existing session | `Msf::Post`, `modules/post/...` |

Prefer Ruby because external Python and Go runtimes do not support all framework features. Check for an existing module or open pull request before adding one. Keep one module per pull request unless the repository explicitly calls for an action or target on an existing module.

New modules need:

```text
modules/<type>/<path>/<name>.rb
documentation/modules/<type>/<path>/<name>.md
```

The documentation type is singular or plural exactly as the module tree uses it; exploit documentation is under `documentation/modules/exploit/...`.

## Build from current framework contracts

- Inspect every included/prepended mixin. Do not re-register inherited options or reimplement helpers.
- Use the repository's `.ruby-version` and maintain Ruby 3.1+ compatibility. There is no project-wide line-length limit, but keep code readable.
- Add `# frozen_string_literal: true` to new Ruby files.
- Follow the repository RuboCop configuration: two-space indentation, no tabs/trailing whitespace, and single quotes unless interpolation/escaping makes double quotes clearer.
- Do not `require` autoloaded MSF/Rex dependencies. Require an otherwise unloaded Ruby stdlib only when the implementation needs it.
- Use `MSF_LICENSE`, and include `DisclosureDate` for exploits.
- Exploit, auxiliary, and post modules require `Notes` with `SideEffects`; provide accurate Stability and Reliability values too.
- Use an ASCII `%q{}` Description. State the vulnerable range and fixed version when known.
- Normal option names are `SCREAMING_SNAKE_CASE`; advanced option names are `CamelCase`.
- Use mixin accessors where current lint supplies one (for example `srvhost`, not `datastore['SRVHOST']`).
- Authentication secret options normally default to `nil`. Use `Faker` for accounts or credentials the exploit creates.
- Randomize attacker-controlled titles, slugs, filenames, branch/plugin names, and similar identifiers when the protocol permits it. Explain constants required by the vulnerability.
- Avoid a default `PAYLOAD` when framework selection works. A per-target default is acceptable only when required for correct selection and verified across target switches.
- If an exploit has one action, normally omit `Actions` and `DefaultAction`.
- Use hashes for structured return values and keyword arguments for reusable APIs.
- Do not introduce `get_`/`set_` accessor prefixes. Method parameter names have at least two characters except established crypto abbreviations.
- All `print_*` messages start with a capital letter. Use `vprint_*` for diagnostics that are useful only under `VERBOSE`.
- In `run`/`exploit`, use the specific `fail_with(Failure::..., reason)` for fatal target/configuration failures rather than `raise`, `abort`, or a silent bare return. Do not swallow unexpected errors in a broad rescue.
- Multiline comments are acceptable for embedded scripts/payloads. Do not reduce their readability merely to satisfy an imagined line-length rule.
- Use `Msf::Exploit::SQLi` for SQL injection, RubySMB for SMB, `Rex::Stopwatch.elapsed_time` for elapsed timing, and `Rex::MIME::Message` for MIME. When generating runtime-language identifiers, pass the language explicitly to `Rex::RandomIdentifier::Generator`.
- Use TEST-NET-1 (`192.0.2.0/24`) in specs/examples that require non-routable public addresses; genuine module-documentation labs may use local/private addresses.

See [metadata.md](./references/metadata.md) and [module-template.md](./references/module-template.md).

## Check and AutoCheck

Implement `check` when reliable detection is possible and normally use:

```ruby
prepend Msf::Exploit::Remote::AutoCheck
```

Every path through `check`, including exceptions and shared helpers, must return a `CheckCode` with a human-readable reason. Never call `fail_with` or raise from `check`.

Exploit subclasses can use bare `CheckCode`. Auxiliary, post, and evasion modules must use `Exploit::CheckCode` or `Msf::Exploit::CheckCode`; bare `CheckCode` raises `NameError` and is rejected by current lint.

| Code | Meaning |
| --- | --- |
| `Safe` | The module's exploit path is not applicable or was disproved |
| `Detected` | The expected service/product was found, but exploitability is unknown |
| `Appears` | A detected version is inside the researched vulnerable range |
| `Vulnerable` | The vulnerability itself was safely confirmed or triggered |
| `Unknown` | The module could not determine the result |
| `Unsupported` | The check operation is not supported for this configuration |

Important framework behavior:

- AutoCheck proceeds on `Detected` with a warning.
- AutoCheck blocks `Safe`, `Unknown`, and `Unsupported` unless `ForceExploit` is enabled.
- `AutoCheck false` bypasses `check` completely.
- Do not add a second mandatory vulnerability check inside `exploit`; that breaks those overrides.
- A manual `check` command and a later `run` can use different module instances. Do not depend on state from a prior console command. State created by AutoCheck may be reused only as an optimization, with a safe fallback when absent.
- Test `check` against unrelated software and both sides of the vulnerable version range.
- Missing permissions, privileges, session capabilities, or response data normally mean `Unknown`, not `Safe`, because assessment was prevented rather than disproved.
- Distinguish an affirmative negative result from a failed assessment. A timeout/parser/protocol error in the primary confirmation path must not be reinterpreted as `Safe` merely because a weaker fallback behaves differently.

Use `Rex::Version` for comparisons and return a `Rex::Version` from version helpers where appropriate.

## HTTP behavior

Use `Msf::Exploit::Remote::HttpClient`, `normalize_uri`, and `send_request_cgi`. Do not pass datastore-backed `SSL` or an arbitrary timeout into every request.

- A response can be `nil`; branch before parsing it. In `check`, return a reasoned CheckCode. In `exploit`/`run`, use the appropriate `fail_with` result.
- Use `res.body.to_s` for string operations.
- Use `res.get_json_document` for JSON. It returns an empty hash on a parse error, so validate the expected product-specific keys, types, and values rather than treating `is_a?(Hash)` as proof of valid JSON.
- `normalize_uri` joins path components and collapses duplicate slashes; it does not encode them. Validate or encode user- and response-controlled path segments before passing them to it.
- Use `res.get_html_document`/`get_xml_document` and precise selectors for markup. For values embedded in JavaScript, first select the relevant `<script>` node, then regex only that JavaScript text.
- Use `vars_form_data` or `Rex::MIME::Message` for multipart bodies. Do not combine a hand-built raw body with `vars_post`.
- Use the default HTTP timeout unless a protocol property requires otherwise. A fire-and-forget payload trigger may use a short finite timeout; explain why. Include the current `Msf::Exploit::Retry` mixin and use `retry_until_truthy`/`poll_until_truthy` with a configurable CamelCase timeout for asynchronous work.
- A `nil` trigger response is not proof of success unless the protocol specifically makes it expected and independent evidence confirms the effect.
- HTTP connection failure is generally represented by a `nil` response; add exception handling only for exceptions the called helper actually raises.
- Keep the base path in `TARGETURI`, query parameters in `vars_get`, and session cookies in the cookie jar. Do not append an ad hoc query string and then also use `vars_get`.

See [http-client.md](./references/http-client.md).

## Payloads and targets

- Prefer fetch payloads when the target command channel only offers download tools such as curl or wget. Do not add a command-stager target solely to download a binary.
- Expose an `ARCH_CMD` path when the primitive executes arbitrary commands. Prefer it to a command stager when native delivery would only invoke curl/wget, but retain a native/dropper target when it has a demonstrated capability or constraint. Define bad characters instead of manually base64-wrapping payloads.
- Use `target['Type']` for target dispatch, not instance-variable sentinels.
- Do not infer success by counting sessions; not every payload opens one.
- Never submit an opaque binary without source and build instructions.
- After changing targets, default payloads, architecture, or staging, manually retest every target. Change `TARGET` without explicitly setting `PAYLOAD` to verify framework reselection, then test representative explicit payloads.
- Normal framework execution owns the handler; do not call `handler` manually without a demonstrated lifecycle reason.

## Reporting

Report what the module actually discovers. Prefer an idempotent helper that reports the application as soon as it is identified and returns its exact service object.

For a web application, model the service hierarchy:

```text
application(resource: base URI) -> http -> tcp
application(resource: base URI) -> https -> ssl -> tcp
```

Link database records to that same application service:

- pass `service:` to `report_vuln`;
- use its `service_id` and matching `service_name` for credential logins;
- pass it as the final `store_loot` argument;
- set credential `access_level` when known and `last_attempted_at` for successful logins.

Do not rely on same-port fallback: it can associate a vulnerability with the TCP parent instead of the application. Do not call `report_host` just because a service, credential, or vulnerability is also being reported; those APIs report the host.

See [reporting.md](./references/reporting.md).

## Cleanup and repeatability

- Include `Msf::Exploit::FileDropper` where appropriate. Register files with `register_file_for_cleanup`/`register_files_for_cleanup` and directories with `register_dir_for_cleanup`.
- Put fallback cleanup in `cleanup`, not only after the trigger. Always call `super` when overriding it.
- Track ownership: delete only artifacts the module created. Preserve pre-existing accounts, files, plugins, repositories, and configuration.
- Maintain a state ledger for application/database rows, cache objects, posts, accounts, branches, tokens, settings, files, and directories—not only filesystem artifacts.
- If a failed call can still partially mutate state, capture the original state and safely arm restoration before the attempt. Otherwise arm cleanup immediately after mutation is confirmed and before the next fallible operation. Preserve any cookie, ID, original value, or other data needed to reverse it.
- Check every target mutation (`write_file`, append, upload, API state change), verify the resulting state, and restore captured original data rather than assumed defaults.
- Treat cleanup as another state-changing operation: validate product-specific success data or read the state back before printing success or clearing ownership flags.
- Cleanup often runs after a session starts, so avoid depending on a request that the payload indefinitely blocks. Register artifacts before triggering them.
- For irreversible or materially destructive behavior, use an advanced `DefangedMode` guard where current framework precedent requires it. Do not label normal, reversible exploit artifacts as destructive merely because cleanup is needed.
- Test failure after each mutation, successful cleanup, and an immediate rerun with the same configured names. A clean rerun is strong evidence against stale-state bugs.

For local/post modules, verify a file exists before opening it. Use `create_process(executable, args: [], time_out: 15, opts: {})` instead of deprecated split-argument `cmd_exec` calls. An operation that fundamentally requires a session belongs under post/local rather than auxiliary with custom session plumbing. Code must tolerate a disconnected database. Follow [non-http-modules.md](./references/non-http-modules.md).

## Documentation

Follow `documentation/modules/module_doc_template.md` and [documentation-template.md](./references/documentation-template.md).

- Explain exact vulnerable and fixed versions, prerequisites, and reproducible setup.
- Option names and defaults must match the module, including case.
- Scenarios must be produced and refreshed by a human from real testing. Do not invent console output, sessions, timestamps, credentials, or target details.
- Redact credentials, API tokens, hashes, and other secrets from human-recorded output, even when the module generated them for a disposable lab.
- Refresh Scenarios after changing defaults, targets, payloads, output, failure behavior, reporting-visible behavior, or cleanup order.
- Use placeholders for secrets; do not include real-looking API keys, private hashes, or credentials.

## Validate in the source checkout

Run from the repository root:

```bash
ruby -c modules/<type>/<path>/<name>.rb
bundle exec rubocop modules/<type>/<path>/<name>.rb
ruby tools/dev/msftidy.rb modules/<type>/<path>/<name>.rb
ruby tools/dev/msftidy_docs.rb documentation/modules/<doc-type>/<path>/<name>.md
git diff --check
```

For library changes, add and run focused RSpec tests:

```bash
bundle exec rspec spec/path/to/spec.rb
```

Also apply repository-specific validators, including `tools/dev/hash_cracker_validator.rb` for new hash-cracking support. Public library APIs need YARD documentation; library specs should follow Better Specs conventions. Do not add new files to `scripts/`.

Do not use invalid `pack`/`unpack` directives, include sensitive IPs/credentials/keys/hashes, or submit opaque blobs without source/build instructions.

LoginScanner or generated fetch-command changes are library work: add contract tests for connection failures/result statuses or shell/download fallback behavior rather than relying only on a module run.

Do not hardcode `/opt` paths. Do not copy a checkout module to `~/.msf4/modules` for normal source-tree testing; a stale user module can shadow the edited file. Start the repository's console with an isolated config root:

```bash
MSF_CFGROOT_CONFIG="$(mktemp -d /tmp/msf-module-test.XXXXXX)" ./msfconsole -q
```

If testing an external project with `loadpath`, point it at a directory containing the proper `modules/` tree and still isolate the config root.

## Final QA

- Review `git diff` and `git diff --check`; ensure unrelated work was not changed.
- Validate syntax, RuboCop, msftidy, docs tidy, and focused specs.
- Exercise `check`, `AutoCheck true`, `AutoCheck false`, and `ForceExploit` where relevant.
- Exercise every target/action and payload-selection path.
- Test unrelated/patched targets and malformed/nil responses.
- Test cleanup after partial failure and success, then rerun immediately.
- Retest after rebasing onto current master; payload and mixin behavior can change without a module diff.
- Confirm explicit/AutoCheck reporting uses the exact application service and remains correct with AutoCheck disabled. Record current API limitations for manual check reporting rather than inventing unsupported CheckCode keywords.
- Confirm human-written Scenarios match the final code and test output.
- Never commit or push unless the user explicitly requests it.

## References

| Topic | Reference |
| --- | --- |
| Review freshness and manual QA | [review-qa.md](./references/review-qa.md) |
| Annotated patterns | [module-template.md](./references/module-template.md) |
| Metadata, options, targets | [metadata.md](./references/metadata.md) |
| HttpClient requests and parsing | [http-client.md](./references/http-client.md) |
| Services, vulnerabilities, credentials, loot | [reporting.md](./references/reporting.md) |
| Scanner, TCP, local, and post modules | [non-http-modules.md](./references/non-http-modules.md) |
| Documentation | [documentation-template.md](./references/documentation-template.md) |
