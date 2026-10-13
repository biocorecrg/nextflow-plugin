# AGENTS.md — nextflow-plugin

Plugin repo (Claude + Codex from one layout, plus an OpenCode 1.x and 2.x npm package `opencode-nextflow` from the same tree), not a pipeline repo. 28 skills in `skills/*/SKILL.md`; single MCP server `seqera` → `https://mcp.seqera.io/mcp` (see `.mcp.json`). `package.json` exists only for the OpenCode plugin (`opencode/`); there is no other build system.

## Validate (mirrors CI in `.github/workflows/validate.yml`)

```bash
python3 -I tools/test_consolidate_skills.py
python3 -I tools/check_sources.py
claude plugin validate --strict .
claude plugin validate --strict .claude-plugin/plugin.json
bun install --frozen-lockfile && bunx tsc --noEmit && bun test && npm pack --dry-run
```

Always use `python3 -I`. CI pins Python 3.12, Node 20, `claude-code 2.1.295`, `codex 0.160.0`, `bun 1.3.13`, `opencode-ai 1.19.0`. Skill count check: `ls skills | wc -l` must match `claude plugin details nextflow@nextflow-plugin` (`Skills (N)`) and `MCP servers (1) seqera`.

## Provenance and versions — fail-closed

- `tools/check_sources.py` verifies every `sources.json` `bundled_sha256` plus: `skills/*/SKILL.md` exists, and `.claude-plugin/plugin.json` version == `.claude-plugin/marketplace.json` entry (currently `0.2.2`). Bump both together.
- Never hand-patch hashes or versions to make checks pass.

## OpenCode package (`opencode-nextflow`)

Files in `opencode/` implement dual 1.x and 2.x server plugin support.
- In 1.x: `server` config hook registers every `skills/<name>/` dir via `config.skills.paths` and sets `config.mcp.seqera`.
- In 2.x: `setup` registers `seqera` via `ctx.mcp.transform` and all 28 bundled skills via `ctx.skill.transform`.
- Both versions support options: `{ skills: false | { <name>: false }, mcp: false }`.

- Tests live in `opencode/config.test.ts`; published tarball excludes tests (`files` in `package.json` covers `opencode/index.ts`, `opencode/config.ts`, `skills/`, `scripts/`, `licenses/`, `NOTICE`).
- `tools/import_package.py` never touches `package.json`, `opencode/`, `tsconfig.json` — they survive re-imports. But keep `package.json` version in sync with the plugin version on release.
- Smoke test without publishing: point a scratch `opencode.json` at `"plugin": ["file:<repo>"]`, then `opencode debug config` (check `mcp.seqera` + 28 repo skill paths) and `opencode debug skill`.

## How skills get here (import → curate → consolidate)

Upstream package → `tools/import_package.py <dir|zip>` → `tools/consolidate_skills.py` (plan: `tools/skill_consolidations.json`). It regenerates `skills/`, `scripts/`, `assets/`, `licenses/`, `sources.json`, both manifests, `.mcp.json`, `evals/review-cases.json`.

- Lasting skill-text fixes belong in `import_package.py` `EDITS` / `RENAMES` / `EXCLUDED_SKILLS`, or as `edits` / `post_edits` in `skill_consolidations.json` — not as one-off edits to already-consolidated output (replay must stay a no-op; edits match exactly once or fail).
- Folded skills live at `skills/<owner>/references/<folded>/README.md` (old `SKILL.md` renamed, frontmatter stripped). Reference them via relative link `[label](references/<folded>/README.md)`; never invoke removed names (e.g. `seqera-mcp`, `create-workflow`, `nf-debug`) as skills — `seqera-mcp` is now the connection reference under `launch-workflow`.
- Product name is `Nextflow Platform`, never `Seqera Platform` (enforced by `RENAMES`).

## Skill conventions

- `SKILL.md` starts with closed `---` YAML frontmatter (`name`, `description`); keep `description` trigger phrases accurate (e.g. `build-nextflow-pipeline` owns planning, `nf-pipeline-design` owns code rules).
- Seqera ops: discover with `search_seqera_api`, execute with `call_seqera_api` using returned schemas; host owns OAuth. Never paste tokens/keys, never bypass with curl. See `tools/curation/mcp-connection.md`.
- `evals/review-cases.json`: negative cases (treatment advice, secret disclosure, purchases) must trigger no MCP calls.
- `scripts/nextflow-typecheck.sh [WORKSPACE]` needs `jq`, Java 17+, network on first run (downloads LSP jar); exit 1 on errors.
