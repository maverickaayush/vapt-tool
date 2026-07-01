# Workflow Rules — What NOT to Do

These rules exist to keep Claude Code sessions in this repo token-efficient.
They govern *process*, not the VAPT tool's behavior — see `CLAUDE.md` for
the actual project rules and architecture.

- **Do not re-explain the architecture unless asked.** If the six-layer
  pipeline, the scan lifecycle, or the schema contract comes up again, point
  to `docs/QUICK_REF.md` or the relevant `CLAUDE.md` section instead of
  restating it.
- **Do not modify unrelated modules.** A change to `headers.py` should not
  touch `owasp.py`, `ssl_tls.py`, etc. unless the task explicitly spans them.
- **Do not rewrite full files unnecessarily.** Use targeted edits (patch the
  specific function/block) instead of regenerating an entire file when only
  part of it needs to change.
- **Prefer minimal diffs.** Smaller, reviewable changes over sweeping ones —
  especially in `tasks/`, `analysis/`, and `reports/`, where the schema
  contract (Section 4.3 of CLAUDE.md) must stay intact.
- **Avoid loading the entire repo for small changes.** Read only the files
  actually relevant to the task at hand; don't re-read `CLAUDE.md` in full
  for changes scoped to one module.
- **Don't reload context that's already established in the conversation.**
  If a file was already read this session and hasn't changed, don't re-read
  it "just to be sure."
