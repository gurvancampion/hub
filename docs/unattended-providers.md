# Unattended agent providers

Hub accepts Claude, Codex, OpenCode, and Cursor for unattended trigger agents. The selected
daemon still validates each provider's model, mode, thinking option, and provider options.

Every launch includes exact MCP preapprovals for `hub.finish_execution` and the materialized
output tools. These grants do not add outputs or bypass the Hub MCP server's execution limits.
Claude, Codex, and OpenCode translate these grants through their native permission contracts.
Their other permission settings come from the authored mode and provider options; Hub does not
answer their permission requests.

| Provider | Daemon preapproval handling                                                                                                            |
| -------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Claude   | Adds exact `mcp__hub__<tool>` entries to `allowedTools`.                                                                               |
| Codex    | Sets the server's `enabled_tools` to the grants, gives each `approval_mode: approve`, and keeps the default approval mode at `prompt`. |
| OpenCode | Adds allow rules for exact `hub_<tool>` permission names alongside authored rules.                                                     |
| Cursor   | Uses ACP `auto_accept` for tool approvals and requires the daemon contract described below.                                            |

## Cursor

Selecting Cursor for a Hub automation enables the daemon's ACP `auto_accept` feature. This
approves tool permission requests for the unattended agent. It grants approval to all tools the
provider asks to run, including tools outside the injected Hub MCP server. The authored mode,
model, thinking option, and provider options pass through unchanged:

```yaml
agent:
  provider: cursor
  model: grok-4.7
  mode: agent
  thinkingOptionId: medium
```

Hub sends `featureValues: { auto_accept: true }` at creation and reapplies it through
`agent.config.apply.request` before observing and delivering each turn. This also covers
continuation, workspace restoration, and recovery after Hub or daemon reconnection. If the daemon
rejects the feature update, Hub does not deliver the prompt. No additional daemon permission is
needed; the config RPC accepts `hub.execute`.

[Cursor's ACP documentation](https://cursor.com/docs/cli/acp) describes permission requests and
the allow decisions that unblock tool execution.

### Required daemon support

Cursor requires a daemon with a Cursor-specific MCP preapproval contract. A daemon whose
`PROVIDER_CONTRACTS` includes only Claude, Codex, and OpenCode will still reject Cursor launches
with `cannot preapprove exact MCP tools for unattended execution`. Hub preserves the tool policy
and propagates this refusal. Enabling ACP auto-accept does not remove that registry check.

The daemon must validate and preserve exact tool grants for the injected `hub` MCP server and
enable Cursor's unattended permission handling across creation and resume. Merely declaring
`supportsExactMcpPreapproval: true` does not implement exact grant matching in ACP.

The existing shared ACP auto-accept handler approves requests with an allow option but leaves
chooser requests and requests without an allow option pending. For fully unattended Cursor runs,
the daemon must resolve those cases without waiting for a human, or fail the turn immediately
when approval is impossible. Re-enabling auto-accept must also resolve requests already pending
on a resumed session. Any blocking Cursor plan or question extension requests also need
an unattended response. Hub's session cannot send `agent_permission_response`, which requires
`workspace.write`; the daemon must own this behavior rather than expanding Hub's authority.

Cursor support is therefore conditional on that daemon follow-up. Validate a real run through
`hub.finish_execution` and check that no permission requests remain pending before relying on it.
