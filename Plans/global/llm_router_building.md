# LLM Provider Routing Refactor — Reusable Plan

## Use case
The project has a problem where switching LLM providers requires touching
multiple places in the code. The goal is to collapse that switch down to
**changing a single model name string**.

## Core design principles (agent-agnostic and project-agnostic — apply as-is)

1. **Route by keyword prefix, not exact model name mapping**
   The routing table stores an identifying keyword (e.g. `deepseek`), not
   every specific model name (`deepseek-v4-flash`, `deepseek-v4-pro`).
   Adding a new model version under an existing provider shouldn't require
   touching the routing config at all.

2. **Longest-prefix-wins matching**
   If keywords ever overlap (e.g. `deepseek-v4-flash` and
   `deepseek-v4-flash-lite` both exist as identifying keywords), the
   matching logic must prefer the longer/more specific keyword, to avoid
   routing to the wrong provider. Explicitly test this edge case during
   implementation.

3. **Routing table separated from secrets**
   The routing table (keyword → base_url + the *name* of an env var holding
   the key) is a static, human-maintained mapping. The actual key value is
   always read from `.env` (or the project's existing secret management),
   never stored as plaintext in the routing table itself.

4. **Manually maintained, no auto-discovery**
   No need for a scanning/auto-registration mechanism. Adding a new
   provider is just: add one line to the mapping + add one key to `.env`.
   Keeps it simple and auditable.

## Execution steps (for the executing agent, in plan mode, adapted to the actual project)

1. **Survey the project structure** to check whether a `config/`,
   `maintain/`, `core/`, or similar convention already exists for this
   kind of routing/config file.
2. **Decide where the routing config file should live**:
   - If a convention already exists → follow it
   - If there's no clear convention → the agent picks a reasonable default
     location, and **must report the final chosen path** rather than
     deciding silently
3. **Design the routing table's data structure**:
   `keyword → {base_url, key_env_name}`
4. **Implement the lookup logic**: longest-prefix-wins matching; on no
   match, fail clearly rather than silently falling back to the wrong
   provider
5. **Replace the existing hardcoded provider-selection logic** with calls
   into this routing lookup
6. **Update the project README**, covering:
   - where the routing config file lives
   - how to add a new provider mapping
   - how to switch the active model (which single string to change)

## Acceptance criteria

- [ ] Switching LLM providers requires changing only one model name string,
      no other code changes
- [ ] Adding a new provider requires only: one line in the routing table +
      one key in `.env`
- [ ] Overlapping-prefix scenarios are tested, and the longer/more specific
      keyword wins
- [ ] README clearly documents the above so a new teammate or a new agent
      can follow it

## How to use this plan
Hand the whole plan to any coding agent's plan mode. Let it first survey
the current project structure and come back with concrete paths and
implementation details for confirmation, then proceed to execution.

## Requirements 
the user must specify the module/the working root path. and this plan will only work on the given directory then.
