# unreal-mcp-crowd-gotchas

Notes from building a crowd cinematic in UE 5.8 — MetaHuman Crowd + an AI
agent (Claude Code) driving the editor over Unreal's MCP plugin. Three things
nearly killed the workflow, none documented anywhere obvious. Written up here
in case it saves someone else the afternoon it cost me.

This repo is free, standalone reference — a config snippet plus the write-up
below. No project files, no product, just the notes.

---

## Trap 1: your MCP server "connects" but exposes almost nothing

If you've enabled Unreal's MCP plugin and pointed an MCP client at it, you've
probably seen the connection succeed and figured you were done. Here's the
tell that you're not:

Call `list_toolsets` (or the equivalent in your client). If it comes back
successful but with **one entry** —

```
ToolsetRegistry.AgentSkillToolset: Provides tools for listing, reading, and
creating/updating skills.
```

— and drilling into it shows just four tools, all for managing `AgentSkill`
assets (not actors, not blueprints, not the level) — that's not a broken
connection. The handshake is fine. You just have nothing registered to serve.

**Root cause:** the base MCP plugin only ships the one bundled toolset. Every
other capability — actors, blueprints, materials, Sequencer, Niagara, PCG,
gameplay tags, the lot — lives in separate Toolset plugins that are **not
enabled by default**.

**The fix:**
1. `Edit → Plugins`, search **"All Toolsets"**.
2. It's filed under **Experimental** — if it's not showing up, toggle "Show
   Experimental Plugins" in the browser first.
3. Enable it, restart the editor when prompted, reconnect your MCP client.

For me this took toolset count from 1 to 52 — actor/asset/blueprint/scene
tools, Sequencer (including keyframing, control rig, camera-cut sections),
Niagara, PCG, StateTree inspection, and more, all after that one plugin
toggle.

**Two things worth knowing before you rely on it:**

- **"All Toolsets" doesn't actually mean all toolsets.** Its manifest lists
  21 dependencies, but the `Toolsets/` folder in the engine ships 27. Notably
  absent from the aggregator: `MetaHumanGenerator`, `ChaosClothAssetToolset`,
  `LiveCodingToolset`, `MVVMToolset`, `SequencerAnimMixerToolset`. If you're
  doing MetaHuman generation work specifically, you need that one enabled
  separately.
- **It's marked `EditorOnly`, `NoRedist`, `IsExperimentalVersion: true`.**
  Treat it as a dev-time aggregator, not something to lean on in anything
  shipped.

Separately — and this is a one-time step, not part of the fix above — you'll
still need to run `ModelContextProtocol.GenerateClientConfig ClaudeCode` from
the editor console once, to actually generate the client config file in the
first place. AllToolsets is what makes that config worth having; without it
you get a config file pointing at almost nothing. See
[`mcp-config/example.mcp.json`](mcp-config/example.mcp.json) for the shape it
should end up with.

---

## Trap 2: crowd spawning fails with zero agents, zero errors — and a decoy actor sends you the wrong way

This one's specific to MetaHuman Crowd (the Mass-framework-based crowd system
— not City Sample Crowds, which is a separate, older system).

Symptom: you've migrated the StarterKit content, wired up a
`MetaHumanMassSpawner`, hit Play — and nothing spawns. Not "a few" agents.
Zero. No error, no warning, nothing in the log. Raising the spawner's `count`
from 12 to 30 changes nothing.

First instinct is to check for a navmesh. That's when it gets actively
misleading: querying the level for anything with "Nav" in the name *returns
a hit* — `AbstractNavData-Default`. Looks promising. It's a decoy: that's
Unreal's internal null/fallback nav data, not a real navmesh, and a naive
"is there a Nav actor here?" check will tell you everything's fine when it
isn't.

**Root cause:** MetaHuman Crowd spawns through an EQS query
(`MassEntityEQSSpawnPointsGenerator`) that projects candidate spawn points
onto an actual navmesh. No real navmesh → the query returns zero points →
zero entities spawn, with nothing surfaced anywhere to tell you that's what
happened. Count was never the variable.

**The fix — and the part that's genuinely undocumented:**
1. Add a `NavMeshBoundsVolume` covering your crowd area.
2. This auto-creates a real `RecastNavMesh-Default` actor.
3. Its `RuntimeGeneration` defaults to **`Static`** — meaning it only builds
   from a saved editor bake. Leave it there and the crowd can *still* spawn
   zero agents in Play mode or in a render, even with a real navmesh now
   sitting in the level, because nothing ever triggered a build. Flip it to
   **`Dynamic`** so it builds at `BeginPlay`, and it works in both PIE and
   Movie Render Queue without depending on that bake.

**Other things that will bite you once agents are actually spawning:**

- **Keep the nav volume's Z extent thin and floor-only.** A volume that
  overhangs a drop or a gap bridges it in the navmesh, and agents either sink
  into nothing or the mesh fragments into sub-regions agents get stuck
  circling inside.
- **"Sliding without animation" is an LOD swap, not an anim bug.** The crowd
  visualization trait swaps representation (high-res actor → low-res actor →
  skinned-mesh-instance → **none**) by camera distance. For a small crowd
  where you want everyone animated the whole shot, push those distance
  thresholds out.
- **Walk speed isn't in the steering trait.** It's easy to go looking in
  whatever trait sounds speed-related (steering reaction time, spring
  smoothing) and find nothing. Base speed lives in the movement trait's
  `defaultDesiredSpeed` / `maxSpeed`.
- **Movie Render Queue is disabled by default too**, separately from the MCP
  toolset gap above — if it's missing from your Window menu, check
  `MovieRenderPipeline` is enabled in Plugins (restart required). And budget
  warmup frames in the render job: agents spawn at `BeginPlay` and need time
  to disperse before the "real" footage starts, or your opening frames show
  everyone still clumped at the spawn point.

---

## Trap 3: the wander radius is compiled into a behavior asset, with no API to touch it

Once agents are actually walking, the next question is how far they roam —
and that number isn't a spawner property, isn't on the movement trait, and
won't show up in a normal property dump.

It's a single field, **Search Radius**, buried inside a task node inside a
StateTree behavior asset — the crowd's wander AI graph. And the tool API for
StateTree assets is **read-only**: it exposes `get_root_states`,
`get_children`, `get_tasks`, and similar — no `set_*` equivalent anywhere.
You can inspect the value programmatically. You cannot change it that way.

The only way to adjust it: open the StateTree asset in its own graph editor,
select the specific child state holding the "Find Reachable Point" task (not
its sibling "Stand" state), click into that task, and edit **Search Radius**
directly in the Details panel. If your crowd is only ever taking tiny hops
and staying clustered near spawn, this is almost certainly why — and it's
worth knowing before you go looking for a config value to bump, because there
isn't one to find through the tooling.

---

## Why I'm writing this up

None of this is exotic — it's the kind of thing that's obvious once you know
it and completely opaque before. I'm putting together a proper kit around
this workflow (working crowd setup + camera rig templates + render presets +
an agent skill layer, so you can describe a shot and have it built rather
than hand-wiring each piece) — genuinely curious whether this is a problem
other people building crowd cinematics in 5.8 have hit too, or whether I'm
niche-of-one here. Open an issue or reach out if any of this helps, or if
you've hit the same walls.
