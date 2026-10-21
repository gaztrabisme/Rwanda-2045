# Agent tooling research: worldbuilding tools, skills, and MCP servers

Research date: 2026-10-21. Three research agents produced this survey. Treat every star count, version and maturity claim as a snapshot. Items marked *(unverified)* came from directory listings and were not confirmed against the source repository.

---

## TL;DR: recommended path

1. **Keep the bible as plain markdown in git.** Split CLAUDE.md into one file per entity, with YAML frontmatter and `[[wiki-links]]`. CLAUDE.md stays the short entrypoint (§4). The same folder can be opened as an Obsidian vault.
2. **Commit custom tools in `.claude/`.** That means skills, subagents and hooks. Cloud sessions (claude.ai/code) load what is in the repo's `.claude/` but not plugins installed on your own machine. Start with `tone-warden`, `lore-keeper` and `umuco-reviewer` (§3).
3. **Add a light MCP memory layer only if it is needed.** The best choice is **Basic Memory**, whose knowledge graph *is* markdown files. The fallback is the official `server-memory`, with its file kept in the repo.
4. **Try the community fiction plugins locally before adopting one.** These are `story-skills`, `creative-writing-skills` and `Claude-Code-Novel-Writer`. Borrow their ideas and copy the parts that fit.
5. **Add Vale** with custom rules, so the tone rules that can be checked mechanically are checked in CI or a pre-commit hook. Examples are jargon, explanations of how a system works, and name spellings.

---

## 1. Claude Code skills, subagents and plugins

### Official
- **[anthropics/skills](https://github.com/anthropics/skills)**: none of its skills is for fiction. **skill-creator** is useful for writing the custom skills in §3, and **doc-coauthoring** is a general co-writing workflow.

### Community fiction packages
| Package | What it offers | Install |
|---|---|---|
| [danjdewhurst/story-skills](https://github.com/danjdewhurst/story-skills) (MIT) | 24 skills, among them worldbuilding, scene-craft, voice-style, revision-continuity and line-editing. The bible is a `story.md` file plus entity files with frontmatter. `continuity/state.md` tracks setups and payoffs, and an optional `story continuity` CLI flags broken continuity like compiler errors. | `/plugin marketplace add danjdewhurst/story-skills` then `/plugin install story-skills@story-skills` |
| [haowjy/creative-writing-skills](https://github.com/haowjy/creative-writing-skills) | 10 agents (muse, writer, critic, editor, reader-sim, character-sim, **continuity-checker**, style-creator) and 13 skills. style-creator builds a voice guide from your own prose samples. | `/plugin marketplace add haowjy/creative-writing-skills` then `/plugin install creative-writing-skills@cw` |
| [forsonny/Claude-Code-Novel-Writer](https://github.com/forsonny/Claude-Code-Novel-Writer) | A full workspace template. Commands include `/plan-novel`, `/write-chapter` and `/continuity-pass`. Agents include a continuity-editor and a worldbuilder. Hooks keep the tracking files up to date. | Clone or copy |
| wordflowlab/novel-writer-skills | A story-consistency-monitor skill that tracks physical traits, ages and so on, plus a world-building skill. *(Seen on claudemarketplaces.com.)* | — |
| claude-novel-writer (npm v0.2.1) | A Consistency Checker subagent, plus MCP servers (see §2). | — |

### Writer write-ups
- [Claude Book (HackerNoon)](https://sia.hackernoon.com/claude-book-a-multi-agent-framework-for-writing-novels-with-claude-code): an orchestrator, worker agents and a continuity agent, with quality gates between steps.
- [Crashdraft, "Claude Code Is Not Here to Write Your Novel"](https://crashdraft.substack.com/p/claude-code-is-not-here-to-write): uses the agent to audit a manuscript you wrote, rather than to write it. A good fit for a project with strict tone rules.

### How the pieces work
- **Skills** ([docs](https://code.claude.com/docs/en/skills)):
  - A skill lives at `.claude/skills/<name>/SKILL.md`, with frontmatter (`description`, `allowed-tools`, `model`, `context: fork`, `paths`).
  - Only the description stays in context until the skill is used.
  - Arguments arrive as `$ARGUMENTS`.
  - `disable-model-invocation: true` means the skill runs only when you call it.
  - Slash commands (`.claude/commands/*.md`) have been merged into skills.
- **Subagents** ([docs](https://code.claude.com/docs/en/sub-agents)):
  - A subagent is defined in `.claude/agents/<name>.md`, with `name`, `description`, `tools`, `model` and `memory`.
  - Each one starts with a fresh context that includes CLAUDE.md, and returns only a summary. That suits reviewers who should not see the drafting conversation.
- **Hooks** ([docs](https://code.claude.com/docs/en/hooks)):
  - Hooks are set in `.claude/settings.json`. The relevant events are `PostToolUse` (matcher `Edit|Write`), `Stop` and `SessionStart`.
  - A script that exits with code 2 sends its stderr back to Claude, so a hook can block or warn.
- **Plugins** ([docs](https://code.claude.com/docs/en/plugins)):
  - A plugin bundles skills, agents and hooks. It is installed with `/plugin marketplace add`.

---

## 2. MCP servers

GitHub's API was not reachable during the research, so star counts are missing. Versions and dates come from the npm and PyPI registries.

### Story bible and lore
| Server | Notes | Fit |
|---|---|---|
| [story-architect-mcp](https://github.com/PTCuong-1102/story-architect-mcp) | Character and location bible, relationship map, plot-hole and foreshadowing log, timeline-conflict checks, export to MD, EPUB and DOCX. v0.1.0 (Aug 2026). The npm name listed for it **does not resolve**, so it probably has to be cloned and built. | Closest in scope, very new |
| claude-novel-writer (npm v0.2.1, [mrskwiw](https://github.com/mrskwiw/claude-novel-writer)) | Syncs YAML and markdown files into SQLite, assembles context for a scene or character, and checks consistency. Early; the docs are inconsistent. | Good, because it works from files |
| Continuity by Ouroboros (lahutchins91/continuity-mcp) | Hosted story bible with OAuth sign-in. *(unverified)* | Lore would live outside git |
| Loreum | Commercial lore database. Changes the AI proposes go to a review queue before they become canon. *(unverified)* | Same caveat |
| Alethea World History Engine, Vibe Worldbuilding, Game World Sandbox | Generators for procedural or game worlds. Alethea is marked inactive. | Low |

### Knowledge graph and memory
| Server | Notes | Install |
|---|---|---|
| **[Basic Memory](https://github.com/basicmachines-co/basic-memory)** (PyPI v0.23.2, AGPL-3.0) | The graph is plain markdown files with `[[WikiLinks]]`, indexed in SQLite. It works with git and opens in Obsidian. **Best fit.** | `claude mcp add basic-memory -- uvx basic-memory mcp` |
| [@modelcontextprotocol/server-memory](https://github.com/modelcontextprotocol/servers) (v2.0.0) | A JSONL graph of entities, relations and observations. Simple and stable, but it duplicates the markdown bible. | `claude mcp add memory -e MEMORY_FILE_PATH=./lore/memory.jsonl -- npx -y @modelcontextprotocol/server-memory` |
| mcp-knowledge-graph (shaneholloman, v1.3.2) | A fork of the official memory server that lets you choose where the file is stored. | — |
| mcp-memory-service (doobidoo) | Semantic vector memory. Better for an agent remembering things than for structured lore. | — |

### Writers' apps
- **Obsidian**:
  - `obsidian-mcp-server` (cyanheads, v3.7.0) and `mcp-obsidian` both need Obsidian's Local REST API plugin.
  - Mostly unnecessary, because Claude Code can edit the vault files directly.
- **Notion**:
  - The official server is `@notionhq/notion-mcp-server` (v2.5.2).
  - Only worth it if the bible moves to Notion.
- **Scrivener**:
  - `scrivener-mcp` (writerslogic, v0.12.0; semantic search needs an OpenAI or OpenRouter key).
  - `@twelvetake/scrivener-mcp`.
- **World Anvil**:
  - datagen24/World-Anvil-MCP, an early implementation.
  - Needs a Grandmaster subscription and an API key, and is geared to D&D.
- **Kanka**:
  - heartran/kanka-mcp-server *(unverified)*.
- **Campfire, LegendKeeper**:
  - No MCP server found.

### Timelines, maps, names, Rwanda
- **Timelines**: Timelines MCP (Haervwe) *(unverified)*.
- **Maps**: the map servers found are all for fantasy maps (Azgaar FMG, mcp-fantasymaps), which doesn't suit a real country. A real-world or OpenStreetMap server would fit better but has not been looked into.
- **Names**: the name generators cover only fantasy datasets. **None covers Kinyarwanda names.**
- **Rwanda**:
  - `@ansvar/rwandan-law-mcp` searches Rwandan legislation. Niche, but could help with plausible civic details.
  - **No Kinyarwanda language or culture MCP exists.** A curated `reference/` folder is the realistic route.

---

## 3. Proposed custom tools for this project

Commit these under `.claude/` so every session, local or cloud, has them.

1. **`tone-warden`** (subagent; read-only tools `Read, Grep, Glob`)
   - Checks a scene against CLAUDE.md §2:
     - worldbuilding stays implicit, with no exposition
     - no jargon
     - technology stays boring to the characters
     - at most one line per background thread
   - Reports each problem with its line number, the rule it breaks, and the smallest fix. It never rewrites the scene.
2. **`lore-keeper`** (subagent with project memory, plus a `/continuity-check <scene>` skill)
   - Keeps `state.md` current: who knows what, where everyone is, relationships, which threads have appeared where.
   - Flags contradictions with the bible and with earlier scenes.
   - Borrows the setup/payoff tracking and the `exemptions.md` idea from story-skills.
3. **`umuco-reviewer`** (subagent for cultural accuracy; *umuco* means "culture")
   - Reviews gusaba and gukwa, inkwano, umuganda, imihigo, greetings, how elders are addressed, Kinyarwanda code-switching, and food.
   - Cites `world/customs/*` and `sources.md`.
   - Marks changes it projects for 2045 as speculation.
   - Lists questions for a human sensitivity reader instead of claiming authority.
4. **`/draft-scene <beat>`** (skill; `disable-model-invocation: true`)
   - Loads CLAUDE.md, the relevant cast files and the last two scenes.
   - Drafts one scene with at most one background-thread line, then runs tone-warden and lore-keeper on it.
5. **`line-editor`** (subagent)
   - Works on prose only: rhythm, filler, repeated sentence shapes, and AI tics such as tricolons, "a testament to" and piles of em-dashes.
   - Keeps the voice described in `world/voice.md`.
   - Outputs a diff, and never touches plot.
6. **Hooks** (optional backstop)
   - A `PostToolUse` hook greps edited scenes for a jargon blocklist.
   - A `Stop` hook blocks if a changed scene breaks the one-line-per-thread rule, or if `state.md` was not updated after scene edits.

The open decisions in CLAUDE.md §10 (POV, length, names, the vote outcome) are the author's to make. Agents should propose options, not settle them.

---

## 4. Repo structure

### Proposed layout
```
CLAUDE.md              # entrypoint: premise, tone rules in full, index of files, agent working rules
world/
  premise.md  complex.md  roles.md  economy.md  voice.md
  calendar.md          # real seasons pinned to months + Kinyarwanda names
  customs/{gusaba,inkwano,umuganda,imihigo}.md
  glossary.md          # Kinyarwanda terms: spelling, one-line gloss, source
cast/                  # one file per character, YAML frontmatter
threads/               # one file per background thread: carrier, appearances
stories/
  wedding/{outline,beats}.md  wedding/scenes/01-....md
  seeds.md
timeline.md  decisions.md  sources.md
.claude/{agents,skills}/  .vale.ini  styles/Kigarama/
```

### Sample character file (`cast/claudine.md`)
```markdown
---
name: Claudine
age: 50s
role: garden keeper
home: Tower ?, floor ?
status_markers: [best balcony on the floor, rare plants]
relationships:
  - { who: "[[aline]]", note: "neighbour; watches the wedding plans" }
  - { who: "[[new-family]]", note: "judges their dead basil" }
threads: []
appears_in: ["[[stories/wedding/beats]]#3"]
pov_candidate: true
---
# Claudine
Quietly judges everyone's tomatoes. Shows care indirectly — cuttings left at doors, never compliments.

## Voice
Short sentences. Talks about plants when she means people.

## Never
Explains how the co-op works. Says something kind directly.
```

### Apps and whether agents can reach them
| Tool | How an agent can get at the data |
|---|---|
| Obsidian | A folder of .md files, so it works natively. Plugins: Longform, Timelines, Calendarium, Dataview or Bases. |
| Manuskript | Open source with a plain-text file format. The most git-friendly desktop app. |
| Notion | API that reads and writes markdown. |
| World Anvil | Beta REST API ("Boromir"); only the GET endpoints are guaranteed. |
| Kanka | REST API. |
| LegendKeeper | HTML or JSON export only. |
| Campfire, Plottr, Bibisco | Export to DOCX, PDF and similar; no API. |
| Scrivener | MultiMarkdown via Compile. |

Two tools can turn the bible into a browsable website:
- **Quartz v4**: backlinks and a graph view; can publish only notes marked `publish: true`.
- **mdBook**: simpler, with no backlinks.

Two Obsidian alternatives were checked:
- **Foam** (VS Code): a reasonable option.
- **Dendron**: in maintenance mode. Avoid.

### Prose linting
- **[Vale](https://docs.vale.sh/topics/styles)** has YAML rule types that map onto the tone rules:
  - `existence` and `substitution` flag jargon.
  - `occurrence` caps how often a thread's keywords appear.
  - `consistency` keeps name spellings the same.
  - `readability` keeps the prose accessible.
  - A custom vocabulary accepts the Kinyarwanda terms.
  - It runs in pre-commit hooks or CI.
- **proselint**: generic rules.
- **LanguageTool**: its public API bans automated use, so self-host it if agents will call it.

---

## 5. Reference sources for Rwandan detail

- **Seasons and farming**: the [NISR Seasonal Agricultural Survey](https://www.statistics.gov.rw/publications/seasonal-agricultural-survey-report-2014) gives Season A as Sep–Feb, B as Mar–Jun and C as Jul–Sep. There are also [Meteo Rwanda seasonal forecasts](https://www.meteorwanda.gov.rw/fileadmin/user_upload/SEASONAL_FORECAST_SOND_2024.pdf).
  - ⚠️ **Verify before use**: how the Kinyarwanda season names (*Itumba*, *Umuhindo*, *Impeshyi*, *Urugaryi*) map to months. The sources did not fully agree.
- **Customs**:
  - New Times on [gusaba and gukwa](https://www.newtimes.co.rw/article/122843/Education/know-your-history-gusaba-and-gukwa-ceremonies)
  - New Times on [how inkwano has changed](https://www.newtimes.co.rw/article/30437/lifestyle/society/a-look-back-at-traditional-rwandan-marriage-and-evolution-of-dowry-giving)
  - An RCHA booklet, [*Indangagaciro z'ubukwe bwa Kinyarwanda*](https://www.rwandaheritage.gov.rw/fileadmin/user_upload/RCHA/Publications/Published_Books/INDANGAGACIRO_Z_UBUKWE_BWA_KINYARWANDA__Booklet.pdf)
- **Umuganda, Imihigo, Girinka**: a [UNDP report on Rwanda's home-grown solutions](https://www.ecoi.net/en/file/local/2063688/nhdr2020_ruwanda.pdf).
- **The 2045 backdrop**: [Vision 2050 (abridged)](https://planipolis.iiep.unesco.org/sites/default/files/ressources/rwanda_Vision_2050_abridged.pdf).
- **Names and language**:
  - [KT Press on popular baby names and their meanings](https://www.ktpress.rw/2026/04/the-most-popular-baby-names-in-rwanda-and-what-they-mean/)
  - The [aflat Kinyarwanda–English dictionary](https://aflat.org/content/online-kinyarwanda-dictionary)
  - [Glosbe](https://glosbe.com/)

None of this replaces a Rwandan sensitivity reader for the wedding story.
