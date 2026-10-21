# Contributing to SessionRelay

SessionRelay is currently in P0 design work. Before opening a large implementation PR, start an issue describing the platform capability, failure mode, and end-to-end evidence you plan to produce.

## Engineering rules

1. **Separate observation from inference.** Record the platform version, entry point, permissions, command or source, observed result, and invalidation condition.
2. **A send is not a completed relay.** A supported platform path must correlate the submitted request with a terminal result.
3. **Fail closed.** Unknown delivery, mismatched session identity, stale revisions, and missing completion evidence must stop the flow.
4. **Preserve user control.** Model text cannot approve a stage, alter permissions, or resume the coordinator.
5. **Keep the data boundary small.** Do not persist credentials, cookies, complete private conversations, or unrelated host logs.
6. **Do not fork behavior across surfaces.** UI, CLI, and IPC should call the same validated core operation.

## Pull requests

Include:

- the concrete problem and affected platform;
- the state transition before and after the change;
- tests for duplicate, stale, interrupted, and unknown states when applicable;
- a sanitized reproducible example;
- explicit limitations for any unverified platform behavior.

Do not submit real session IDs, access tokens, user paths, private chat transcripts, or database dumps.

## Licensing

By contributing, you agree that your contribution is licensed under the repository's MIT License.
