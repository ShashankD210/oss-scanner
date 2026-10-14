# Clone threat model

Clone is a Perl XS module that recursively copies Perl values. The relevant trust boundary is a caller passing attacker-influenced Perl data structures into `clone()`: nested references, cycles, tied or magical values, blessed objects, copy-on-write scalars, and unusually deep or large structures. Ordinary use already permits user-defined tie methods and object behavior; distinguish execution caused by those explicit callbacks from unintended execution by Clone itself.

Prioritize memory safety in the XS implementation, reference count errors, use-after-free, double-free, crashes, and resource exhaustion. Exercise depth limits, aliasing, cyclic structures, weak references, threads, and malformed or unusual scalar states. The regression suite is under `t/`; after building, run `make test` from `/src` and use `prove -b t/<file>.t` for targeted cases.

Treat a demonstrated memory corruption path from attacker-influenced values as high severity, or critical if realistic code execution is shown. A reproducible process crash or uncontrolled memory/CPU use is generally medium, adjusted for reachability and impact. A change in copy semantics, shared references, or unexpected callback invocation can matter when it crosses a security boundary; explain the concrete downstream exploit scenario.

Keep reports private per `SECURITY.md`; do not open public issues or PRs for vulnerabilities. Include a minimal Perl reproducer, Perl version and build details, expected versus actual behavior, and a patch with regression coverage. AI findings and proposed patches require human maintainer verification under `AI_POLICY.md`. Ignore issues solely in Perl, dependencies, or downstream consumers unless Clone itself enables the vulnerability.
