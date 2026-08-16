<!-- title: Context budget — measure it before you optimize it -->
# Context budget

Every session starts with a fixed cost, paid before the agent does anything. This page says what is
in it, how to measure it yourself, and — the part that surprises people — **where the weight actually
is**, which is almost certainly not where you would trim first.

All numbers below are measured on the reference deployment (Claude Code 2.1.224), not estimated.

## What a session actually loads

| Layer | root agent | cos agent | dev agent |
|---|---:|---:|---:|
| Harness base (system prompt + built-in tool schemas) | 22,618 | 17,701 | 22,624 |
| **MCP tool listings** | **8,824** | 4,528 | ~1,400 |
| Skill + slash-command listings | 2,544 | 2,527 | 2,447 |
| `CLAUDE.md` + auto-memory index | 4,188 | 6,041 | 4,041 |
| **Total** | **38,174** | **30,797** | **30,486** |

Read that table before rewriting a single line of your `CLAUDE.md`.

**Your own content is ~11–20% of the budget.** The contract, the memory index, the skills — everything
you author and agonize over — is the smallest slice. Trimming prose is optimizing the wrong layer. On
this deployment, an aggressive edit of every agent contract would have recovered maybe 1,000 tokens
out of 38,000, while two configuration changes recovered 15,000.

## The one that surprises everyone: MCP tool *names*

**Tool schemas are deferred** — the model fetches them with `ToolSearch` when it needs them, which is
why a 546-tool server does not cost hundreds of thousands of tokens. **Tool names are not deferred.**
Every configured server's full tool list sits in the always-on context at roughly **16 tokens per
tool**, paid on every session, whether or not that server is touched.

On the reference deployment one server — an infrastructure-management MCP with ~546 tools — cost
**8,824 tokens, 23% of the root agent's entire session**, on every single session including the ones
that never went near it.

Corollaries worth knowing:

- **A single built-in tool can outweigh your whole contract.** The `Workflow` tool measured ~4,900
  tokens on its own — larger than that agent's `CLAUDE.md` and memory index combined.
- **A *failed* server costs no context but is not free.** It contributes zero tools, but still attempts
  a connection at every session start and shows up as health-check noise. Fix it or park it.
- **Two servers exposing the same tools is a selection problem, not a token one.** One agent here ran
  an Atlassian MCP whose 31 tools were a *strict subset* of a connector's 40 — 31 pairs of
  identically-named tools competing for selection. That is the shadowing hazard
  [`skills.md`](skills.md) warns about for skills, and it applies to tools with more force.

## Park what you do not use continuously

The fix is not to delete a useful server; it is to stop paying for it when you are not using it.

```
mcp-park status                    # attached / parked
mcp-park on  <server> --restart <topic>
mcp-park off <server> --restart <topic>
```

A reference implementation ships at
[`../templates/root-agent-skills/manage-agents/scripts/mcp-park`](../templates/root-agent-skills/manage-agents/scripts/mcp-park).
It moves a server definition between the live config and a parked file — both `0600`, both outside
git, the credentials it carries never printed — atomically, and verified byte-identical across a round
trip.

Two things to internalize:

- **A session reads MCP config at startup.** Nothing changes until the topic restarts. A restart
  resumes the same conversation, so no context is lost — but **verify reachability afterwards**, per
  [`runtime.md`](runtime.md).
- **If you need one fact, do not attach at all.** A one-shot
  `claude -p --mcp-config <parked.json> --strict-mcp-config '<question>'` answers it without touching
  the running session.

## Measuring it yourself

`claude doctor` will not help: it reports installation health only — version, update status, install
issues — and says **nothing** about context. Useful, but not for this.

Real numbers come from a one-shot with token accounting, isolating layers by subtraction:

```bash
m(){ claude -p --output-format stream-json --verbose "$@" "say ok" \
     | python3 -c 'import sys,json
for l in sys.stdin:
    try: d=json.loads(l)
    except: continue
    if d.get("type")=="result":
        u=d.get("usage",{})
        print(u.get("input_tokens",0)+u.get("cache_creation_input_tokens",0)+u.get("cache_read_input_tokens",0))'; }

m                                                        # everything
m --mcp-config '{"mcpServers":{}}' --strict-mcp-config    # minus MCP  → the MCP layer is the difference
m --mcp-config '{"mcpServers":{}}' --strict-mcp-config --disable-slash-commands   # minus skills too
```

Three traps that will give you wrong numbers:

1. **Measure from the agent's real working directory.** MCP servers can be user-scoped or
   project-scoped; running from `/tmp` silently omits the project-scoped ones.
2. **Take the "before" before you change anything.** An obvious point that is easy to get wrong when
   the edit and the measurement are minutes apart — a full prompt-cache hit will happily report a
   delta of exactly zero and look like proof.
3. **Model choice barely matters.** A cheap model measures the same context as an expensive one
   (verified within 1.5% here), so measure with the cheap one.

## Where MCP configuration actually lives

Worth stating plainly, because the obvious assumption is wrong. In a deployment where the supervised
session runs with `WorkingDirectory=%h` — cwd is the agent's **home**, not its brain repo — a
`.mcp.json` sitting in the brain repo is **out of project scope and never read**. The live
configuration is the runtime's own user-scoped config file.

The reference deployment carried a decorative `.mcp.json` in every brain for weeks, and it had drifted
from reality in all three: one listed a server that was parked, another was missing two that were
running. Nothing noticed, because nothing read it.

See [`portability.md`](portability.md): MCP configuration is on the **not carried** list. If you want
it portable, the fix is to pass it explicitly (`--mcp-config`) from the supervising unit and keep the
secrets in an env file — but do that deliberately, and then check it, rather than assuming a file in
a repo is doing something.
