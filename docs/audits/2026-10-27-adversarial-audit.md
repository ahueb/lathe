# Adversarial audit of lathe 3.1.0

- **Date:** 2026-10-27
- **Commit audited:** `2cfd25d` ("Release engineering 3.1.0"), all 420 tracked files
- **Method:** read every file; reproduce claimed and suspected behaviour in a scratch shell; check every platform claim against the current Claude Code documentation; check cited sources in full text where the host was reachable; then try to refute each finding before keeping it.

## Summary

No finding is rated High. Nine are Medium, fourteen Low, and several are informational. The most important points:

1. **The auditor's "read-only" git commands run programs chosen by the repository.** `git status`, `git diff`, `git log -p`, `git show` and `git blame` are all on the guard's allowlist, and the auditor is told to use the first four. In a checkout whose `.git/config` defines a textconv driver, an external diff or a clean filter, each of them runs that program (reproduced). Claude Code runs read-only git forms without a prompt in every permission mode. Lathe already hardens its readiness probe against this vector (risk register R20) but not the guard.
2. **`browser-tester` is documented as "read-only by tool list", but it holds an RCE-equivalent tool.** Its `tools` entry grants the whole Playwright MCP server, including `browser_run_code_unsafe`, which Playwright's own README calls "RCE-equivalent".
3. **On Windows without Git Bash, the guard hook blocks every shell call for every agent.** That follows from the current hook documentation and the hook's `bash "…"` command form. README's Windows advice points users toward this state. Not reproduced: no Windows host was available.
4. **The guard's documented property is false as written.** README says it "denies any command whose first token is not on the allowlist". A function definition (`cat ()(touch X); cat`), zsh process substitution (`cat =(touch X)`) and `uniq - FILE` each pass the guard's checks and write files (all reproduced in a shell; the guard's verdict was traced in its code). Lathe scopes the guard to accidental mutation, and the guard never grants permission itself, so the practical risk is lower than the wording suggests.
5. **One piece of assurance evidence was relabelled rather than re-recorded.** The fixture described as a payload "recorded live" and the risk-register sentence saying `lathe:auditor` "was observed" were both changed by find-and-replace during the rename. Git history shows no new recording.
6. **The evaluation evidence is weaker than it is presented.** Three runs per case; cases are clustered; fixtures, graders and skill text were tuned against the same cases; the judge is the same model as the model under test; there are no reference solutions and no without-plugin baseline. The readiness review's negative trigger cases run in an empty directory. The candidate comment guidance gained worked examples that mirror its headline test case.
7. **`plan-execution` is marketed as "the fast path", but lathe's own benchmark points the other way.** All 14 runs that fanned out cost more, and on average took 2.6× longer, than all 4 that did not.

The project also has strengths, listed under "What held up". Among them: every release tag verifies against `.allowed_signers`; every PR's squash merge kept the tree CI checked; the guard fails closed and never grants permission; the probe's git hardening is thorough; and the CI installer verifies a pinned key fingerprint, a signature and a SHA-256.

## Scope, method and limits

**Executed:** my own reproductions in a scratch directory: shell and git behaviour (bash 5.2.21, zsh 5.9, git 2.43.0), Python `shlex` tokenisation, exact binomial intervals and Fisher's exact test. All of them run outside this repository.

**Traced, not executed:** the guard (`plugins/lathe/hooks/readonly-guard.sh`), the installer, `scripts/*.py`, `repo_probe.py`, `ci.sh`, the release script and the evals. The session's permission classifier refused to run code from the repository and refused to run a downloaded PowerShell build (its SHA-256 matched the published hash). Where this report says "the guard allows X", that verdict comes from reading guard lines 64–137 against X, not from running the guard.

**Not run:** `ci.sh`, the unit tests under `scripts/tests/` and `plugins/lathe/skills/production-readiness-review/tests/`, Claude Code itself, and the eval suites. Their results are as recorded in `plugins/lathe/evals/RESULTS.md` and were not re-measured.

**Blocked hosts:** most paper and standards hosts were unreachable from this environment's proxy. These include arxiv.org, aclanthology.org, semgrep.dev, sre.google, docs.aws.amazon.com, gov.uk, csrc.nist.gov, nist.gov, iso.org, slsa.dev, dora.dev, owasp.org, w3.org, cisa.gov, digital-strategy.ec.europa.eu, pmc.ncbi.nlm.nih.gov, sma.nasa.gov, cs.ubc.ca, principlesofchaos.org, agentskills.io, peps.python.org, eslint.org and downloads.claude.ai. Claims from those sources are marked **not verified**. No claim in this report rests on an abstract.

**Severity scale:** severity weighs impact by likelihood under lathe's documented use. Medium means a documented guarantee or claim fails in a way a user could rely on, or a realistic path causes harm. Low means a narrow, latent or documentation-only defect. Info means an observation with no defect.

## Findings

| ID | Severity | Area | Status |
|---|---|---|---|
| M1 | Medium | Guard: allowlisted git verbs run repository-configured programs | Reproduced |
| M2 | Medium | `browser-tester` holds an RCE-equivalent Playwright tool | Documented behaviour + config |
| M3 | Medium | Windows without Git Bash: guard hook blocks every shell call | Documented behaviour, not reproduced |
| M4 | Medium | Guard's stated property is false; fail-open paths | Reproduced (shell) + traced (guard) |
| M5 | Medium | "Recorded live" fixture and "observed" claim relabelled, not re-recorded | Git history |
| M6 | Medium | Retry-on-denial instruction conflicts with the auditor's hard stop and can defeat form-specific deny rules | Text |
| M7 | Medium | Eval evidence weaker than presented | Records + statistics |
| M8 | Medium | `plan-execution` "fast path" claim contradicted by its own benchmark | Records |
| M9 | Medium | Installer: marketplace migration not atomic; rollback scope overstated | Traced |
| L1 | Low | `git status` (allowlisted) writes `.git/index` | Reproduced |
| L2 | Low | "No shell" for `docs-check`/`literature-review` is a denylist; MCP tools remain | Documented behaviour |
| L3 | Low | README's 1M-context advice is wrong | Documented behaviour |
| L4 | Low | Stale or false README statements | Text + history |
| L5 | Low | A second bare `--restore` undoes the first; README is silent | Traced |
| L6 | Low | Release tagging signs `origin/main` after a subject and version check only | Traced |
| L7 | Low | Plan and decision labels remain in comments after a commit that claimed to remove them | Text |
| L8 | Low | CI and pre-push gaps | Traced |
| L9 | Low | Settings-diff redaction misses header and argument secrets | Traced |
| L10 | Low | Readiness probe hangs on a FIFO; excluded directory names apply at any depth | Reproduced (pattern) + traced |
| L11 | Low | Comment checker: JS regex false pass; directive protection is opt-in | Traced |
| L12 | Low | `literature-review` cannot verify quotes or tell full text from abstracts | Text + tool behaviour |
| L13 | Low | Source-basis accuracy: one stale, one mislabelled, some abstract-based | Full text where reachable |
| L14 | Low | Installer default plugin set widens what the policy has to override and adds network MCP servers | Traced |

### M1. The auditor's allowlisted git verbs run programs the repository configures

**Where:** `plugins/lathe/hooks/readonly-allowlist.txt` line 18 (`git|status log diff show blame ls-files rev-parse|^-[A-Za-z]*o|^--ou|^--ext-diff|^--textconv|`); `plugins/lathe/agents/auditor.md` line 12 ("Use read-only Bash (e.g. `git log`, `git diff`, `git show`, …)").

**Evidence (reproduced, git 2.43.0).** In a scratch repository whose `.gitattributes` selects a driver and whose `.git/config` defines it:

| Command (passes the guard) | Config | Result |
|---|---|---|
| `git show` | `diff.evil.textconv` | driver ran (file created) |
| `git diff` | `diff.evil.textconv` | driver ran |
| `git log -p -1` | `diff.evil.textconv` | driver ran |
| `git blame f.txt` | `diff.evil.textconv` | driver ran |
| `git diff` | `diff.external` | external program ran |
| `git status --short` | `filter.evil.clean` | clean filter ran |
| `git blame --no-textconv f.txt`; `git diff --no-textconv --no-ext-diff` | as above | nothing ran |

The forbidden-argument regex denies the opt-in flags `--textconv` and `--ext-diff`. Textconv is already on by default for `diff`, `log`, `show` and `blame`, and `diff.external` applies to `git diff` with no flag, so denying those flags gives no protection. The safe forms (`--no-textconv`, `--no-ext-diff`) are allowed but not required.

**Why it matters.** The Claude Code permissions page says read-only commands, including "read-only forms of `git`", run "without a permission prompt in every mode". The guard adds no prompt either: on its allow path it exits 0 with no output. In a local checkout or archive that carries its own `.git/config`, a normal `/lathe:deep-audit` run therefore executes repository-chosen programs as the user. Lathe's risk register R20 documents this same vector for `repo_probe.py`, reproduced it, and blanks every filter driver before the probe's `git status`. The guard, which README describes as "enforced", gets none of that hardening, and the auditor is told to use exactly these verbs.

**Counter-evidence considered.** SECURITY.md scopes the guard to accidental mutation and assumes the repository "is not actively trying to defeat the guard". A fresh `git clone` does not carry `.git/config`, so exposure needs an untrusted local checkout or archive, the same exposure R20 treats as in scope. Severity stays Medium rather than High because of that precondition.

**Fix direction.** For `diff`, `log`, `show` and `blame`, require `--no-textconv` and `--no-ext-diff`, or deny those verbs. Drop `status` from the allowlist, or have the hook rewrite the command (PreToolUse `updatedInput`) to add `-c` overrides the way `repo_probe.py` does. Add these vectors to `ci.sh`'s guard table. Update auditor.md line 12 to name the safe forms.

### M2. `browser-tester` holds an RCE-equivalent tool while documented as read-only

**Where:** `plugins/lathe/agents/browser-tester.md` line 4 (`tools: Read, Grep, Glob, mcp__plugin_playwright_playwright, mcp__playwright`); README line 212 ("read-only by tool list, no Bash, plus the Playwright MCP tools it needs").

**Evidence.**
- The Claude Code permissions page says "`mcp__puppeteer` matches any tool provided by the `puppeteer` server". Its exception covers only server names containing `__` or ending in `_`, which `plugin_playwright_playwright` does not.
- The Playwright MCP README lists `browser_run_code_unsafe` in the default "Core automation" group: "executes arbitrary JavaScript in the Playwright server process and is RCE-equivalent". It also states: "Playwright MCP is **not** a security boundary."
- The installer enables the official `playwright` plugin by default (`install.sh` line 1036).

**Why it matters.** The agent reads web pages, which are untrusted input. A page that injects instructions can ask for a Node-process code run. The label "read-only by tool list" invites users to approve such calls, or to grant broad allow rules, without knowing what the tool can do.

**Counter-evidence considered.** With lathe's recommended settings (no allow rules), an MCP tool call prompts in the default permission mode. Playwright MCP restricts file access to workspace roots by default, so `browser_file_upload` is narrower than I first assumed. The code tool, though, runs in the Node process with the user's privileges.

**Fix direction.** List the specific tools the procedure uses (`browser_navigate`, `browser_snapshot`, `browser_click`, `browser_type`, `browser_take_screenshot`, `browser_console_messages`, `browser_network_requests`, …) instead of the whole server. Change README line 212 to say what the remaining tools can do.

### M3. Windows without Git Bash: the guard hook blocks every shell call

**Where:** `plugins/lathe/hooks/hooks.json` (PreToolUse, matcher `Bash|PowerShell`, command `bash "${CLAUDE_PLUGIN_ROOT}/hooks/readonly-guard.sh"`, no `shell` or `onFailure` field); README line 188.

**Evidence (documented behaviour).** The hooks page says shell-form hooks run in PowerShell when Git Bash isn't installed. It also says: "Without Git Bash, any exit code other than 0 or 2 from that hook counts as a failure", and a failed PreToolUse hook "with `onFailure` unset can instead block the tool call". The Claude Code changelog for v2.1.299 records the change: "a guard that fails now blocks the call". Under PowerShell, `bash "…"` is most likely either not found, or (with WSL's `bash.exe`) handed a Windows path it cannot open. Either way it exits non-zero, so every PowerShell call by every agent, including the main session, would be blocked.

**Why it matters.** README line 188 tells Windows users they "need Git Bash on `PATH`, or should run `install.sh` so the policy is a rules file and the hook is not needed". That is true only of the SessionStart hook; the PreToolUse hook still runs on every call. README line 34 also offers a marketplace-only install with no Git Bash prerequisite stated.

**Not reproduced:** no Windows host was available.

**Fix direction.** Ship a PowerShell implementation of the guard for Windows without Git Bash (a hook entry with `"shell": "powershell"`), or set `onFailure` explicitly and document which way the guard then fails on Windows. At minimum, correct README line 188 and add a Windows-without-Git-Bash scenario to the Windows CI job.

### M4. The guard's stated property is false, and the platform can skip it

**Where:** README line 217 ("the guard denies any command whose first token is not on the allowlist, and denies redirection, command substitution, backticks, and background `&` outright"); SECURITY.md "The readonly guard's limits".

**Evidence.** Each string below contains none of the denied characters (`> < $ \` and backtick, `&` outside `&&`, unquoted `{}*?[]~`). Python's `shlex.split` gives `cat` or `uniq` as the first token of each simple command, so the guard allows it (traced, lines 64–137). The shell then writes:

| Command | Shell | Effect (reproduced) |
|---|---|---|
| `cat ()(touch PWNED); cat` | bash 5.2.21, zsh 5.9 | defines a function named `cat` whose body runs `touch`; file created |
| `cat =(touch PWNED)` | zsh 5.9 (the default macOS shell; the Bash tool uses the user's shell) | process substitution runs `touch`; file created |
| `uniq - keep.txt` | bash | `uniq` writes its output to its second operand; with empty stdin `keep.txt` is truncated to 0 bytes. The operand limit (1) does not count `-` |
| `ls (Remove-Item x)` via the PowerShell tool | PowerShell | in argument mode a parenthesised expression is evaluated. **Not reproduced:** running the downloaded PowerShell was refused |

Fail-open paths, from the current hooks documentation:
- A hook that cannot start, or that times out, is a non-blocking error and "the call continues through the normal permission flow".
- `onFailure: "block"` exists only from v2.1.295.
- A project's `.claude/settings.json` can set `"disableAllHooks": true`, which turns off plugin hooks for that repository once it is trusted.

**Counter-evidence considered.** SECURITY.md disclaims an adversarial auditor or repository and calls this "an accidental-mutation guard, not a sandbox". None of these strings is something a model writes by accident unless the audited content steers it there. The guard never emits `allow`, so Claude Code's own analysis still applies: it prompts for commands it cannot parse or does not recognise as read-only, so these strings most likely prompt in the default mode. The defect is mainly that the README states a property the code does not have. Users of `bypassPermissions`, or of broad allow rules such as `Bash(cat *)`, are the ones exposed.

**Fix direction.**
- Reword README line 217 to the property the guard actually enforces.
- Deny unquoted `(` and `)` and a leading `=(`.
- Count `-` as an operand for `uniq`, or drop `uniq`.
- Do not apply POSIX `shlex` rules to the PowerShell tool; deny PowerShell for the auditor unless it is parsed with PowerShell's grammar.
- Consider `onFailure: "block"`, the hooks page's recommendation for policy hooks. Because the hook runs for every agent, a guard failure would then block every agent's shell, not only the auditor's (compare M3). State the `disableAllHooks` limit in SECURITY.md.

### M5. Assurance evidence relabelled during the rename

**Where:** `ci/fixtures/pretooluse-payload.json`; `docs/risk-register.md` R12.

**Evidence.** The fixture has two commits. In `8905109` (3.0.0, the rename) its only change is `"agent_type": "engineering:auditor"` → `"agent_type": "lathe:auditor"`; `agent_id` and `prompt_id` are unchanged. R12's sentence "The `agent_type` value `engineering:auditor` was observed in a live PreToolUse payload from a `/engineering:deep-audit` run" became "…`lathe:auditor` was observed… from a `/lathe:deep-audit` run" in the same commit. CHANGELOG, RESULTS.md and the docs record no post-rename recording.

**Why it matters.** The value is probably right: the hooks page says plugin subagents report a plugin-scoped `agent_type`. But the record now claims an observation that was never made. In a project whose purpose is evidence-first engineering, provenance is the evidence. CI's guard table feeds this fixture to the guard, so the "live payload" test is now a hand-written one.

**Fix direction.** Record a fresh payload from a `/lathe:deep-audit` run, or relabel the fixture as synthesised from the documented format, and correct R12.

### M6. Retry-on-denial instruction

**Where:** `verification-loop/SKILL.md` step 8, `implementation-loop/SKILL.md` step 7, `plan-execution/SKILL.md` step 3, `agents/hard-repair.md`, `agents/mechanical-worker.md`. Compare `agents/auditor.md` line 9: "treat any denial as a hard stop on that approach, not something to route around".

**Evidence.** The skills say: "if a permission rule (not the user) denies a check's command, retry it once in the simplest form: one command with no redirect, pipe, or chain, invoking Python as `python3`". Risk register R17 shows the rule was added to pass eval runs whose allow rules were narrow.

**Why it matters.** Permission rules are the user's or administrator's configuration. A deny rule that targets one form, such as `Bash(python *)` or a pipe to `tee`, expresses intent that a rewritten command can defeat. The instruction says "one retry, then stop", and the example it gives is a different interpreter name. It also reads ambiguously in auto mode, where a classifier denial is neither "a permission rule" nor "the user". The policy contradicts itself: the auditor stops at a denial while the implementers route around one.

**Fix direction.** Retry only when the denied form was the model's own embellishment (a redirect, pipe or chain the check does not need), never with a different program name. Report every denial.

### M7. Evaluation evidence is weaker than presented

**Sample size and clustering.** Exact 95% Clopper–Pearson intervals:

| Claim | As reported | 95% interval | If clustered by case |
|---|---|---|---|
| One readiness scenario | 3/3 | 0.29–1.00 | – |
| Readiness scenarios pooled | 24/24 | 0.86–1.00 | 8/8: 0.63–1.00 |
| Trigger, positives | 30/30 | 0.88–1.00 | 10/10: 0.69–1.00 |
| Trigger, negatives | 0/30 | 0.00–0.12 | 0/10: 0.00–0.31 |
| Judge calibration | 0 errors / 25 | 0.86–1.00 accuracy | – |

The three runs of a case share a prompt and fixture, so they are not independent. Anthropic's "A statistical approach to model evaluations" recommends "clustering standard errors on the unit of randomization", and reports clustered errors "over three times as large as naive standard errors". Lathe's own `change-eval` skill (step 5) says not to report pass@k or pass^k "without stating the estimator or independence assumptions". RESULTS.md reports k/3 counts without either.

**Adaptive tuning.** RESULTS.md (2026-10-03 row) records the scenario-4 fixture being "revised several times … each time adding evidence for gaps the skill had raised". A rubric that failed every run was replaced by a regex, and a grader was added after the failures it targets. Scores measured on the same cases that drove those revisions overstate generalisation.

**Construct overlap in the comment-guidance comparison.** `docs/change-eval/comment-guidance-2026-09.md` scores B01 as "improved" (baseline 0/3, candidate 3/3). The baseline's two failures were keeping an unsupported thread-safety claim and changing a "Python < 3.11" workaround comment. The candidate guidance added a rule on exactly that claim type (`comment-guidance.md` line 33) and a worked example: "A comment reads 'Python < 3.11 lacks X; remove when the minimum supported version is 3.11…'. Preserve the version condition". B01 was added on 2026-09-12 (`48406c4`); the example first appears in `cae5368` (2.10.0), and the baseline guidance at `e158b1e` has no "3.11". B01 therefore measures whether the model follows an example that mirrors the test.

**Judge.** The configured judge is the same model as the model under test. On final messages it failed 3 of 9 passing reports (RESULTS.md). Every FAIL is rechecked, but passes are rechecked only for selected graders, so false passes are under-sampled. Anthropic's "Demystifying evals for AI agents" recommends grading by "an isolated LLM-as-judge" per dimension, closely calibrated "with human experts".

**Missing controls.**
- No case has a reference solution. The same post: "create a reference solution: a known working output that passes all graders. This proves that the task is solvable and verifies graders are correctly configured."
- No valid behaviour run has a without-plugin arm. `claude plugin eval` runs one by default; the Claude Code plugin-evals page says `--ablation none` makes "every case run one arm". Every recorded measurement but one passes `--ablation none` (RESULTS.md lines 3, 7, 8, 23–36 and 94). The exception, a 2026-10-02 `--ablation with-without` run (line 9), had its behaviour results voided by the record itself. The readiness and comment-guidance scores therefore cannot be attributed to the plugin. The same page warns that "a suite that passes says nothing about whether the plugin is safe".
- The readiness prompts and fixtures state the gaps in prose.
- `allowed_tools` excludes `Agent`, so the subagent fan-out the skill specifies is never exercised.

**Asymmetric negatives.** The 10 `prr-no-trigger-*` cases have no `case.yaml` or fixture, so even with `--scaffold` they run in an empty directory, where not invoking a repository review is easy. The `cleanup-no-trigger-*` cases do get a fixture. "0/30 negative" for the readiness review therefore says little about discrimination.

**Traces.** Raw results are git-ignored (`/evals/results/`) and not published, so no reader can re-grade the transcripts. The post's rule: "we do not take eval scores at face value until someone digs into the details of the eval and reads some transcripts."

**Fix direction.**
- Report intervals clustered by case.
- Freeze a held-out case set before tuning.
- Add reference solutions, a without-plugin arm and scaffolded negatives.
- Use a judge from a different model family, or human labels, for pass/fail.
- Publish redacted traces.

### M8. `plan-execution` "fast path" is contradicted by its own benchmark

**Where:** README line 9 ("Ship a feature from a plan fast") and line 221 ("the fast path for a written plan"); `docs/change-eval/plan-execution-2026-09-summary.md`.

**Evidence.** In the benchmark's 18 runs, the 14 that fanned out passed 14/14 at a mean of $0.845 and 181 s. The 4 that did not fan out (marked invalid by the protocol) passed 3/4 at a mean of $0.168 and 70 s. Every no-fan-out run cost less than every fan-out run ($0.10–$0.20 against $0.43–$1.27). The pass-rate difference is not significant (Fisher's exact, two-sided p = 0.22). Both variants produced the same diff size on the fixtures where both passed. The benchmark's verdict is "No decision", and no measurement in the repository supports "fast".

**Grounding.** Anthropic's "How we built our multi-agent research system" reports that multi-agent systems use "about 15× more tokens than chats" and that "most coding tasks involve fewer truly parallelizable tasks than research".

**Counter-evidence considered.** The no-fan-out runs were not randomised, n = 4, and the fixtures (2–6 packages) may be too small for parallelism to pay off.

**Fix direction.** Measure plan-execution against a single-agent baseline on plans large enough to parallelise, and drop "fast" until then.

### M9. Installer: non-atomic marketplace migration and overstated rollback

**Where:** `install.sh` lines 641–642 (both bindings computed) and 713–750 (step 6); README lines 93–97.

**Evidence (traced).** Step 6 removes the legacy `engineering` marketplace when it is bound to this source. Per the script's own comment, that "also uninstalls the plugins installed from it". Only afterwards does it check whether `lathe` is bound to a foreign source, and if so it exits 1. A user with both states ends with no lathe install and the old plugin uninstalled. Separately, `claude plugin marketplace add` failing after the removal leaves neither marketplace. No `ci.sh` scenario combines a legacy match with a foreign `lathe` binding (S23 covers only the foreign binding).

Rollback restores only files this run wrote. CLI-side changes (marketplace removal or addition, plugin install or uninstall) are not undone, and the settings backup is taken after the CLI may already have rewritten `settings.json`. README's "every file the run wrote is restored to what it was before the run" is accurate for files but reads as a whole-run undo.

**Fix direction.** Check every refusal condition before the first CLI mutation. Re-add the legacy registration if the new add fails. Say in README which changes rollback does not cover.

### L1. `git status` (allowlisted) writes `.git/index`

`git status` refreshes the index stat cache. In a scratch repository the index mtime changed from `…541` to `…542` after one `git status`. This is the kind of accidental mutation the guard is meant to stop. `--no-optional-locks` would avoid it, but the guard rejects any command whose second token is not a verb (README's own `git -C d status` example), so the safe form cannot be expressed. Fix: teach the guard to accept a leading `GIT_OPTIONAL_LOCKS=0` assignment or a `--no-optional-locks` option before the verb, or drop `status` (see M1).

### L2. "No shell" for `docs-check` and `literature-review` is a denylist

Both skills use `disallowed-tools: Bash, PowerShell, Edit, Write, NotebookEdit, Agent, Skill, Artifact` with `agent: general-purpose`. The sub-agents page says a denylist leaves the agent with "Bash, MCP tools, and the rest of its pool" minus the named tools. Any MCP server the user has configured that can run commands remains available. Risk register R12 calls this "a tool-list boundary", and README line 213 says it "removes Bash, PowerShell, and every edit tool". Both are true only of the named tools. Fix: deny `mcp__*` too, or fork to a custom agent with an explicit `tools` allowlist.

### L3. README's 1M-context advice is wrong

README line 187: "Drop the `[1m]` suffix or set `CLAUDE_CODE_DISABLE_1M_CONTEXT=1` to stay at 200K." The model-config page says the `[1m]` suffix has "no effect" when the alias already resolves to a model with a native 1M window, which is the case for current models, so dropping the suffix leaves 1M on. Only the environment variable gives 200K. The Claude Code changelog also records that the recommended `[1m]` value combined with `CLAUDE_CODE_DISABLE_1M_CONTEXT=1` failed requests until v2.1.307. Fix: delete the first half of the sentence and note the minimum version.

### L4. Stale or false README statements

- **Line 30:** `--source X` registers X "as the `engineering` marketplace". Since 3.1.0 it is `lathe`.
- **Line 140:** "there is no direct push to `main`, including for the maintainer". `a01092e` (2026-09-29) is a maintainer-committed, maintainer-signed commit on `main`'s first-parent history. It landed after the PR flow existed (`e158b1e`, `cae5368` were merged by GitHub earlier). Line 148 hedges ("once the required check is applied"). The branch reports `protected: true`, but the bypass settings could not be read.
- **Line 148:** "Commits on `main` are GitHub-signed (from the rebase merge)". Releases are squash-merged (`release.sh` line 398 and SECURITY.md say rebase is refused), and `b5aed61`, `48406c4` and `a01092e` are not GitHub-signed.
- **Lines 188 and 34:** Windows and marketplace-only guidance (see M3).
- **Line 217 vs SECURITY.md:** README says "quotes are not parsed"; SECURITY.md says the guard tokenises "with shell quoting rules". Both describe part of the truth: separators split before quotes are honoured, and tokens are then unquoted.
- **"Credentials and data access":** does not mention that the default installer enables plugins with network MCP servers (see L14).
- **Release naming:** commit subjects, tag messages and `ENGINEERING_RELEASE` keep the pre-rename name. This is harmless but undocumented.

### L5. A second bare `--restore` undoes the first

Each restore takes its own pre-restore backup, which becomes the newest stamp, so a second bare `--restore` restores that backup and undoes the first. `ci.sh` lines 930–931 and 1071–1072 work around this by naming stamps; README lines 81–83 do not warn. Pre-restore stamps also count toward the keep-5 pruning, so two restores can prune install backups. Fix: make bare `--restore` skip pre-restore stamps, or document the behaviour.

### L6. Release tagging trusts `origin/main` by subject and version

`release.sh` `cmd_tag` (lines 410–444) fetches `origin/main`, checks the head commit's subject (`Release engineering X.Y.Z`) and the manifest version, then signs a tag over it. It does not verify the commit's signature or compare its tree with the reviewed release branch. Anyone able to put a matching commit on `main` (an admin bypass, per L4) gets it signed. Fix: verify the commit is GitHub-signed, and compare its tree with the PR head that passed CI.

### L7. Plan and decision labels remain

`e158b1e` ("Remove references to uncommitted plan documents") says it removed "plan task/decision numbers that are gitignored and never committed". Several remain:
- `install.sh` lines 506, 540 and 705 ("decision 3", "decision 8", "decision 11").
- `ci.sh` lines 206 ("Task 8 of the assurance plan"), 390, 708, 1081, 1108, 1285 and 1482 ("decision 3/6/7").

`docs/plans/` is git-ignored, so readers cannot resolve these.

### L8. CI and pre-push gaps

- `CI_DOCS_STRICT` defaults to 0, so the README/flag cross-check is a permanent skip (`ci.sh` line 276), and it does not detect stale descriptions such as L4.
- The Windows job's `bash -n` list (`.github/workflows/ci.yml` line 72) omits `plugins/lathe/hooks/readonly-guard.sh`, the hook that runs on every tool call.
- `.githooks/pre-push` runs `ci.sh` on the working tree, which is clean but may not be the ref being pushed (for example `git push origin other:main`).
- `ci.sh`'s guard table has no case for parentheses or function definitions, `uniq -`, zsh `=(…)`, or the git-driver vectors in M1.
- Actions are pinned by SHA (good), but there is no Dependabot or equivalent to refresh the pins.
- GitHub's current secure-use page calls workflow lockfiles "the strongest protection available", superseding SHA pinning "for repositories that adopt it".

### L9. Settings-diff redaction misses some secrets

`scripts/merge_settings.py` redacts values under `env`, `apiKeyHelper`, and keys matching `key|token|secret|password|credential` (line 34). A value under `headers.Authorization` (for example in an `http` hook) is printed in `--dry-run` diffs, as is a token embedded in a hook `command` string, a URL, or an `args` list, because none of those key names match. Fix: add `authorization` and `headers`, and redact URL userinfo and query strings.

### L10. Readiness probe: FIFO hang and broad exclusions

`repo_probe.py` skips symlinks but not FIFOs. `should_scan_text` (lines 368–377) accepts a FIFO named, say, `README.md`; its `stat().st_size` is 0, so the size caps pass; and `read_bytes()` (line 490) blocks. Demonstrated with the same `pathlib` calls: the read was still blocked when `timeout 3` killed it. Separately, `EXCLUDED_DIRS` (line 35) drops `env`, `build`, `out`, `tmp`, `secrets` and similar at any depth, so evidence under, say, `deploy/env/` or `src/build/` is never seen. Fix: check `stat.S_ISREG` before reading, and apply generic names only at the top level.

### L11. Comment checker blind spots (grading tool)

`scripts/comment_guidance_checks.py` `line_scan_region_check` (lines 930–1000) finds a trailing `//` comment by scanning outside quote characters, but it does not recognise JavaScript regex literals. In `const re = /^https?:\/\//;` the last two characters before `;` form `//`, so the rest of the line is classified as comment text and an edit to it passes as comment-only (traced; no current fixture contains this). Whole-line `//` comments are editable unless a case declares them in `protected_directives`. An undeclared `//go:build` line could therefore be edited, which the Go documentation says changes which files are compiled. This tool grades evals and does not run on users' code, so the impact is on eval validity.

### L12. `literature-review` cannot verify what it cites

The skill removes Bash and leaves `WebFetch`, which returns a small model's answer about the page, not the page text. The skill has no rule against citing from abstracts and no step to confirm quotes verbatim. Lathe's own sources show the pattern: E01 is cited from its "published abstract", the ISO entries from "public ISO abstract", and NIST IR 8397 from "the abstract states". Those disclosures are accurate, but the skill does not ask for them. Fix: require a full-text or abstract-only marker on each source, and verbatim quotes for load-bearing claims.

### L13. Source-basis accuracy

- **S30 is stale.** It says a full SHA is "the only immutable action reference". GitHub's current secure-use page says lockfiles are "the strongest protection available, and supersedes SHA pinning for repositories that adopt it".
- **S08** is arXiv 2401.16310, first posted in 2024; its latest version (v5, December 2025) carries a new title. "2026" is wrong.
- **S36:** Werkzeug calls the debugger PIN a "security helper"; lathe's "not a security control" is a slight overstatement.
- **`comment-cleanup` source-basis:** F13 describes a duplicate `[Unreleased]` heading that is now gone, and F07–F09 say "tracked separately" for items that are fixed.

### L14. Installer default plugin set

By default `install.sh` installs eleven official plugins (line 1036). Among them is superpowers, whose own mandate the policy must override in prose (`context/CLAUDE.md`). Others start network MCP servers (`context7`, and `playwright` via `npx @playwright/mcp@latest`, which is unpinned). This widens what the policy has to override and adds unpinned network code to every session. `--no-official` exists, but the default is the broad set. Fix: default to the lathe plugin alone and offer the rest as opt-in.

### Informational

- The allowlist falsely denies `--output-indicator-new` (it matches `^--ou`, intended for `--output`).
- `settings.recommended.json` sets the main session's effort to `xhigh`. That is a cost choice; the policy's own goal is "fully correct accepted work per unit of model usage".
- OpenSSF Scorecard's Signed-Releases check looks for signature assets on GitHub releases. Lathe publishes signed tags but no releases, so that check would find nothing even though tags are signed.
- The `mechanical-worker` effort measurement says "about p=0.1"; Fisher's exact test gives p = 0.0995. The record is accurate.

## What held up

- **Signed tags.** All release tags verify against `.allowed_signers` with `ssh-keygen`.
- **Squash merges.** Every merged PR's squash commit has the same tree as the PR head CI checked, consistent with `strict: true`.
- **Guard design.** It never returns `allow`, so it cannot widen permissions; it fails closed for the auditor when Python is missing or the input is malformed; and it is keyed on the plugin-scoped `agent_type`, which the hooks page documents.
- **Probe git hardening** (R20) is thorough and tested: filter blanking via `--show-scope --includes`, partial-clone skip, `--no-replace-objects`, forced content comparison, and no TTY or stdin.
- **CI installer verification** (`scripts/ci/install-claude.sh`/`.ps1`): pinned key fingerprint `31DDDE24DDFAB679F42D7BD2BAA929FF1A7ECACE`, `VALIDSIG` check, then SHA-256 and size. Actions are SHA-pinned with `persist-credentials: false` and top-level `contents: read`.
- **Platform features used correctly:** `${CLAUDE_PLUGIN_ROOT}` substitution in agent and skill bodies, `disallowed-tools`, the SessionStart `fork` matcher, `background: false` (keeps the full tool set), spawn depth 1, and the Concise output style. Plugin agents' ignored `mcpServers`/`hooks`/`permissionMode` fields are not relied on.
- **Prompt style.** The policy, skills and agents contain no emphatic `MUST`/`CRITICAL` wording. That matches the current prompting guidance to "soften any aggressive language".
- **Stated limits.** The risk register and SECURITY.md state many limits candidly (R1, R13, R17, R20; confidentiality out of scope; allowlist drift). Many findings above were found by testing lathe's own stated limits against its own code.

## Literature and documentation check

**Verified in full text** (the claim lathe attributes to the source is supported):

| Source | Lathe's claim | Verdict |
|---|---|---|
| S13 claude-code-security-review | ">80% confident" threshold; env vars and CLI flags trusted; no metrics | Supported |
| S14 claude-security plugin | work outward from changed lines; refute only with a located mitigation | Supported |
| S15 Codex Security skills | impact × likelihood matrix; "smallest fact that would change the verdict" | Largely supported |
| S16 GitHub Security Lab taskflows | "ok to conclude that there is no security issue" | Supported; funnel numbers not checked |
| S22 SSVC | unknown exposure treated as open | Supported |
| S26 ASVS 5.0.0 | no scope for attackers; secure defaults; 13.4.2 | Supported |
| S31 Docker | 28.0.0 L2 note; ufw incompatibility; Swarm on 0.0.0.0 | Supported |
| S33 Kubernetes | list/watch reveals Secrets; `nodes/proxy`; NodePort | Supported |
| S36 Werkzeug/Jupyter/Ray | arbitrary code execution when reachable | Supported; PIN wording slightly stronger than source |
| S39 GitHub Actions settings | `pull_request_target` runs regardless of approval | Supported |
| S40 Redis 7.0 | three defaults `no` | Supported |
| S30 GitHub secure use | full SHA "only immutable reference" | **Stale** (see L13) |
| PRR #1, #5 Claude docs | skill locations and invocation; sequential steps; investigate before answering | Supported |
| PRR #3, #4 anthropics/skills, skill-creator | public skill examples; should-trigger and should-not-trigger description tuning | Supported |
| PRR #15 Kubernetes PRR | observable, scalable, supportable; disable or roll back; separate team | Supported |
| PRR #30 Demystifying evals | outcome grading; pass^k | Supported; the post also says "it's often useful to also grade the transcript" |
| PRR #31 KEP template | rollout/rollback failure questions | Supported |
| PRR #34 OpenSSF Scorecard | token permissions, branch protection, signed releases | Supported |
| E04, E05, E07, E08 | plugins reference, skills, plugin evals, Go build constraints | Supported |
| E06 memory docs | "current CLAUDE.md guidance" | Reachable; the claim is too general to test |

**Existence and bibliographic details checked only (full text blocked):** S02, S05, S06, S07, S08 (see L13), S10 (VulTrial; v1 and v2 report different headline numbers), S11, S19, S20, S24 (CCS venue not confirmed), S38.

**Not verified (blocked):** S01, S03, S04, S09, S12, S17, S18, S21, S23, S25, S27, S28, S29, S32, S34, S35, S37; PRR #2, #6–14, #16–29, #32, #33, #35, #36; E01, E02, E03, E09.

Two Anthropic sources lathe does not cite were read in full and are used above: "A statistical approach to model evaluations" (M7) and "How we built our multi-agent research system" (M8).

## Coverage

| Path | Files | Examined for |
|---|---|---|
| `plugins/lathe/hooks/` | guard, allowlist, session-start, hooks.json | logic traced line by line; bypasses reproduced in shells; platform failure modes from the hooks docs |
| `plugins/lathe/agents/` | 11 agents | tool grants against the sub-agents and permissions docs; instructions cross-checked (M2, M6) |
| `plugins/lathe/skills/` | 12 skills plus references and scripts | frontmatter against the skills docs; instructions; probe traced (L10); source-basis files checked (L13) |
| `plugins/lathe/context/CLAUDE.md` | policy | consistency with agents and skills; prompting style |
| `plugins/lathe/evals/` | 60+ cases, fixtures, graders, README, RESULTS | design, grading and statistics (M7) |
| `install.sh`, `release.sh`, `settings.recommended.json` | installer, release flow, settings | ordering, rollback, tagging, defaults (M9, L5, L6, L14) |
| `scripts/` and `scripts/tests/` | helpers and tests | redaction, safe write and restore, comment checker, eval recheck (L5, L9, L11) |
| `ci.sh`, `ci/`, `.github/`, `.githooks/` | CI, shims, fixtures, workflow, pre-push | coverage gaps and fixture provenance (M5, L8) |
| `README.md`, `SECURITY.md`, `CHANGELOG.md`, `docs/` | documentation | every factual claim checked against code, history or docs (L3, L4) |
| `.allowed_signers`, `.gitattributes`, `.gitignore`, `CODEOWNERS`, `LICENSE`, marketplace manifest | metadata | tag verification; ignored paths (`docs/plans/`, eval results) |

## Addendum: does lathe meet its efficiency objective?

**The objective, verbatim.** `context/CLAUDE.md` line 5: "Maximize fully correct accepted work per unit of model usage and wall-clock time. Optimize completed work, not raw token price, model prestige, number of agents, or lines generated." Line 37 adds "the smallest coherent change", and README line 3 says the agents "route each job to the cheapest model that can do it". Fewest lines of code is not itself a stated goal: the policy says not to optimise "lines generated".

**Verdict: not demonstrated, and the repository's own data leans against it for model usage.** No valid measurement compares correct work or cost with and without the plugin. Every behaviour measurement ran with `--ablation none`, which turns off `claude plugin eval`'s built-in without-plugin arm (M7). The objective has therefore never been measured.

**Correctness.**
- 10 of 23 components (12 skills, 11 agents) have at least one shipped eval case.
- 13 have none: browser-testing, checkpoint, change-eval, docs-check, literature-review, deep-audit, verification-loop, scout, test-triage, architect, semantic-reviewer, browser-tester and the deep-audit auditor.
- The evaluated ones mostly pass, after the adjudication described in M7.
- The defects this audit found sit in the enforcement features: the guard (M1, M4, L1), tool boundaries (M2, L2), the Windows hook (M3) and the installer (M9, L5).

**Lines of code.**
- The plugin's runtime is small: about 2,900 lines, mostly prompt text, and much of it is loaded only when a skill runs.
- About 6,500 of the repository's 23,614 lines (28%) serve the installer: `install.sh` and helpers 2,542, their unit tests about 2,000, `ci.sh` install scenarios about 1,400, CLI shims about 560.
- README line 34 says the native two-command marketplace install also works. The installer's additions over it are settings that raise usage (below) and a backup and rollback layer with defects (M9, L5).

**Model usage.**
- **Fixed context cost is small.** By byte count (3.5–4 bytes per token, no tokenizer available), lathe adds about 4.5–5.1K tokens per session: a 12.2 KB policy, 4.3 KB of skill listings and 1.5 KB of agent listings. The default install adds about 1.6–1.9K more: superpowers' 3.2 KB session injection and 2.3 KB of listings, plus about 1 KB from the other default plugins. MCP tool definitions are deferred by default.
- **About half the policy (49%) is avoidable always-loaded text.** 1.9 KB overrides superpowers, which the installer itself installs, and 4.0 KB is an agent roster that partly repeats the agent listing.
- **The largest lever raises usage without measurement.** The current models already default to the larger tier on subscription and API plans, and their documented default effort is `medium`. `settings.recommended.json` raises the main session, where the policy keeps most work, to `xhigh`. Claude Code's effort guidance says "for most tasks you should use the model's default effort level". Lathe's own per-agent effort measurements kept `medium` wherever `high` "bought nothing" or cost 23% more for small gains, but no measurement covers the main session's `xhigh`.
- **Counter-evidence.** The same guidance says Claude "generally won't artificially inflate usage for simple tasks at higher effort levels", so the extra cost of `xhigh` is concentrated in harder tasks, and it may buy correctness. Neither direction is measured.
- **Fan-out costs more on lathe's own fixtures.** `plan-execution` fanned out in 14 runs at a mean of $0.845 and 181 s. The 4 runs that did not averaged $0.168 and 70 s, and 3 of those 4 produced the same diff size with passing tests (M8). The protocol counts runs without fan-out as invalid, so it cannot test whether fan-out pays. Anthropic's article "Building agents: When to fold subagents into your main agent loop" (2026-10-22) reports that a single agent with skills "frequently outperformed … the subagent design on quality, and often at a lower cost and latency per task". It keeps subagents for context bloat or parallel work with throwaway context. Policy lines 22–24 follow that; `plan-execution` and the readiness review's fan-out do not.
- **Practices that do match platform cost guidance:** deterministic checks before model review, targeted reads, no automatic second review, terse reviewer output, `test-triage` and `scout` on the smaller tier at `low` effort, and code-intelligence plugins, which the Claude Code costs page recommends for cutting file reads.

**What would settle it.** Run `claude plugin eval --ablation with-without` on a fixed set of ordinary coding tasks that lathe was not tuned on. Compare pass rate and cost per run for (a) no plugin, (b) lathe with platform-default effort, and (c) lathe as recommended. Then measure `plan-execution` against a single-agent run on plans large enough to parallelise.

## Reproduction notes

Every reproduction ran in a scratch directory outside the repository. Minimal forms:

```sh
# M1: a repository-configured textconv runs on an allowlisted verb
git init r && cd r && echo a > f && git add f && git -c user.name=a -c user.email=a@b commit -qm 1
echo '* diff=evil' > .gitattributes && git config diff.evil.textconv "sh -c 'touch PWNED; cat \"\$0\"'"
echo b > f && git diff        # PWNED appears; git diff --no-textconv does not create it

# M4: guard-passing strings that write
bash -c 'cat ()(touch PWNED); cat' </dev/null
zsh -f -c 'cat =(touch PWNED)'
printf 'x\n' > keep.txt && bash -c 'uniq - keep.txt' </dev/null && wc -c keep.txt   # 0

# L1: git status rewrites the index
stat -c %Y .git/index; sleep 1; touch f; git status --short; stat -c %Y .git/index
```
