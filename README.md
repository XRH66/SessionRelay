# SessionRelay

**Relay work between two existing AI sessions without rebuilding either chat UI.**

SessionRelay is a proposed local-first Windows desktop application for connecting two AI sessions, for example a Codex implementation session and a DeepSeek Harness review session. One side produces a validated handoff envelope; SessionRelay records the transition, requests delivery to the other side, waits for a correlated result, and pauses whenever the outcome is uncertain.

> **Project status: P0 — research and architecture only.**
>
> There is no runnable application yet. The repository currently contains the product contract, architecture, platform research, implementation stages, and sanitized interface findings. P1 has not started.

## Why

People already use capable desktop and CLI agents. The missing piece is a controlled bridge between their existing conversations:

- preserve the sessions and tools users already rely on;
- turn chat output into a typed, reviewable handoff;
- correlate submission, completion, and the returned result;
- pause safely when identity, delivery, or state is ambiguous;
- keep credentials and private conversation history out of logs and storage.

SessionRelay is not another agent runtime. It is coordination software around existing runtimes.

## Planned scope

- Windows desktop application using TypeScript, Electron, React, and SQLite.
- Codex and DeepSeek Harness as the first target platforms.
- Connections between two existing sessions, including same-platform sessions.
- Three independent collaboration modes: automatic relay, human intervention, and manager-driven dispatch.
- Structured handoff envelopes, durable outboxes, delivery receipts, round-trip limits, and explicit user approval for stage transitions.
- System tray operation, pause/resume controls, and a visible approval queue.
- No HTTP listener, background system service, startup task, or arbitrary renderer command execution.

## Safety model

- Credentials are held only in process memory and are never persisted.
- Session titles are display labels; identity uses platform, instance, and session IDs.
- A successful send is not proof of completion.
- Delivery-to-result correlation is required before work advances.
- Unknown delivery state locks the session and requires human reconciliation.
- Model output cannot approve a stage, change permissions, or resume a pause.
- The project does not promise exactly-once delivery; it promises detectable duplication and no blind resend.

## Platform status

| Platform | P0 conclusion |
| --- | --- |
| Codex | Queue wake-up was demonstrated. Completion/result correlation through a stable external connection remains unverified. |
| DeepSeek Harness | Start/send/status/result paths expose request-based correlation in the investigated local CLI. A formal session-enumeration path is still missing there. |
| Antigravity | Official headless CLI documentation exists, but no stable local entry point was found during P0. Treat as blocked. |
| ChatGPT ordinary chats | Candidate only and disabled by default. No verified generic external send/complete/correlate path. |

See [docs/platform-contracts.md](docs/platform-contracts.md) for evidence levels and limitations.

## Documents

- [Product definition](ProjectContext.md)
- [System architecture](docs/architecture.md)
- [Platform contracts](docs/platform-contracts.md)
- [Implementation phases](docs/phases.md)
- [Sanitized P0 findings](docs/p0-evidence-summary.md)
- [Open-source references](docs/references.md)
- [LoopX architecture analysis](docs/references/loopx-analysis.md)

## Development status

P0 is complete enough to preserve the design and expose the unresolved platform questions. P1 will begin only after an explicit decision to implement the smallest Codex-to-DeepSeek Harness relay and prove the completion chain end to end.

## Contributing

Read [CONTRIBUTING.md](CONTRIBUTING.md) before proposing platform adapters or changes to the state machine. Contributions that claim support must include a reproducible completion-chain test, not only a successful send.

## Security

Please report credential leaks, unsafe command surfaces, or accidental session-history persistence according to [SECURITY.md](SECURITY.md).

## License

MIT. See [LICENSE](LICENSE).
