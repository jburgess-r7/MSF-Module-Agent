# Review freshness and QA

Use this reference when reviewing a module, responding to pull-request feedback, or retesting after a rebase. It exists to prevent old modules from becoming accidental policy.

## Evidence order

For each non-obvious design decision, use the strongest available evidence:

1. The user's requested outcome defines scope; applicable `AGENTS.md` governs the files. Surface any direct conflict.
2. Current mixin/API implementation and its specs.
3. Official Metasploit developer documentation.
4. Recent Rapid7 review discussion and recently merged implementations.
5. Older modules, only as corroborating examples.

Search the current source before inventing a helper:

```bash
rg -n "def helper_name|module MixinName" lib modules spec
rg -n "report_service\(|create_credential|store_loot\(" modules lib
git log --since='12 months ago' -- modules/<area> lib/msf/core/<area>
```

When review feedback changes a convention, update the rule and the example that demonstrates it. A correct prose rule paired with stale sample code will reproduce the same defect.

## Pull-request feedback ledger

For an existing pull request, enumerate every current top-level and inline review conversation before editing and again before handoff, including threads GitHub marks outdated. Track each thread's URL or path, request, disposition, rationale, resulting code/documentation/spec change, manual evidence, and required reply. Distinguish already resolved threads from unresolved work.

Refresh the ledger after applying suggestions through the GitHub UI or receiving new feedback. Do not claim that all feedback is addressed until every entry is reconciled. If authentication, pagination, or page-loading failures prevent complete enumeration, state that review coverage is incomplete.

## Fast review searches

Use these as prompts for inspection, not automatic findings:

```bash
rg -n "JSON\.parse|session\.db_record\.id|write_file\(|append_file\(|upload_file\(" <changed-files>
rg -n "report_host\(|report_service\(|report_vuln\(|create_credential|store_loot\(" <changed-files>
rg -n "CheckCode::(Safe|Detected|Appears|Vulnerable|Unknown|Unsupported)" <changed-files>
rg -n -U "send_request_cgi\([\s\S]{0,500}?\},\s*[0-9]+\)" <changed-files>
rg -n "register_advanced_options|Opt[A-Za-z]+\.new" <changed-files>
```

Then inspect context:

- a numeric HTTP timeout may be justified by protocol behavior;
- a file write is correct only when its result and resulting state are checked;
- reporting calls may be duplicates if a mixin already owns them;
- an advanced option is CamelCase even though normal options are SCREAMING_SNAKE_CASE;
- a CheckCode needs a reason and evidence matching its strength.

## Reviewer-derived recurring checks

These are patterns seen repeatedly in recent Rapid7 reviews; verify them against the current checkout rather than treating the linked discussion as an eternal API contract.

- Let the framework select a payload where possible; prefer fetch payloads to a dropper whose only staging mechanism is curl/wget. See [PR 21515](https://github.com/rapid7/metasploit-framework/pull/21515#discussion_r3318290659) and [PR 21336](https://github.com/rapid7/metasploit-framework/pull/21336#discussion_r3127077305).
- Use the framework HTTP timeout unless an operation has a demonstrated timing requirement. See [PR 21515](https://github.com/rapid7/metasploit-framework/pull/21515#discussion_r3318419177) and the justified asynchronous timeout discussion in [PR 21417](https://github.com/rapid7/metasploit-framework/pull/21417#discussion_r3218735854).
- Keep `TARGETURI` as the base path, use `vars_get` for query parameters, and use the cookie jar for multi-step sessions. See [PR 21615](https://github.com/rapid7/metasploit-framework/pull/21615#discussion_r3505515761) and [PR 21615](https://github.com/rapid7/metasploit-framework/pull/21615#discussion_r3505510812).
- Parse HTML/XML structurally, guard a nil response before parsing, and use `vars_form_data` for multipart requests. See [PR 21371](https://github.com/rapid7/metasploit-framework/pull/21371#discussion_r3319070007) and [PR 21371](https://github.com/rapid7/metasploit-framework/pull/21371#discussion_r3319070078).
- Do not treat no response as proof that an exploit succeeded. See [PR 21371](https://github.com/rapid7/metasploit-framework/pull/21371#discussion_r3319070207).
- Model application, HTTP/TLS, and transport services and link the vulnerability to the application service, including when AutoCheck is disabled. See [PR 21515](https://github.com/rapid7/metasploit-framework/pull/21515#discussion_r3343502772) and [PR 21515](https://github.com/rapid7/metasploit-framework/pull/21515#discussion_r3349008637).
- Give every CheckCode a useful reason and use AutoCheck instead of manually calling `check` inside `exploit`. See [PR 21019](https://github.com/rapid7/metasploit-framework/pull/21019#discussion_r3059259296) and [PR 21019](https://github.com/rapid7/metasploit-framework/pull/21019#discussion_r3059292757).
- Validate the response after a state-changing request; an HTTP success code alone may not prove the requested state change. See [PR 21417](https://github.com/rapid7/metasploit-framework/pull/21417#discussion_r3201854718).
- Advanced options are CamelCase and genuinely destructive behavior may require an advanced DefangedMode guard. See [PR 21417](https://github.com/rapid7/metasploit-framework/pull/21417#discussion_r3218632432) and [PR 21417](https://github.com/rapid7/metasploit-framework/pull/21417#discussion_r3218622064).
- Register directories with the directory cleanup API, not the file API. See [PR 21371](https://github.com/rapid7/metasploit-framework/pull/21371#discussion_r3319070142).
- Keep documentation Scenarios aligned with the actual cleanup/control flow and never use real-looking secrets. See [PR 21417](https://github.com/rapid7/metasploit-framework/pull/21417#discussion_r3201854862) and [PR 21585](https://github.com/rapid7/metasploit-framework/pull/21585#discussion_r3474792402).
- Derive SideEffects from behavior: authentication brute force can cause logs and lockouts, while even temporary write probes create disk artifacts. See [PR 21379](https://github.com/rapid7/metasploit-framework/pull/21379#discussion_r3199959252).
- LoginScanner transport failure is `UNABLE_TO_CONNECT`, not rejected credentials; rescue narrowly and test nil/timeout/reset paths. See [PR 21379](https://github.com/rapid7/metasploit-framework/pull/21379#discussion_r3200013590) and [PR 21379](https://github.com/rapid7/metasploit-framework/pull/21379#discussion_r3200147099).
- Do not duplicate reporting already owned by a protocol/login mixin or repeat credentials/banners as notes. See [PR 21379](https://github.com/rapid7/metasploit-framework/pull/21379#discussion_r3199971455) and [PR 21379](https://github.com/rapid7/metasploit-framework/pull/21379#discussion_r3227251097).
- Check every remote write and verify the resulting state. See [PR 20778](https://github.com/rapid7/metasploit-framework/pull/20778#discussion_r2718947105) and [PR 21473](https://github.com/rapid7/metasploit-framework/pull/21473#discussion_r3353453711).
- When privileges or permissions prevent assessment, return `Unknown`, not `Safe`. See [PR 21501](https://github.com/rapid7/metasploit-framework/pull/21501#discussion_r3319473831).
- Modules must tolerate a disconnected database; do not dereference `session.db_record.id` without a guard. See [PR 20778](https://github.com/rapid7/metasploit-framework/pull/20778#discussion_r2719006959).
- Randomize attacker-controlled titles, slugs, filenames, and similar identifiers when possible. See [PR 21686](https://github.com/rapid7/metasploit-framework/pull/21686#discussion_r3720507457).

## Static review pass

Review the whole behavior, not only lines mentioned by a reviewer:

- metadata, affected/fixed versions, references, targets, platform and architecture;
- inherited versus explicitly registered options, including advanced-option casing;
- every return path from `check` and its shared helpers;
- precondition order, so intended `Safe`, `Unknown`, and `Detected` branches are actually reachable;
- nil, malformed, localized, redirected, authenticated, and unexpected responses;
- product-specific JSON shape (`get_json_document` returns `{}` on parse failure) and validated or encoded dynamic URI path segments;
- exact success evidence after every state-changing request;
- target/payload compatibility, bad characters, staging, and handler behavior;
- generated fetch-command behavior under each claimed shell, including false-success downloaders and fallback;
- created-versus-pre-existing artifact ownership;
- cleanup after each mutation and after session creation, including database rows, cache objects, accounts, tokens, branches, and application records;
- exact service linkage for vulnerabilities, credentials, logins, and loot;
- documentation defaults, option names, setup, output, and cleanup order.

Use `git diff --check` and inspect the complete diff. A reviewer suggestion can be accepted through the GitHub UI or implemented locally; correctness is the same, but do not apply both copies or silently overwrite adjacent work.

## Manual test matrix

Start the checkout's `./msfconsole` with an isolated `MSF_CFGROOT_CONFIG`. Record the exact revision, target product version, target architecture, selected target, selected payload, and relevant network topology.

For every case applicable to the module and changed behavior, either test it or record that it was not run and why:

| Case | Expected evidence |
| --- | --- |
| Unrelated service | No false-positive Vulnerable/Appears result |
| Patched or out-of-range version | Reasoned Safe result |
| Vulnerable clean instance | Check and exploit complete as documented |
| `AutoCheck true` | Framework check leads into exploitation correctly |
| `AutoCheck false` | Exploit does not depend on check-created state |
| `ForceExploit true` | No module-local duplicate guard defeats the override |
| Every target/action | Correct platform, architecture, payload, and output |
| Fresh default payload | A new isolated console and module instance with no local or global `PAYLOAD` setting selects the intended compatible payload before payload-specific options are set |
| Change target without setting payload | Framework selects a compatible payload or gives a clear error |
| Partial failure after each mutation | Owned artifacts are removed or clearly reported |
| Immediate rerun with identical names | No stale account/file/plugin/repository collision |
| Verbose mode | Diagnostics are useful and normal output is not noisy |
| Database disconnected | Reporting/session metadata is optional and does not crash execution |

For session exploits, interact with the session (`getuid`, `sysinfo`, `id`, or an equivalent harmless command) instead of treating “session opened” as the only evidence. Test command and staged/session targets separately when both exist. For timing-sensitive changes or a `REPEATABLE_SESSION` claim, run multiple consecutive cycles; ten clean exploit/cleanup/rerun cycles is a useful review target when practical.

Before manual QA, inventory relevant pre-existing lab instances, processes, listeners, sessions/jobs, and configuration. Where configurable, use test-owned names, unique local ports, and isolated configuration. Remove only test-owned state unless broader cleanup is explicitly authorized; verify cleanup-managed state is gone, measure and report intentionally persistent target artifacts, and confirm anything meant to be preserved remains. Distinguish target artifacts, payload/session state, framework state, local evidence, and lab teardown: destroying a disposable lab is not proof that module cleanup worked.

For every mutation, verify that ownership/cleanup state is armed before the following network request can fail. Exercise that exact failure boundary. A cleanup HTTP status is insufficient when the product returns a success field or supports a read-back check.

## Regression discipline

A rebase is a behavioral change even when the module file has no conflict. Framework payload generation, handlers, mixins, parsers, and reporting code can change underneath the module.

After a rebase or payload/default change:

1. Remove or isolate user-module shadows and stale config.
2. Confirm the module path printed by the source checkout if behavior is surprising.
3. Record payload size/type and framework revision as observations, not causes.
4. Reproduce on more than one architecture or topology when the failure may be timing- or transport-sensitive.
5. Compare packets/logs or bisect the relevant framework change before asserting a transport root cause.
6. Rerun the complete target/payload and cleanup matrix.

Do not extract solely for line count or superficial similarity. A cohesive, independently testable module-specific library can be justified with one consumer; normally keep module metadata, options, targets, and top-level `check`/`run`/`exploit` orchestration in the module. Mirror library tests under `spec/lib`, preserve include order, visibility, constants, state, and exception contracts, and prove equivalence before deleting prior coverage. Require demonstrated reuse and stable contracts before promoting it to a broadly shared API.

Bind every test claim to the source state it exercised. For a dirty worktree, a commit identifier alone is insufficient; retain the relevant diff or content hashes with primary evidence, with secrets redacted or access restricted. Distinguish validators and live tests run against the final source state, pre-change live tests, human-provided Scenarios, and historical or summary-only evidence. After later changes, retest affected behavior or qualify the older result. This traceability belongs in working evidence; do not add revision hashes or local diff output to a pull-request description unless they help reviewers.

In the pull-request description or review comment, distinguish:

- validators run and their results;
- environments, versions, targets, and payloads manually exercised;
- retained live evidence and whether it matches the final source state;
- cleanup, repeatability, and measured persistent artifacts;
- historical or summary-only evidence when cited;
- anything not tested or not proven.

Treat the live pull-request title, body, and reviewer replies as diff-aware artifacts. After the final edit, accepted suggestion, rebase, or target/payload/default change, reopen them and reconcile option and target names, payloads, commands, file/spec counts, test results, evidence provenance, and limitations with the final source state. If changing the pull request is not authorized, report the exact paste-ready corrections instead.

## Maintaining this QA package

Before a major module review, refresh against:

1. `git log -p -- AGENTS.md` for policy changes;
2. current module RuboCop cops and their specs;
3. current mixin/reporting/payload implementations and specs;
4. recent Rapid7 PR discussion in the same module/protocol area;
5. the official module documentation template.
6. GitHub's current [custom-agent configuration](https://docs.github.com/en/copilot/reference/custom-agents-configuration) and [agent-skills format](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/add-skills).

Promote a review comment to a general rule only when it follows current framework behavior or recurs across contexts. Label context-specific guidance instead of turning it into an absolute rule. Examples include target-specific `WfsDelay`, `ExitFunc`, payload offsets, and whether a particular exploit needs DefangedMode.

Recent custom cops are themselves useful convention-change signals. At the time of this refresh they enforce reasoned CheckCodes, qualified CheckCode constants in non-exploit modules, `Rex::Version`, `srvhost`, current `cmd_exec` usage, valid pack directives, DisclosureDate, Notes, and non-redundant target Arch/Platform metadata. Run the repository's RuboCop rather than maintaining a frozen duplicate of every cop here.

After editing this package:

- validate `SKILL.md` with the skill validator;
- when GitHub CLI 2.90.0+ is available, optionally run `gh skill publish --dry-run` for the current Agent Skills schema;
- check all relative links and Markdown fences;
- syntax-check Ruby code fences and run repository RuboCop against full module examples;
- forward-test with a clean agent on both an HTTP and a non-HTTP review task;
- revise rules that generate false positives or miss evidence-backed issues.
