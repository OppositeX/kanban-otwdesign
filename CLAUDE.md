# CLAUDE.md — kanban-otwdesign

This repo is the kanban board Omri uses to track all his projects
(`index.html`, deployed to Vercel at `kanban-otwdesign.vercel.app`). Agents
working on other projects sync their task state into it — see `AGENTS.md`
for that protocol; it's unrelated to how *this* repo itself gets developed.

## Standing instruction: always merge to `main`

Omri's persistent link only serves what's on `main` — Vercel deploys `main`
to production. He has explicitly authorized always merging finished work on
this repo into `main` and pushing, without asking for confirmation each
time. Do this as the normal last step of any change here, the same way you'd
commit and push to a feature branch — no need to check in first.

Practical notes:
- If `main` and your working branch have diverged (not a fast-forward),
  do a real merge (`git merge --no-edit`), don't force-push or rewrite
  history.
- If a merge produces actual textual conflicts, resolve them by
  understanding both sides first — don't blindly prefer one side. If the
  right resolution is genuinely ambiguous (e.g. discarding another
  session's unrelated work entirely), that's still worth a quick check
  with Omri before pushing; the standing authorization covers routine
  merges, not "throw away someone else's implementation" calls.
- Verify basics before pushing to production: syntax-check the extracted
  `<script>`, and re-run whatever the last verification pass covered
  (overflow sweep, etc.) if the change touches layout/CSS.
