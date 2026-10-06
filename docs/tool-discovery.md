# Third-party Codex tool discovery

The Responses gateway supports **client-executed** tool search for Chat
Completions upstreams. Declare `type: tool_search`, `execution: client`; the
gateway converts it into a collision-safe function and restores the native
Responses item on JSON and streaming responses. Hosted/server search is rejected.

Deferred tool definitions are withheld until a client search result loads them.
Schemas from `tool_search_output` and `additional_tools` are promoted even when
they still carry `defer_loading: true`. Repeated definitions are deduplicated;
namespace identities and tool-call/result pairs survive history replay.

The client model catalog must advertise `supports_search_tool: true`, and its
provider must support namespace tools. Codex 0.159.2 snapshots its model catalog
at app-server startup: changing the catalog requires an idle restart. Do not
interrupt active turns or synthesize new user turns to activate this change.
The old `features.tool_search` setting is not the capability switch in this
version.

Streaming requests wait for the upstream's trailing usage chunk before emitting
their terminal Responses event. This preserves input/cache usage and therefore
the client's context-compaction accounting. Later content after a finish reason
is ignored. Truncated or malformed tool-search arguments are never executable.

Validated on October 6, 2026 with Codex 0.159.2 and real WorkBuddy DeepSeek
v4.1 Flash, GLM 5.3, GLM 5.2, and Kimi K3-1. Disposable MCP fixtures exposed 420
tools and executed steps 1, 2, and 3 in order. Usage injection forced three
actual Codex compaction lifecycles; fixture checkpoint summaries preserved the
completed steps. This is a continuation/protocol test, not a measurement of a
production thread's compression ratio. Tool catalogs sent upstream were about
10–25 KB, compared with about 712 KB observed in the affected production thread.
The first use of an unloaded tool adds a discovery round trip.

Run relevant regression checks:

```sh
python -m unittest server.tests.test_tool_discovery \
  server.tests.test_responses_api server.tests.test_reasoning_credential_roundtrip
```

Existing histories do not need deletion or a task restart. After capabilities
are refreshed, previously unloaded tools can be discovered again when needed.
Long-running production behavior still requires observation; this change does
not guarantee a particular compression percentage or remove upstream rate limits.
