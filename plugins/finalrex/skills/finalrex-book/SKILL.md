---
name: finalrex-book
description: Turn a topic, a file or the current conversation into FinalRex Book pages - pick or create the book, plan a page tree, check existing pages so nothing is duplicated (update instead), and write the pages as Markdown drafts with batch_pages after a dry run and one confirmation. Never publishes. Use when the user says /finalrex-book, "เขียนลง book", "สร้างหนังสือ", "เขียนหน้า book", "ลงคู่มือใน finalrex", "write a book page", "put this in a book", "turn this into a book".
---

# FinalRex book

Write knowledge into a Book (ADR-068): a tree of Markdown pages. This skill writes **drafts only**. Readers see a page only after a person presses Publish in the web app; the connector has no publish tool.

Settings: `~/.claude/finalrex-report.json` (shared with the other finalrex skills). This skill adds one key and keeps every other key:

```json
"book": { "id": "...", "title": "...", "workspace": "<slug or null for personal>" }
```

Ask every question with AskUserQuestion, in Thai. Talk to the user in Thai, short.

## Names, not ids

- In chat, AskUserQuestion questions and options, confirmations, dry-run summaries and reports, refer to workspaces, boards, views, tasks, notes, books, pages, atlases, nodes and people by name or title (plus the url a tool returns when linking). Never show raw ids / UUIDs unless the user asks for them.
- Before every `mcp__finalrex__*` call, write one short Thai line saying what it will do in names and counts, e.g. "จะอ่านงาน 20 รายการจากบอร์ด Master Plan" or "จะสร้างหน้า 3 หน้าในหนังสือ คู่มือทีม", so the user understands the permission prompt, which shows raw ids.
- Tool calls still pass ids / UUIDs exactly as before. If a name is not known yet, take it from an earlier `list_*` / `get_*` result (look it up first if cheap) instead of printing the id.

## 0. Connect FinalRex

Look for `mcp__finalrex__*` tools (ToolSearch "+finalrex" if deferred). If none exist, stop and tell the user, in Thai, to run `claude mcp add --transport http --scope user finalrex https://api.finalrex.com/mcp`, then `/mcp` › finalrex › log in, and run the skill again. Auth error → tell them to sign in the same way. Never ask for tokens or passwords.

**Brain.** Creating a book needs FinalRex Brain. If a call fails with 402 `limit_reached` (or says Brain / the `book` feature is needed), stop and say in Thai: "สร้าง book ต้องมี FinalRex Brain ให้เจ้าของบัญชีเปิด Brain ในหน้า plan ก่อน แล้วลองใหม่". A 402 on `book.editors.max` means the book has too many editors; tell the user and stop. Editing a book someone invited you to does not need Brain.

## 1. Pick the book

1. `list_books` (your books and the books shared with you). If `workspace` is saved in the json, also `list_books` with that workspace slug. Keep only books whose `role` can edit (owner, manager, editor); a `reader` cannot write.
2. Ask "จะเขียนลง book ไหน?":
   - the saved `book` first, if it is still in the list — description: "ใช้ครั้งก่อน"
   - up to 2 other books (label = title, description = personal / workspace)
   - "สร้าง book ใหม่"
3. "สร้าง book ใหม่" → ask in chat for the title (default from the topic), then ask "เก็บไว้ที่ไหน?": "ส่วนตัว" / the saved workspace name. Language = the language of the content (`th` or `en`). Do not pass `visibility` (it stays `private`) unless the user asked for `invited`. Create it after the confirmation in step 5, not before.
4. Save `book: { id, title, workspace }` in the json, keeping the other keys (for a new book, save after it is created).

## 2. Get the content

- Input: a topic in the message, a file path (read it), or "จากที่คุยกัน" (use the current conversation). Nothing given → ask in chat: "บอกหัวข้อ วางเนื้อหา หรือบอก path ไฟล์ได้เลย".
- Pull out the sections that should be pages. Keep what the source says; do not invent facts, numbers or dates. Mark anything you had to assume as "(ต้องเช็ค)".

## 3. Check what exists

1. `get_book` for the table of contents (every page: `id`, `parentId`, `title`, `version`, `status`).
2. For each planned page, look for an existing page with the same or a very close title (any level). Unsure → `get_page` and compare the body.
3. Match → **update** that page instead of creating a new one: `get_page` for its current draft and `version`, then write the old draft plus your changes (merge; do not drop content the user wrote). Keep every image line `![alt](fr-img:<id>)` exactly as it is.
4. Unsure if it is the same topic → ask "หน้า <title> มีอยู่แล้ว จะทำยังไง?": "อัปเดตหน้าเดิม" / "สร้างหน้าใหม่แยก".

## 4. Plan the tree

- One page per section, sub-pages for sub-sections. Keep it shallow (2 levels is usually enough). New pages go under an existing parent page when one fits.
- Titles short and clear, up to 200 characters, in the language of the content.
- Body = Markdown: headings, lists, tables, code blocks. Up to 200 KB per page; split a longer page.
- Link to another page with `[[page:<uuid>]]` (existing pages only: a batch `ref` is not replaced inside a body, so name a page created in the same batch by its title in plain text). A note or task can be linked with `[[note:<uuid>]]` / `[[task:<uuid>]]` when you have its id from `search`.
- Up to 200 ops per batch. More pages → split into several batches, parents first.

Show the plan:

```
Book: <book title> (<ส่วนตัว | workspace>)
สร้างใหม่
- <page>
  - <sub-page>
อัปเดต (มีอยู่แล้ว)
- <page> — <what changes, one line>
```

## 5. Dry run, confirm once, run, verify

1. `batch_pages` with `dryRun: true` (a new book has no id yet: skip the dry run for that book and check the ops yourself). Ops:
   - new page: `{"op":"create","ref":"p1","title":"...","body":"...","parentId":"<page id or ref>"}`
   - existing page: `{"op":"update","pageId":"<id>","version":<version from get_page>,"body":"..."}`
   Fix any error.
2. **Confirm once:** show the tree from step 4 and ask "เขียนลง book ตามนี้ไหม? (สร้าง N หน้า, อัปเดต N หน้า, เป็น draft ยังไม่ publish)". Do not write before the yes. This is the only confirmation; creating a new book is part of it.
3. New book → `create_book` now (title, language, `workspace` if chosen). Then run `batch_pages` for real with the same ops. A conflict on `version` → `get_page` again, merge, retry that op only.
4. If the batch stopped partway, the error lists the ops already applied; do the rest without creating pages twice (`get_book` to check).
5. Verify with `get_book`: the new and updated pages are there with status `draft` or `changed`.

**Never:** publish, change visibility, invite people, move or delete pages unless the user explicitly asks for it (publish and visibility are not possible through the connector at all; tell the user to do them in the web app). No secrets, credentials, tokens or customer personal data (names, emails, phones) in pages.

## 6. Tell the user

Short, in Thai. Use the urls the tools return; never build links yourself.

```
เขียนลง book แล้ว (draft ยังไม่ publish)
Book: <title> — <book url>
- สร้าง: <page> — <page url>
- อัปเดต: <page> — <page url>
กด Publish ในเว็บเมื่อพร้อมให้คนอ่านเห็น
```
