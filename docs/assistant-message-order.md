# Assistant progress messages in Codex

The WorkBuddy Responses-to-Chat gateway assigns assistant message phases when
the Chat upstream cannot supply them. Text accompanying a tool call is
`commentary`; a complete `stop` response without tools is `final_answer`.
Truncated responses remain commentary. Reasoning stays a separate reasoning
item and is not reclassified as a user-visible progress message.

For JSON responses, a tool preamble precedes the corresponding tool items.
Streaming Responses messages begin as commentary and receive their definitive
phase on `response.output_item.done`. The same item ID is retained. Terminal
response arrays follow `output_index` even when tool calls were buffered.
Reopened text/reasoning items get fresh buffers, preventing repeated content.

This aligns the adapter with Codex's existing `AgentMessageItem.phase` contract.
It changes newly generated messages. Existing persisted histories are not edited.
The Android app is separately owned; verify the visual result on new messages
after deployment. No Codex binary, Router, or Sub2API change is required.

Relevant checks:

```sh
python -m unittest server.tests.test_response_message_order \
  server.tests.test_tool_discovery server.tests.test_responses_api \
  server.tests.test_reasoning_credential_roundtrip
```

The acceptance fixture also runs the actual Codex app-server with disposable
read-only tools and checks its completed-item timeline: commentary, tool,
commentary, tool, final answer. Keep this separate from Android visual testing.

## Locally configured Codex model list

The affected installation uses `model_catalog_json` in `~/.codex/config.toml`,
pointing to `~/.codex/model_catalog_compat.json`. This catalog contains a `models`
array. Common editable fields include:

- `slug`: exact model name sent in API requests; must exist in the upstream route.
- `display_name`: model picker label.
- `description`, `priority`, `visibility`: help text, ordering, and visibility
  (`list` displays a model; `hide` hides it in the catalog).
- `context_window`: the model/provider's actual supported context limit.
- `supported_reasoning_levels`, `default_reasoning_level`: supported effort choices.

Preserve the remaining capability fields. For a new compatible model, copy an
existing third-party entry and adjust its identity and verified limits. Enabling
`supports_search_tool` requires protocol support or a validated adapter.

Codex 0.159.2 takes a model-catalog snapshot at app-server startup. Changes require
an idle Remote restart and reopening the model picker; CLI processes must also
reload their configuration. The existing Control website manages CCS providers
and routing; it currently has no model-catalog editor. CCS provider selection
and the Codex model catalog are separate settings.
