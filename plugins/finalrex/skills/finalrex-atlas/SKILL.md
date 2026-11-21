---
name: finalrex-atlas
description: Turn a plan or idea (text, a file, or the current conversation) into a FinalRex Atlas map - card nodes and links between them, plus reference nodes that pin existing book pages, notes and tasks (found by search, never duplicated) - laid out in tiers so cards do not overlap, written with batch_atlas after a dry run and one confirmation. Use when the user says /finalrex-atlas, "วาง atlas", "ทำแผนที่ความคิด", "วาดแผนลง atlas", "map this on atlas", "put this plan on a canvas", "mind map this".
---

# FinalRex atlas

Lay a plan out on an Atlas (ADR-069): a canvas of card nodes (plain text) and reference nodes (a book page, a note or a task) joined by edges that use the workspace's link types.

Settings: `~/.claude/finalrex-report.json` (shared with the other finalrex skills). This skill adds one key and keeps every other key:

```json
"atlas": { "id": "...", "title": "...", "workspace": "<slug or null for personal>" }
```

Ask every question with AskUserQuestion, in Thai. Talk to the user in Thai, short. Quoted English strings below are meant to be shown to the user: translate them to Thai when you say them.

## Names, not ids

- In chat, AskUserQuestion questions and options, confirmations, dry-run summaries and reports, refer to workspaces, boards, views, tasks, notes, books, pages, atlases, nodes and people by name or title (plus the url a tool returns when linking). Never show raw ids / UUIDs unless the user asks for them.
- Before every `mcp__finalrex__*` call, write one short Thai line saying what it will do in names and counts, e.g. "Will read 20 tasks from the board Master Plan" or "Will create 3 pages in the book Team Handbook", so the user understands the permission prompt, which shows raw ids.
- Tool calls still pass ids / UUIDs exactly as before. If a name is not known yet, take it from an earlier `list_*` / `get_*` result (look it up first if cheap) instead of printing the id.

## 0. Connect FinalRex

Look for `mcp__finalrex__*` tools (ToolSearch "+finalrex" if deferred). If none exist, stop and tell the user, in Thai, to run `claude mcp add --transport http --scope user finalrex https://api.finalrex.com/mcp`, then `/mcp` › finalrex › log in, and run the skill again. Auth error → tell them to sign in the same way. Never ask for tokens or passwords.

**Brain.** Atlas needs FinalRex Brain to create one. If a call fails with 402 `limit_reached` (or says Brain / the `atlas` feature is needed), stop and say in Thai: "Atlas needs FinalRex Brain. Ask the account owner to turn on Brain on the plan page, then try again.". A 402 on `atlas.editors.max` means the atlas has too many editors; tell the user and stop.

## 1. Pick the atlas

1. `list_atlases` (yours and the ones shared with you). If `workspace` is saved in the json, also `list_atlases` with that workspace slug. Keep only atlases whose `role` can edit (owner, manager, editor).
2. Ask "Which atlas should I put this on?":
   - the saved `atlas` first, if it is still in the list — description: "used last time"
   - up to 3 other atlases (label = title)
3. **The connector cannot create an atlas.** None fits, or the user wants a new one → tell them in Thai: "Create a new atlas in the web app first (menu Atlas › My atlases), then run this again or paste the atlas link." and stop. A pasted atlas link `/atlas/<id>` → use that id.
4. Save `atlas: { id, title, workspace }` in the json, keeping the other keys.

## 2. Get the plan

- Input: text in the message, a file path (read it), or "from this conversation" (use the current conversation). Nothing given → ask in chat: "Paste the plan / idea, or give a file path.".
- Pull out: the goal, the main parts, the steps or ideas under each part, and how they relate (leads to, depends on, blocks, part of). Do not invent parts the plan does not have.

## 3. Read the atlas and find references

1. `get_atlas`: existing `nodes` (kind, text or reference title, `targetId`, x, y, w, h, version), `edges` (from, to, linkTypeId, label) and `linkTypes`.
2. **Do not duplicate.** A planned card whose text matches an existing card, or a reference whose target is already on the canvas (same `targetId`), → reuse that node (as an edge end); do not create it again. Cards stay plain text, so do not paste a long body into a card (up to 5000 characters; keep cards to a title and 1-3 short lines).
3. **Pin existing work as references** instead of retyping it:
   - tasks and notes: `search` with 2-3 keywords per part (`kinds` `task` / `note`, `workspace` = the atlas workspace when it has one).
   - book pages: `list_books` (and with the workspace), `get_book` of the likely books, match page titles. (`search` does not cover book pages.)
   - A clear match → a reference node (`kind` `page` / `note` / `task`, `targetId` = its id). Unsure → ask. Only pin what the user can open; a target they cannot open is refused.
4. **Edges.** Pick a `linkTypeId` from `linkTypes` whose name fits the relation (e.g. blocks, depends on, relates to). None fits → `""` (plain line) and put the relation in `label` (up to 200 characters). Never invent a link type id.

## 4. Lay out (no overlaps)

Default node size is w 240 × h 120. Use a tier layout:

- Tier 0: the goal card. Tier 1: the main parts. Tier 2: steps / refs under each part.
- Column step 300 (240 + 60 gap), row step 200 (120 + 80 gap). x = left edge of column, y = top of tier.
- Children of one part sit in consecutive columns under it; center a parent over its children.
- Long tiers (more than about 6 nodes): wrap into a second row in the same tier block.
- **Existing atlas:** place the new group to the right of everything there: start x = max(x + w) of existing nodes + 200, start y = min(y) of existing nodes (or 0 when empty). Never move existing nodes unless the user asks.
- Check every new node's box against all nodes (new and existing) before the dry run; shift right by 300 until none overlap.
- Up to 200 ops per batch (nodes + edges). The atlas holds at most 1,000 nodes and 2,000 edges.

Show the plan:

```
Atlas: <title>
New cards
- [goal] <text> (x, y)
- <text> (x, y)
References (pins to existing items)
- task: <title> / note: <title> / page: <title>
Links
- <from> → <to> (<link type | label>)
Already exists (not created again): <node>
```

## 5. Dry run, confirm once, run, verify

1. `batch_atlas` with `dryRun: true`. Ops:
   - card: `{"op":"create_node","ref":"goal","kind":"card","text":"...","x":0,"y":0}`
   - reference: `{"op":"create_node","ref":"t1","kind":"task","targetId":"<task id>","x":300,"y":200}`
   - edge: `{"op":"create_edge","from":"goal","to":"t1","linkTypeId":"<id or empty>","label":"..."}` (`from` / `to` = a ref or an existing node id)
   Fix any error (the whole batch is validated first; nothing is written when one op is invalid).
2. **Confirm once:** show the list from step 4 and ask "Put this on the atlas as listed? (N cards, N references, N links)". Do not write before the yes.
3. Run `batch_atlas` for real with the same ops. It is one transaction: all or nothing. A version conflict → `get_atlas` again and rebuild the ops.
4. Verify with `get_atlas`: the new nodes and edges are there.

**Never:** delete or move existing nodes or edges, change atlas settings, visibility or people, unless the user explicitly asks (settings are web-only anyway). No secrets, credentials, tokens or customer personal data (names, emails, phones) in cards or labels.

## 6. Tell the user

Short, in Thai. Use the url the tool returns; never build links yourself.

```
Put on the atlas
Atlas: <title> — <atlas url>
- N new cards, N references (N tasks, N notes, N pages), N links
Already exists (not created again): <node>
```
