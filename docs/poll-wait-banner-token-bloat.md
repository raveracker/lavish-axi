# Poll wait-banners bloat the harness capture and waste agent input tokens

## Summary

When an agent runs `lavish-axi poll <file>` and the wait lasts a while, the returned
output contains dozens of `[lavish-axi] Still waiting...` progress lines in addition to the
useful payload. An agent harness captures the process output and feeds it back into the model
as input tokens, so those progress lines cost real tokens for zero agent value. A single
35-minute poll measured ~2.2k wasted input tokens.

## Root cause

The `.output` file an agent reads (for example `b464rkzgy.output`) is the harness's
**background-task capture**, and the harness merges **stdout and stderr into one stream**.

`lavish-axi poll` correctly separates the two:

- **stdout** carries only the final TOON payload (`prompts[...]` + `next_step`) - the part the
  agent needs.
- **stderr** carries the wait narration: an initial `Long-polling for user feedback...` banner
  plus one `Still waiting for user feedback (Nm)...` line every 60 seconds.

The stderr banners were written **unconditionally** (`src/cli.js` `startPollWaitReporter`), so
even though the split keeps stdout clean, the harness recombines the streams and every tick
lands back in the agent's context. The agent never sees the interim stderr live - it reads the
whole merged capture once, at completion - so the ticks provide no liveness signal to the
agent; they only accumulate as tokens. Their real audience is a human watching a terminal.

## Byte / token chart (35-minute poll, the reported example)

```
                bytes in the merged .output capture
useful payload  ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  ~0.9 KB   (prompts + next_step)
stderr banners  ████████████████████████████████████  ~6.3 KB   (1 banner + 35 ticks)  <- pure noise
```

| Segment                 | Lines | Approx bytes | Approx tokens | To the agent |
| ----------------------- | ----- | ------------ | ------------- | ------------ |
| Useful payload (stdout) | 4     | ~0.9 KB      | ~0.3k         | required     |
| Wait banners (stderr)   | 36    | ~6.3 KB      | ~2.2k         | wasted       |
| **Total before fix**    | 40    | ~7.2 KB      | ~2.5k         | -            |
| **Total after fix**     | 4     | ~0.9 KB      | ~0.3k         | -            |

Waste scales linearly with wait time (one tick per minute), so long reviews cost the most.

## Example

### Before (40 lines returned to the agent)

```
prompts[2]{uid,prompt,selector,tag,text}:
  "1","Proceed with Option A now..."
  "2","For the capacity mismatch banner..."
next_step: "Apply the requested changes to ...
[lavish-axi] Long-polling for user feedback ...
[lavish-axi] Still waiting for user feedback (1m). ...
[lavish-axi] Still waiting for user feedback (2m). ...
   ... 33 more tick lines ...
[lavish-axi] Still waiting for user feedback (35m). ...
```

Lines 5-40 are pure noise the agent re-reads.

### After (4 lines, banners gone)

```
prompts[2]{uid,prompt,selector,tag,text}:
  "1","Proceed with Option A now..."
  "2","For the capacity mismatch banner..."
next_step: "Apply the requested changes to ...
```

Byte-identical payload; every `[lavish-axi]` line removed.

## Resolution

Gate the wait reporter on whether stderr is an interactive terminal. Agent harnesses run the
CLI with piped, non-TTY stdio, so they get silence; a human running the poll in their own
terminal keeps the full narration. This is vendor-neutral - it does not special-case any
particular agent harness.

```js
export function shouldNarratePollWait({ timeoutMs, isTTY }) {
  return !timeoutMs && Boolean(isTTY);
}

const waitReporter = shouldNarratePollWait({ timeoutMs, isTTY: process.stderr.isTTY })
  ? startPollWaitReporter({ file: absolute })
  : null;
```

### Unchanged

- The final TOON payload on stdout (`prompts` + `next_step`).
- The SIGINT/SIGTERM interrupt guidance (`pollInterruptedText`), which stays unconditional: one
  line, only on a kill, carrying the actionable re-run instruction.
- The whitespace heartbeat bytes, which travel over the HTTP wire (server to CLI) and never
  reach the CLI's own stdout.
- The `--timeout-ms` path (reporter already skipped there).

## Tests

- `shouldNarratePollWait` unit test: narrates only when `isTTY` and no `--timeout-ms`.
- Rewrote the spawned-poll test to assert that a poll with piped stderr stays silent while
  still leaving re-run guidance on kill, synced on the agent-presence stream.

## Files changed

- `src/cli.js`
- `test/cli-output.test.js`
