# Repository Guidance

## Purpose

This repository distributes Agent Skills for Microsoft Fabric development and administration. Keep each skill usable from OpenAI Codex and Claude Code unless a requirement targets one client.

Treat each `SKILL.md` as the shared source of truth. Keep client metadata beside it or at the repository root instead of duplicating instructions.

Keep repository guidance domain-neutral. Put workflow-specific rules, APIs, commands, naming policies, and output schemas in the skill that owns them. Do not carry assumptions from one skill into another.

## Repository layout

- Put each skill in `skills/<skill-name>/SKILL.md`.
- Put OpenAI UI metadata in `skills/<skill-name>/agents/openai.yaml`.
- Keep Claude plugin metadata in `.claude-plugin/plugin.json`.
- Let Claude discover the root `skills/` directory. Add a custom `skills` path to the manifest only when the layout changes.
- Do not add a README, changelog, installation guide, or quick-reference file inside a skill package. Add a resource only when the skill needs it at runtime.

## Skill authoring

- Use lowercase kebab-case for the folder and frontmatter `name`. Keep them identical.
- Limit shared `SKILL.md` frontmatter to `name` and `description`. Put what the skill does and when to invoke it in `description`.
- Write instructions in imperative form. Keep the main file below 500 lines when possible.
- Give fragile operations exact commands and fail-closed checks. Give judgment-based work concise decision rules.
- State prerequisites, supported platforms, required shells, permissions, and preview status before the workflow.
- Use official product documentation when an API, CLI, schema, or platform contract may have changed.
- Keep examples executable and internally consistent. Do not present pseudocode as a ready-to-run command.
- Preserve user-provided text and identifiers unless the target contract requires a documented transformation.
- Add scripts, references, or assets only when they remove repeated work or supply information the agent cannot infer.

## Operational safety

- Identify the workflow's preconditions, external-write boundaries, failure modes, and recovery properties before changing external state.
- Validate relevant command exit codes, responses, required fields, and paginated results. Stop when a required preflight check fails.
- Treat non-idempotent operations as non-retryable until a read-only check proves that another attempt is safe.
- Require user approval for destructive or material external actions unless the current request grants that authority.
- Do not request or store passwords, tokens, connection strings, tenant secrets, or other credentials. Use interactive authentication in the user's terminal.
- Persist enough provenance to resume multi-step work safely. Record source identifiers, requested targets, results, warnings, and remediation status when the workflow spans external writes.
- Record assumptions and observed preview behavior. Do not present tenant observations as documented platform guarantees.

## Client metadata

### OpenAI Codex

- Generate `agents/openai.yaml` with the installed `skill-creator` tooling.
- Include `interface.display_name`, a 25 to 64 character `interface.short_description`, and an `interface.default_prompt` that names the skill as `$<skill-name>`.
- Do not add icons, brand colors, dependencies, or invocation policy without a current requirement and the required assets.

### Claude Code

- Keep the repository manifest valid under `claude plugin validate . --strict`.
- Use the repository name as the plugin namespace unless a rename has been approved.
- Do not add `allowed-tools`, model overrides, invocation restrictions, hooks, or custom component paths without a concrete requirement. Permission grants need security review.
- Review the manifest description and keywords whenever a skill is added, removed, or changes the plugin's scope. Keep discovery metadata representative of the full plugin.
- Bump `.claude-plugin/plugin.json` using semantic versioning when a committed change alters a plugin-visible skill or Claude metadata. Repository-only guidance and development tooling do not require a plugin version bump.

## Validation

Run checks that match the change before committing:

1. Run Python-based repository tooling through `uv`. Validate every changed skill with `uv run --frozen skills-ref validate skills/<skill-name>`.
2. Run the validator from the installed OpenAI `skill-creator` package when it is available, because client-specific checks can be stricter than the shared Agent Skills specification.
3. Run `claude plugin validate . --strict` for Claude metadata or plugin-layout changes.
4. Parse or lint changed executable examples with the appropriate runtime or language tooling. Run scripts when the change affects executable behavior and a safe local test is possible.
5. Run `git diff --check` and inspect the full staged diff.
6. Forward-test complex skill changes with a realistic prompt that does not reveal the expected answer.
7. Run an independent review after non-trivial changes. Address confirmed findings and repeat the same review until no material finding remains.

Treat `pyproject.toml` and `uv.lock` as development tooling. Keep the lockfile committed. Change validator dependencies and refresh the lockfile only as an intentional tooling update.

Do not claim a check passed unless it ran successfully. Report environment failures separately from product failures.

## Git workflow

- Check the branch, worktree status, remote state, and recent commits before editing.
- Preserve user changes. Stage only files changed for the current request.
- Do not work on `main` without user permission.
- Follow the existing commit style. Include a focused title and a body that explains the behavior or packaging change.
- Do not push, open a pull request, merge, publish, or create a release unless the user asks for that action.

## Handoff

Report changed files, validation results, branch and commit state, plugin version impact, unresolved risks, and required user configuration. State whether any hardcoded policy, fallback, compatibility path, or technical debt was introduced.
