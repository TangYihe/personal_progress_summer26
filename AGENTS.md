# Agent & repo rules

This repo tracks progress on the **model-training / policy-learning** workstream of a
summer-2026 robotics project. It is a *notes* repo, not a code repo. Read this before working.

## Repo structure & each doc's job

| File / dir | Job | Rules |
|------------|-----|-------|
| `README.md` | **Entry point & overview** (#1). High-level plan, current status, milestones, compute status, open questions. | Keep **concise** — key message must not be buried. Human-readable, scannable. Holds the *context section* a fresh agent reads first. Details go to `notes/`, not here. |
| `HANDOFF.md` | **Session bookmark** (#2). "Where we left off / where we start next." | **Current state only.** No history, no permanent content. Short. Update at the end of every working session. |
| `notes/` | **Detailed notes** (#3), one per sub-topic/milestone. | **Create lazily** — only when we actually reach a milestone. A simple topic = one `.md`; a big topic = a folder with an `index.md` + linked sub-pages. Don't pre-scaffold placeholders; plans change. |
| `AGENTS.md` | **These rules** (#4). | Update when conventions change. |

## Linking convention (Notion-style)

Cross-link docs with relative markdown links: `[label](path/to/doc.md)` (and `#L`/anchors for
sections). For folder-based milestones, the folder's `index.md` is the "page", sub-docs are its
"subpages", and the index links down to them while they link back up to the index. Keep the web
navigable — every detailed note should be reachable from the README or a milestone index.

## Update cadence (per working session)

1. Do the work; put detail in the right `notes/` doc (create it if the milestone is now live).
2. Update `HANDOFF.md` to reflect the *new* current state and next pickup point.
3. If a real milestone was hit or status changed, add a concise line to the README progress
   log / status / compute checklist. Don't dump session detail into the README.
4. Bump the "Last updated" / "Last session" dates.

## Tone

- **Academic / research content** (paper notes, method write-ups, surveys): concise and formal.
- Planning / status / handoff: plain and direct; reframes and morale notes are welcome where
  they help (this is also a personal progress tracker), but keep them out of formal notes.

## Conventions

- **Dates are absolute** (`2026-06-02`), never relative ("today", "last week").
- Newest-first for logs.
- Prefer editing the existing right-home doc over creating a new overlapping one.
