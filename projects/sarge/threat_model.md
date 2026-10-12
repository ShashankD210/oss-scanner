# Threat model — Sarge

## What the project does

Sarge is an agent-driven NIST SP 800-53 hardening and assessment tool for OpenClaw hosts. It
ships as bash scripts plus a Claude Code agent skill (`SKILL.md`): assessment scripts
(`assessment/`) read system state and report which controls pass or fail; hardening scripts
(`scripts/`) apply remediations when explicitly invoked; drift detection (`drift/`) re-checks
previously-passing controls on a schedule and flags regressions. It is designed to run with no
network calls of its own and no persistent elevated access — see "Security design properties"
below.

## Where untrusted input enters

- **Community PRs/issues on the Sarge repo** — treated as untrusted. Sarge's own operating rules
  require anomalous instructions found in community content to be escalated, never executed.
- **The system state Sarge inspects** — a compromised host could, in principle, try to craft file
  contents or command output intended to manipulate the agent reading it. This is the most
  security-relevant input path and the one we most want findings against.
- Sarge does not accept network input; it has no listening service and no API.

## Which components matter most (in scope, highest to lowest priority)

1. `assessment/` and `assessment/checks/*.sh` — a **false negative** here (reporting a failing
   control as passing) is the single worst outcome: it gives an operator false assurance about
   their security posture. Treat false negatives as **critical**, even without a demonstrated
   downstream exploit — the harm is the false assurance itself, not a separate payload.
2. `scripts/*.sh` (the hardening scripts) — any path by which a hardening script could be
   tricked into granting broader access than intended, escalating privilege beyond what the
   operator authorized, or leaving the system in a worse state than before. Treat as **critical**.
3. `drift/*.sh` — bypass conditions or ways to suppress a drift notification silently. Treat as
   **high**; the operator depends on this to notice regressions between full assessments.
4. `SKILL.md` and agent-prompt-adjacent files — prompt injection pathways that could get the
   agent to run a hardening script, skip a check, or otherwise act without explicit operator
   authorization. Treat as **high to critical** depending on what the injected action would do
   (compare against items 1-2 above).
5. Control-to-NIST-mapping data (catalog files) — an incorrect mapping that gives false assurance
   about compliance coverage. Treat as **medium to high**.

## Severity calibration (explicit, since this isn't a typical web app)

Sarge has no web/API surface, so the usual SQLi/XSS calibration questions don't map directly.
Instead:

- **Critical**: any false negative in `assessment/`; any privilege-escalation path in
  `scripts/`; any way to make Sarge silently skip or misreport a check.
- **High**: drift-detection bypass/suppression; prompt-injection paths that could trigger a
  hardening script without operator confirmation; incorrect control mappings for
  high-impact controls (access control, audit logging, remote access).
- **Medium**: incorrect control mappings for lower-impact controls; non-security-relevant bugs
  in report formatting that could mislead an operator reading output quickly.
- **Low / out of scope for severity escalation**: cosmetic issues, missing controls that are
  explicitly future-roadmap (Sarge is mid-build toward full 800-53 coverage; an unimplemented
  control is a feature gap, not a vulnerability, unless it's advertised as implemented and
  isn't).

## Explicitly out of scope

- OpenClaw itself (the agent runtime Sarge runs under) — report upstream to the OpenClaw project.
- The OS packages Sarge recommends installing (ufw, fail2ban, auditd, etc.) — report to their
  respective upstreams.
- Social engineering of maintainers.
- Theoretical attacks with no practical exploit path and no plausible false-assurance effect.

## Security design properties (so findings against these are especially interesting)

1. No network calls — Sarge never transmits data off the local system during assessment.
2. No persistent elevated access — hardening scripts require explicit sudo invocation; no
   setuid binaries, no background daemons.
3. Read-only by default — assessment scripts do not modify system state; only hardening
   scripts make changes, and only when explicitly run.
4. Checksum verification — scripts ship with SHA-256 checksums intended for pre-execution
   verification.

If a finding violates any of the four properties above, treat it as high or critical regardless
of the component-based guidance earlier in this document — these are the properties we most
want to hold.

## What we want from reports

A reproduction we can run against the built image, and — where it makes sense — a proposed
fix. We're a small, early-stage project (pre-1.0, actively adding NIST 800-53 control
coverage), so please err toward flagging things even if you're not fully certain; we'd rather
triage a false positive than miss a real finding.
