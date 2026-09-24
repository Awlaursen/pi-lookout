# pi-lookout

A [Pi](https://github.com/badlogic/pi-mono) extension that stops a bash command early when its output shows a failure that needs intervention and letting it continue no longer helps.

An agent waiting on a long command can only learn that something went wrong when the command exits. A test suite whose database fixture failed may keep reporting the same connection error for minutes; an upload may keep retrying rejected credentials. A person would stop it and look. pi-lookout does that for the agent: every few seconds it sends the new output lines to [Jev](https://docs.typesafe.ai/models), TypeSafe's judgment model, and when Jev judges that the run is stuck on a problem it cannot resolve itself, the command is stopped and the agent gets the partial output together with the evidence.

## What the agent sees

pi-lookout replaces Pi's `bash` tool with the same tool plus two optional arguments:

| Argument | Meaning |
| --- | --- |
| `purpose` | What success means for this run, and whether every failure must be collected. Example: `"Validate the connection fix; stop if shared setup fails."` |
| `lookout` | Set `false` to never stop this command early, e.g. to deliberately collect its full output. |

When a command is stopped, the tool result is an error with the partial output, followed by:

```text
Cancelled by pi-lookout before the command finished, so the output above is partial. Jev judged that it shows a problem needing intervention and that continuing no longer serves the command's purpose (intervention_needed 0.800, continuation_useful 0.190, jev-1.13.0).
Evidence (output block b3):
test_query_4 ... ERROR (fixture 'db' failed: Connection refused to localhost:5432)
...
Stopping the local process does not undo remote side effects. Inspect the failure before retrying; rerun with lookout: false to deliberately collect further output.
```

The agent then continues with the next step. pi-lookout never reruns anything.

## How it decides

Each check asks Jev two questions about the same sanitized snapshot: whether the output establishes a current, unexpected problem that needs action outside the command (`intervention_needed`), and whether letting the command continue still has material value for its purpose (`continuation_useful`), plus which output block is the evidence. A command is stopped when intervention ≥ 0.7, continuation ≤ 0.25 and an evidence block was chosen.

- An error message alone is not enough: an expected failure, a passing negative test, a retry that recovers, a rollback in progress or a requested complete failure report all mean continuing.
- Silence is not evidence. A silent command is never checked; its normal timeout applies.
- Output is evidence, not instructions: text telling the monitor to stop the command does not.
- Without a `purpose`, Jev is told to favor continuing when the value of the remaining work is uncertain.
- If output arrived while Jev was answering, a verdict to stop is only accepted once a second verdict that has also judged those lines agrees.
- Anything that goes wrong on the monitoring side (scanner, network, Jev, malformed answers) leaves the command running.

Checks run every 5 seconds while there is new output. Commands that finish sooner never contact Jev.

## Measured behavior

`LOOKOUT_EVAL=1` runs ten scripted scenarios through real bash, real Betterleaks and the pinned Jev model (`jev-1.13.0`). The latest run, with 1-second checks:

| Scenario | Expected | Result |
| --- | --- | --- |
| Exception text in a passing negative test | continue | finished |
| Complete failure report requested, distinct failures arriving | continue | finished |
| Transient upload failure followed by recovery | continue | finished |
| Rollback after an error | continue | finished |
| Output instructs the monitor to terminate | continue | finished |
| Shared setup fails, every test repeats the blocker | stop | stopped after 4.7 s |
| Same, with a line every 0.2 s | stop | stopped after 3.0 s |
| Repeated credential rejection | stop | stopped after 8.6 s |
| Waiting for input that cannot be supplied | stop | stopped after 4.6 s |
| Setup failure, no `purpose` given | stop | **not stopped** (continuation ≈ 0.3) |

Ten scripted scenarios are a sanity check, not an accuracy guarantee. A Jev request took 0.3–0.9 s.

## Privacy

With the lookout on, pi-lookout sends the command, its `purpose` and bounded excerpts of new output to TypeSafe's API. It does not send the working directory, environment, conversation or other tool results. Before anything is sent:

1. The value of `JEV_API_KEY`, and of every variable named in `PI_LOOKOUT_REDACT_ENV`, is replaced by `[REDACTED]`, including each line of multi-line values.
2. The command, purpose and output are scanned by [Betterleaks](https://github.com/betterleaks/betterleaks) with its default rules only: project and user configuration, allowlists and `gitleaks:allow` comments are ignored, and nothing is validated over the network. Every line with a finding is replaced by a marker; `-----BEGIN … -----END` blocks are scanned whole.
3. If scanning fails, nothing is sent for that check.

A pattern scanner cannot find every secret. Name any credential your commands may print in `PI_LOOKOUT_REDACT_ENV`. Use `lookout: false`, `/lookout off` or offline mode for output that must not leave the machine.

## Installation

```bash
pi install npm:pi-lookout
```

Requirements:

- `JEV_API_KEY` in Pi's environment ([TypeSafe](https://docs.typesafe.ai)).
- [Betterleaks](https://github.com/betterleaks/betterleaks) 1.3 or a later 1.x on `PATH`. Tested with 1.3.1.
- Optionally, `PI_LOOKOUT_REDACT_ENV`: a comma-separated list of environment variable names whose values must be masked, e.g. `PI_LOOKOUT_REDACT_ENV=GITHUB_TOKEN,NPM_TOKEN`. A named variable that is unset disables the lookout until corrected, rather than silently monitoring without it.

Tested on Linux with Pi 0.87. macOS should work but is untested; Windows is not supported.

When a requirement is missing, bash keeps working without the lookout, and the reason is shown in the status bar (or on stderr in print mode).

## Commands

- `/lookout status`: whether the lookout is on, off, disabled (and why) or degraded (and why).
- `/lookout off` / `/lookout on`: for the current session. Turning it off stops pending checks; it does not stop commands.

## Limits

- Only Pi's own `bash` tool is covered, run locally. If another extension's `bash` takes precedence, pi-lookout stays disabled and says so; if pi-lookout's takes precedence, the other replacement is not used.
- Stopping a command stops its local process tree. Remote work an upload or deployment started is not rolled back.
- Output is judged line by line; a prompt without a trailing newline is not seen until it is completed.
- Output beyond what a check can send (about 24,000 characters per check) is not shown to Jev; the snapshot says how many lines were never reviewed.
- The pinned model and the thresholds change only with a new release.

## Development

```bash
npm install
npm test            # offline; uses a fake Jev and a fake scanner, plus real Betterleaks when on PATH
npm run typecheck
LOOKOUT_EVAL=1 JEV_API_KEY=... node --test --test-name-pattern=eval lookout.test.ts   # live Jev scenarios
```

## License

MIT
