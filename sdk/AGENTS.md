# RULES

- NEVER run `git push`, `git branch`, `git add`, `git commit`, `git restore` or `git checkout` commands (strictly forbidden)

# RULES for code change and development do:
- ALWAYS start by reading `docs/.digest.md` and `docs/.graph.json`
- ALWAYS run `npm install` to check dependencies
- ALWAYS run `npm run lint` to check code syntax
- ALWAYS run `npm run build` before `npm run typecheck`
- ALWAYS run `npm run typecheck` before `npm run test`
- ALWAYS verify if `OpenApiSpecGenerator.ts` is updated after changes in `src/server` with endpoint changes.