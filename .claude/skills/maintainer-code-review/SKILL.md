---
name: maintainer-code-review
description: Review a pull request, branch or diff for the Appium .NET client (appium/dotnet-client) the way the maintainers KazuCocoa and mykola-mokhnach do. Use when asked to review, pre-review or self-review a change, or to check a PR before it is opened. Covers server-contract correctness, Appium 3 deprecations, test strategy, CI stability, PR scope and title conventions.
---

# Maintainer code review (Appium .NET client)

Reviews a change against expectations distilled from the maintainers' review comments on the 107 PRs (of the
latest 300) that they reviewed, #838 to #1162. Each rule cites the PRs it came from, so that findings can be
traced and challenged.

Read `AGENTS.md` first. Its rules (backwards-compatible public API, no new warnings on any target framework,
tests with every fix or feature, PR title prefixes) are binding and are not repeated here.

## How to run a review

1. Identify the target: a PR number, branch or the working-tree diff. Fetch the diff and the changed-file list.
2. Classify each changed file as `src/` (product, `netstandard2.0`), `test/integration/` (server-free or
   functional), `.github/` (CI, templates) or docs.
3. Apply the checks below that match the file class. Skip checks that do not apply; do not pad the review.
4. Report findings ranked by severity (see "Output"). Do not post comments, approve or request changes on GitHub
   unless the user asks you to.
5. Never recommend approval while any CI check is failing, pending or unknown (`AGENTS.md`).

Keep the review proportionate. The maintainers approved mechanical changes without comment: dependency bumps,
compatibility-matrix docs, CI tweaks and template edits (#940, #956, #973, #1085, #1157). Spend effort where the
checks below bite, and say "no findings" when that is true.

## 1. Server contract and Appium 3 readiness (`src/`)

- **Know the endpoint.** For every driver call, state which server API it hits. Legacy JSONWP/MJSONWP routes
  (for example `/appium/device/start_activity`) are deprecated and removed in Appium 3. Prefer `mobile:` extension
  commands. Ask the author for the full route if it is unclear (#902, #874, #939).
- **Check what is actually sent.** The wire payload must match the server's expectation, for example `pushFile`
  needs a base64 string. Parameter renames must not blur the data type (#901).
- **Typed helpers need an escape hatch.** A strongly typed wrapper over an extension command should also accept a
  raw `Dictionary<string, object>` of arguments that takes precedence, so server-side additions do not force a
  client release (#986).
- **Prefer BiDi where it exists.** If a feature is available over BiDi, an extension-command or WebSocket
  implementation is acceptable only if the XML docs say it is a stop-gap and point to BiDi (#969).
- **Breaking changes.** A legacy fallback (try the `mobile:` command, fall back to the old route) can avoid a
  breaking change. If the change is genuinely breaking, the title must say so, and breaking changes should be
  batched into one release where possible (#874, #939, #1118).
- **Align timeouts with the server.** Client-side waits must exceed the server's, for example Appium's default
  `--shutdown-timeout` is 5000 ms and `AppiumLocalService` knows its own server arguments (#1006).
- **Fewest round trips.** Flag helpers that fetch an attribute per element over the wire. Prefer one XPath or
  server-side filter. A test may exercise the slow helper deliberately, but say so (#902).
- **Platform and host details.** Use `RuntimeInformation.IsOSPlatform` (not ad hoc checks) and avoid repeating
  the call; cache it (#1006, #1075). Use `127.0.0.1` explicitly rather than `localhost` (#969). Use invariant
  casing (`ToLowerInvariant`) to avoid host-locale bugs (#921).
- **Paths and state.** Join paths with `Path.Combine`/`Path.Join`, never assume a trailing separator, and set
  "initialised" flags last and not on failure (#906). Extract path validation into one reusable helper instead of
  inlining it (#1075).
- **Mechanical slips.** Identical ternary arms, typos in identifiers and XML docs (#902). Match surrounding
  formatting (#965).

## 1a. Upstream sources of truth

This client does not define server or driver behaviour; it follows it. When a change adds or alters a command,
capability or default, check it against the owning upstream, and say which one you used. Do not rely on memory.

| Area | Upstream | What to check |
|---|---|---|
| Server, base path, `mobile:` extension model, Appium 3 removals | `appium/appium` | Route is not deprecated or removed; `--base-path` and server arguments behave as assumed (#1006, #902). |
| Android | `appium-uiautomator2-driver`; Espresso driver for Espresso tests | Command and argument names against its execute-methods reference (the README table links to it, #939, #986). |
| iOS | `appium-xcuitest-driver` | `mobile:` command names and argument shapes, for example `installApp` (#1068, #1031). |
| macOS | Mac2 driver | Command names and capabilities; the repo has a `Mac` folder and tests but links no driver repository. |
| Windows | `appium-windows-driver`; NovaWindows driver (third party) | Not the unmaintained WinAppDriver. A Wincore driver is only in open PR #1159, so do not assume it exists on `main`. |
| Selenium .NET binding | `SeleniumHQ/selenium` (`Selenium.WebDriver` on NuGet) | See below. |

Selenium .NET is the most influential upstream because `Appium.WebDriver` extends it:

- Check new or changed Selenium types and members that the change touches for deprecation. For example, the
  `Response` constructor change in Selenium 4.30 forced test updates (#910).
- Keep the supported Selenium range open. Raise the minimum only when a Selenium release forces a client change,
  and let Dependabot keep exercising newer versions in CI (#980, README compatibility matrix).
- Changes that follow a Selenium release, such as strong-name signing aligned with Selenium 4.44.0, are breaking for
  consumers and need the `!` or `BREAKING CHANGE:` title (#1118).
- When an upstream behaviour cannot be confirmed from the diff or the repo, report it under "Not verified" and name
  the upstream document to consult. Do not invent driver behaviour.

## 2. Tests (`test/integration/`)

- **Server-free where possible.** Client-side logic (caching, builders, guard clauses, URI parsing, retry logic)
  must be tested without an Appium server, and the new test namespace must be added to the `--filter` in
  `unit-test.yml` so CI runs it (#985, #1128, #1139).
- **Assert behaviour, not noise.** Do not assert full exception messages; assert the exception type or that one is
  thrown (#922). A test that only checks "does not throw" must say so, and the reviewer will ask whether the
  behaviour really happened (#1129).
- **Do not test the test helpers.** Tests that only exercise `Caps.cs` or similar infrastructure are low value
  (#1120).
- **Unique custom command names.** Custom-command tests must not reuse an existing command name such as
  `getStatus` (#965).
- **Stability belongs in CI, not in test code.** Fix simulator or emulator flakiness in the workflow YAML rather
  than sprinkling waits or timeout overrides in tests (#1120). Where a test must be skipped on CI, use
  `Env.IsCiEnvironment()` so skipped tests stay greppable (#993, #994).
- **Do not over-engineer test infrastructure.** Test helpers should fail loudly. Skip defensive validation of
  downloaded files or log rotation that only hides real problems (#987, #993, #1012).
- **Use an established library over a hand-rolled fake.** For example WireMock.Net instead of a custom HTTP mock
  server. Remove unused helpers and keep the PR description in sync with the implementation (#1160).
- **Prove the CI matrix.** New test infrastructure must pass `net48` (Windows) and `net8.0` on Windows, Linux and
  macOS before merge (#1160).

## 3. Scope, docs and process

- **Title and scope.** The title prefix must match what the change does ("Is this a fix?" on a `test:` PR, #918).
  Split unrelated changes into their own PR and keep CI or simulator tuning out of feature PRs (#1120).
- **Docs.** Do not duplicate dependency lists that will drift; link to the csproj or NuGet page (#848). Never
  delete dead links without a replacement (#1098). Docstrings should state the recommended approach, not only
  the implemented one (#969).
- **Dependencies.** Leave the Selenium range open so users can upgrade themselves. Bump the minimum only for a
  breaking change, and let Dependabot keep exercising newer versions in CI (#980). Library dependencies must not
  be bumped by the test-only Dependabot entry (#1144).
- **Product versus test dependencies** stay separate (`src/Appium.Net/Appium.Net.csproj` versus the integration
  project) (#848).

## 4. Reviewer conventions (user preferences)

- Write review comments, XML docs and prose in British English (for example "initialise", "behaviour").
  Do not rename existing public identifiers for spelling; that would break API compatibility.
- Suggested code must follow SOLID and DRY (single-purpose helpers, no duplicated validation) and must not
  introduce patterns an AI static-analysis gate would flag as Critical or Major: unused members, empty catch
  blocks, swallowed exceptions, duplicated blocks, high cognitive complexity.

## Output

Report in this shape, most severe first:

```
Verdict: <ready | changes needed | blocked on CI>

1. [Blocker | Should fix | Nit] path/File.cs:LINE: finding
   Why: <rule above, with PR reference> 
   Suggest: <concrete change, as a small snippet when useful>
```

- **Blocker**: wrong wire contract, deprecated-route dependence without a plan, failing CI, missing tests for a
  fix or feature, security issues.
- **Should fix**: scope, brittle assertions, flakiness handled in the wrong layer, duplicated logic.
- **Nit**: typos, formatting, naming. Group them in one list.

State what you could not verify (for example, functional tests that need a device) rather than implying it passed.

## Evidence base and limits

Derived from review threads on 107 PRs created in the latest 300, #838 to #1162. KazuCocoa left several dozen
substantive comments; most of his other reviews were bare approvals. mykola-mokhnach left substantive comments on
five PRs only (#901, #902, #906, #922, #1006), all from early 2025, so rules in section 1 are weighted towards his
server-protocol expertise and may under-represent his current views. Some of KazuCocoa's comments come from PRs he
authored himself (#987 to #994, #1012, #921), where they are replies to Copilot, not review of others' work.
Treat the rules as strong defaults, not law, and refresh this file as new review feedback accumulates.
