# Contributing

This is an internal repository. Changes go through normal branch +
pull request flow.

## Workflow

```bash
git clone https://github.com/Sanchitkanwar31/Revora.git
cd Revora

cp .env.local.example .env.local   # fill in Supabase + Meta creds
npm install
npm run dev
```

1. Branch off the latest `main`.
2. Make your change.
3. Run `npm run typecheck` and `npm run format` locally.
4. Open a PR, fill in the PR template (especially the **Test plan**).
5. One logical change per PR.

## Reporting bugs / security issues

- Bugs: open a GitHub issue.
- Security issues: follow the private disclosure flow in
  [SECURITY.md](./.github/SECURITY.md) — do not file publicly.

## Dev-loop reference

| Command | What it does |
| --- | --- |
| `npm run dev` | Turbopack dev server on port 3000. |
| `npm run build` | Production build. Next also runs its own typecheck here. |
| `npm run typecheck` | `tsc --noEmit`. Fast TS-only pass. |
| `npm run lint` | ESLint. |
| `npm run format` | Prettier write. |
| `npm run format:check` | Prettier in check-only mode. Useful in CI. |
