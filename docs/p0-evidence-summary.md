# P0 evidence summary

This public summary replaces the local raw probe files. It intentionally omits machine paths, live session identifiers, dispatch records, credentials, and private conversation content.

## Evidence levels

- **Official public:** vendor documentation, public protocols, and published source code.
- **Observed locally:** an entry point or behavior was exercised on one Windows host during P0.
- **Partial:** only part of the submit → complete → correlate chain was observed.
- **Unverified:** a protocol or documentation path exists, but the complete behavior was not exercised.

## Codex

Observed locally:

- a Codex CLI was available and its queue help exposed a command for queueing a message into a thread;
- queueing a probe into an existing thread produced a receipt, and the queued message was observed on the following turn after the current turn ended;
- an ephemeral `codex exec` run created an isolated temporary session and completed without modifying an existing desktop thread;
- the locally generated app-server schema exposed thread enumeration/read/resume, turn start/steer/interrupt, completion notifications, approval requests, and a nullable client message identifier.

Not established:

- that a separate SessionRelay connection can safely attach to an already running desktop thread;
- that the queue receipt can be correlated to a specific terminal turn and result;
- that completion notifications would arrive on SessionRelay's own connection;
- that external human input can be distinguished reliably.

Conclusion: **partial verification; completion correlation remains a P1 gate.**

## DeepSeek Harness

Observed locally:

- the investigated CLI exposed start, send, status, and result operations;
- start/send returned a session identifier and request identifier;
- result lookup used the request identifier and exposed terminal state and final output;
- status exposed agent state, turn state, queue count, and pending approval count.

Not established:

- a formal session-enumeration operation in that CLI;
- approval actions through the same interface;
- direct detection of unrelated human input.

Conclusion: **usable request-based correlation for the investigated adapter, with session discovery and approval UX still open.**

## Antigravity

Official documentation describes a headless CLI with structured session output and conversation resumption. The researched Windows host did not expose a verified stable CLI entry point.

Importing a desktop conversation into a CLI session is a history clone; it is not proof that the original desktop session is externally controllable.

Conclusion: **blocked for automatic relay.**

## ChatGPT ordinary chats

No verified generic path was found for selecting an existing ordinary chat, sending from an external local program, waking the chat in the background, and correlating a terminal result.

Conclusion: **candidate only and disabled by default.**

## Reproduction principle

Future probes should be read-only by default and publish sanitized method-level findings. Re-run platform probes when the platform or adapter version changes.
