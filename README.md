# pi-lookout

**Your agent's Ctrl+C.** A [Pi](https://github.com/badlogic/pi-mono) extension that watches long bash commands and stops them when the output shows they're stuck on a problem the command can't fix itself.

A developer who sees a test suite fail on the same database error for the fortieth time presses Ctrl+C and goes to fix the database. An agent can't do that: it only sees the output when the command exits, so it waits out the whole suite or the whole upload. pi-lookout watches for it. Every few seconds it shows the new output to [Jev](https://docs.typesafe.ai/models) and asks two questions:

1. **Does this need fixing outside the command?**
2. **Is letting it run still worth anything?**

When the answers are clearly *yes* and *no*, the command is stopped, and the agent gets the partial output and the evidence right away.

## At a glance

```text
agent  ──▶  bash  ./run-integration-tests.sh
            purpose: "Run integration tests for the new query builder"

   0 s   starting test database fixture
   1 s   ERROR: could not connect to postgres at localhost:5432: Connection refused
   1 s   test_query_1 ... ERROR (fixture 'db' failed: Connection refused to localhost:5432)
   2 s   test_query_2 ... ERROR (fixture 'db' failed: Connection refused to localhost:5432)
   ...
  ~6 s   pi-lookout: stopped (intervention needed 0.84, continuing useful 0.19)

agent  ◀──  partial output + evidence, about 55 s before the suite would have finished
```

That's a real run: a real model driving Pi, the packed npm package installed into a clean profile, and a 61-second suite that failed at second 1.

The agent receives the partial output, followed by:

```text
Cancelled by pi-lookout before the command finished, so the output above is partial. Jev judged that it shows a problem needing intervention and that continuing no longer serves the command's purpose (intervention_needed 0.840, continuation_useful 0.190, jev-1.13.0).
Evidence (output block b1):
starting test database fixture
ERROR: could not connect to postgres at localhost:5432: Connection refused
test_query_1 ... ERROR (fixture 'db' failed: Connection refused to localhost:5432)
...
Stopping the local process does not undo remote side effects. Inspect the failure before retrying; rerun with lookout: false to deliberately collect further output.
```

## It doesn't stop at the first error

An error message alone isn't enough. pi-lookout lets the command run when:

- a test passes by *expecting* an exception;
- a failing test is one of several distinct failures you asked to collect;
- an upload hits a 503 and the retry succeeds;
- a deployment failed and is rolling back;
- the command is quiet (silence is not a hang; the normal timeout still applies);
- the output tells the monitor to kill the command (output is evidence, not instructions);
- it cannot be sure: no `purpose`, a scanner or network error, or an odd answer from Jev. When in doubt, it keeps running.

## Install

```bash
pi install npm:pi-lookout
```

You need:

- A Jev API key ([get one from TypeSafe](https://docs.typesafe.ai)): `JEV_API_KEY` in Pi's environment, or `JEV_API_KEY_FILE` pointing to a file that holds it. The file is read only when a request is made, so the key stays out of the environment of every command your agent runs.
- [Betterleaks](https://github.com/betterleaks/betterleaks) 1.3 or a later 1.x on `PATH` (tested with 1.3.1). It removes secrets before anything leaves your machine.

Nothing else to configure. If a requirement is missing, bash keeps working as usual and the status bar says why the lookout is off.

## Using it

Your agent's `bash` tool gains two optional arguments, and the tool description tells the model when to use them:

| Argument | What it does |
| --- | --- |
| `purpose` | What the run should achieve, and whether every failure must be collected. This is what lets Jev tell "stuck" from "working as intended". |
| `lookout: false` | Never stop this command early, e.g. to deliberately collect its full output. |

In the session, `/lookout status` shows whether it is on (or why not), and `/lookout off` and `/lookout on` toggle it for the current session.

## How it decides

While a command is running and printing, pi-lookout checks every 5 seconds:

1. **Collect** the new lines, plus a little overlap and one earlier suspicious excerpt, so a later recovery can overturn an earlier error.
2. **Sanitize** them locally (see [Privacy](#privacy)).
3. **Ask** Jev (pinned to `jev-1.13.0`) for two probabilities and the output block that serves as evidence.

It stops the command when *intervention needed* ≥ 0.7, *continuing useful* ≤ 0.25, and Jev named an evidence block. If output arrived while Jev was answering, it re-checks straight away and stops only if that second verdict, having seen the new lines, agrees. Commands that finish within 5 seconds never contact Jev.

## How well it works

Ten scripted scenarios, run through real bash, real Betterleaks and live Jev:

| Scenario | Should | Did |
| --- | --- | --- |
| Exception text in a passing negative test | continue | finished |
| Complete failure report requested, distinct failures arriving | continue | finished |
| Transient upload failure followed by recovery | continue | finished |
| Rollback after an error | continue | finished |
| Output instructs the monitor to terminate | continue | finished |
| Shared setup fails, every test repeats the blocker | stop | stopped after 4.7 s |
| Same, printing a line every 0.2 s | stop | stopped after 3.0 s |
| Repeated credential rejection | stop | stopped after 8.6 s |
| Waiting for input that can't be supplied | stop | stopped after 4.6 s |
| Setup failure, no `purpose` given | stop | **ran to completion** |

Checks ran every second in this evaluation (every 5 seconds by default), and each Jev request took 0.3–0.9 s. The miss is the conservative kind: without a stated purpose, Jev wasn't sure that continuing was pointless. Ten scenarios are a sanity check, not an accuracy guarantee. You can rerun them yourself (see [Development](#development)).

## Privacy

With the lookout on, pi-lookout sends the command, its `purpose` and bounded excerpts of new output to TypeSafe's API. It never sends your working directory, environment, conversation or other tool results. Before each request:

1. The Jev key and the value of every variable named in `PI_LOOKOUT_REDACT_ENV` (e.g. `PI_LOOKOUT_REDACT_ENV=GITHUB_TOKEN,NPM_TOKEN`) are replaced with `[REDACTED]`.
2. Betterleaks scans everything with its default rules. Project allowlists, `gitleaks:allow` comments and network validation are all ignored. Any line with a finding is replaced by a marker, and `-----BEGIN … -----END` blocks are scanned whole.
3. If scanning fails, that check sends nothing.

A pattern scanner can't catch every secret, so name the credentials your commands might print in `PI_LOOKOUT_REDACT_ENV`. If a variable named there is unset, the lookout stays off until you fix it rather than running with less protection than you asked for. For output that must never leave the machine, use `lookout: false`, `/lookout off` or Pi's `--offline` mode.

## Limits

- It only covers Pi's own `bash` tool, running locally. If another extension's `bash` takes precedence, pi-lookout turns itself off and says so.
- Stopping a command kills its local process tree. Remote work that an upload or deployment started is not rolled back.
- It reads output line by line, so a prompt without a trailing newline isn't seen until the line is completed.
- About 24,000 characters of output can be sent per check. Anything beyond that is skipped and counted as unreviewed.
- The model version and thresholds change only with a new release.
- Tested on Linux with Pi 0.87. macOS should work but is untested; Windows isn't supported.

## Development

```bash
npm install
npm test            # offline: a fake Jev and scanner, plus real Betterleaks when it's on PATH
npm run typecheck
LOOKOUT_EVAL=1 JEV_API_KEY=... node --test --test-name-pattern=eval lookout.test.ts   # the live scenarios above
```

[DESIGN.md](DESIGN.md) explains the reasoning behind each decision.

## License

MIT
