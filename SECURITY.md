# Security policy

Do not open a public issue containing credentials, cookies, live session IDs, personal conversation content, or host database files.

Report a suspected vulnerability privately to the repository owner using GitHub's private vulnerability reporting or security advisory feature when available. Include:

- affected component and version or commit;
- impact and required preconditions;
- a minimal sanitized reproduction;
- whether credentials or private conversation data may have been exposed.

SessionRelay is expected to fail closed when delivery or result correlation is uncertain. Reports that demonstrate blind resend, permission escalation from model output, arbitrary renderer command execution, or persistence of authentication material are security-relevant.
