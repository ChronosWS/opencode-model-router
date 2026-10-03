# OpenCode v2 compatibility

Issue [#40](https://github.com/marco-jardim/opencode-model-router/issues/40) is a
real API migration. V1 imports a function and awaits a hooks object. V2 validates
the module's default export as an object with an `id` and `setup` or `effect`.
The old function fails with `Expected object at ["default"]` before any routing
code runs. Merely wrapping it in `{ server }` does not satisfy v2.

The adapter targets `@opencode/plugin` **2.0.20** and the official
[`v2` source at 74dbc509](https://github.com/anomalyco/opencode/tree/74dbc509d74df46a2523676dd4068225c4f0c9b0).
The old `@opencode-ai/plugin/v2` types bundled with v1 are a different, obsolete
API; they should not be used to assess current v2 capabilities.

## Loading without breaking v1

The package retains `main: ./src/index.ts` and that module's single callable
default export. Old loaders invoke every exported value, so changing it to an
object or adding runtime helper exports would break those installations.

V2's [host resolver](https://github.com/anomalyco/opencode/blob/74dbc509d74df46a2523676dd4068225c4f0c9b0/packages/plugin/src/host.ts)
tries `/server` first. The published `server.ts` re-exports the v2 definition,
including a `server` factory for modern v1 loaders that explicitly use it. Both
versions share the existing routing engine and configuration.

## What replaces the old hooks

| Existing behavior | V1 surface | V2 implementation |
| --- | --- | --- |
| Register tiers, models, limits and prompts | `config` | `agent.transform` |
| Register `/tiers`, `/preset`, `/budget`, `/router`, `/bypass`, `/annotate-plan` | `config`, `command.execute.before` | `command.transform` and prompt admission |
| Track a tier's dispatch and read budget | `chat.message` | `session.hook("prompt")` |
| Inject the routing protocol; configure grader requests | system transform, `chat.params` | `session.hook("context")` and hidden grader instructions; interim grader temperature policy |
| Block disallowed tool calls; repair task instructions | `tool.execute.before` | `tool.hook("execute.before")` with native argument translation |
| Count reads; inspect returned work; attach verification | `tool.execute.after` | `tool.hook("execute.after")` with structured-result translation |
| Expose `delegate` and `router_verify` | `tool` map | `tool.transform`, retaining argument schemas |
| Create a producer or an independent grader | SDK session create/prompt | Native `subagent.execute`, with actual parent/call identity |
| Track lifecycle and clear state | `event`, `dispose` | Event subscription plus setup cleanup |
| Flag narration without work | `experimental.text.complete` | `session.text.ended` plus a synthetic warning |

The [official migration guide](https://opencode.ai/v2/docs/build/plugins/migrate-v1)
describes these domains. Their callbacks have different data shapes; returning
v1 hooks from `setup` would register nothing.

## Child sessions, cancellation and verification

The v2 public session creation method cannot set `parentID`. Creating a standalone
session would lose the parent relationship and the native permission/depth checks.
Instead, the adapter invokes the host's registered `subagent` implementation with
the real originating tool context. Its awaited creation notification registers
the child's guards and captures the verification reference before the first model
request. A separate hidden grader agent receives the selected grader model and
grading instructions, without static temperature settings. An interim router-owned
`enforcement.verify.graderTemperaturePolicy` filters V2 grader temperature after
the shared `chat.params` hook supplies `graderTemperature`. This policy is pending
upstream temperature-support capability metadata; it is not an OpenCode setting.
Ordinary agent generation options and V1 `graderTemperature` behavior are unchanged.

Resolution uses the exact `models["providerID/modelID"]` entry, then
`providers["providerID"]`, then `default`, then implicit `"auto"`:

- `"allow"` preserves the configured temperature, including `0` and nonzero values.
- `"deny"` removes temperature, leaving model/provider defaults in effect.
- `"auto"` consults an internal exact-model capability table. Its only evidenced
  entry is `openai/gpt-6-luna-fast` (unsupported); unknown models also omit
  temperature conservatively. No provider, prefix, family, or reasoning-status
  inference is used. An explicit `"auto"` at a more specific level does **not**
  inherit a broader `"allow"`.

The bundled policy contains only `{"default":"auto"}`: an exact bundled deny
would duplicate the internal capability evidence. Overrides deep-merge normally.
For a model whose temperature support you have confirmed, use an exact allow:

```json
{
  "enforcement": {
    "verify": {
      "graderTemperature": 0,
      "graderTemperaturePolicy": {
        "default": "auto",
        "models": { "your-provider/compatible-model": "allow" }
      }
    }
  }
}
```

Explicit allow overrides even known unsupported evidence; only use it when the
selected endpoint accepts temperature. It never invents a temperature if the
shared hook supplied none. Policy and provider/model maps must be objects, not
arrays or null, and all policy values must be `"auto"`, `"allow"`, or `"deny"`.

Grader working directories are applied to the real child session. Native results
must name that child and report completion before they can be graded. Deadlines,
cancellation and cleanup interrupt running children. Queued verification retains
bounded copies of actual parent tool contexts rather than inventing session or
message IDs.

When verification uses another working directory, the router must also be enabled
at that location (for example through global plugin configuration). V2 resolves
agents and hooks per location; an unavailable grader fails verification instead
of silently using an unrelated agent.

V2 does not expose session removal on its public plugin context. Completed child
history remains attached to its parent; v1 still aborts and deletes its temporary
sessions. V2's immutable completed-text events also require anti-narration warnings
to appear as separate synthetic entries. These are host behavior differences, not
claims that the old delete or text-mutation hooks still exist.

The adapter awaits native subagent completion, even if a dispatch requested
`background: true`, so the shared acceptance gate never treats an unfinished
background response as completed work. The router's configured deferred and
background verification operate on completed artifacts as before.

If a user backgrounds an already-running native job, its pending response is
explicitly marked unverified. It is never sent to the acceptance gate as if it
were a completed artifact.

## Validation

Focused tests cover the exact module schema failure, preservation of the callable
v1 export, hook/argument/result translation, enforcement errors, independent child
dispatch, model variants, cancellation, queued parent identity and cleanup. The
SDK dependency is for development-time type checking; the v1 runtime entrypoint
does not import the v2 SDK.

The native registration smoke check loads the plugin in OpenCode v2, then reads
plugin, agent and command catalogs without a model call. A separate provider-backed
smoke suite starts `opencode run --standalone` with an isolated home and a local,
deterministic OpenAI-compatible provider. It exercises protocol injection, exact
tier-model selection, native child dispatch, guarded tool-result translation,
awaited background input, and the verified producer/grader path without using user
credentials or the shared OpenCode service. Grader scenarios prove automatic
omission against a provider that rejects temperature and explicit allowance of
both zero and nonzero temperature against a compatible provider.

Run the committed native loader regression with `OPENCODE_V2_BIN` set to the
absolute path of an OpenCode 2.0.20 executable:

```sh
npm run smoke:v2
```

It starts an isolated local server, demonstrates rejection of the old function,
then checks the fixed package's catalogs. It requires no provider credentials.

Run the provider-backed standalone suite with the same executable:

```sh
npm run smoke:v2:e2e
```

Set `OPENCODE_TEST_TMPDIR` to select the parent for test-owned temporary homes.
Every spawned OpenCode process uses `--standalone`; the suite removes provider
credentials and OpenCode settings inherited from its parent environment.
