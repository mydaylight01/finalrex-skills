# finalrex-skills

Claude Code skills for FinalRex.

## ติดตั้ง

ต้องมี Claude Code และบัญชี FinalRex

### 1. เชื่อม FinalRex MCP

เช็คว่าเชื่อมแล้วหรือยัง:

```bash
claude mcp list
```

ถ้ายังไม่มี `finalrex`:

```bash
claude mcp add --transport http --scope user finalrex https://api.finalrex.com/mcp
```

แล้วเปิด `claude` พิมพ์ `/mcp` เลือก **finalrex** แล้ว login ในเบราว์เซอร์
(ถ้าใช้ Claude desktop app ให้ sign in finalrex ที่ตั้งค่า connector แทน)

### 2. ติดตั้ง plugin

พิมพ์ใน Claude Code:

```
/plugin marketplace add mydaylight01/finalrex-skills
/plugin install finalrex@finalrex-skills
```

หรือรันใน terminal:

```bash
claude plugin marketplace add mydaylight01/finalrex-skills
claude plugin install finalrex@finalrex-skills
```

### 3. เปิด session ใหม่ แล้วเช็ค

skill จะขึ้นหลังเปิด session ใหม่เท่านั้น

```bash
claude plugin list
```

ต้องเห็น `finalrex@finalrex-skills`

### 4. ใช้งานครั้งแรก

พิมพ์ `/finalrex-report` (หรือ "เขียน daily") ครั้งแรกจะถาม workspace, board (เลือก Master Plan), โน้ตกลาง (Work Log Reports), format และชื่อของคุณ แล้วจำไว้ใช้ครั้งต่อไป

## ให้ Claude ติดตั้งให้

ก็อป prompt นี้ไปวางใน Claude Code:

```
ช่วยติดตั้ง FinalRex skills ให้หน่อย ทำตามลำดับนี้ บอกผลทีละขั้น

1. รัน `claude mcp list` เช็คว่ามี finalrex หรือยัง ถ้ายังไม่มี ให้รัน
   `claude mcp add --transport http --scope user finalrex https://api.finalrex.com/mcp`
   แล้วบอกให้ฉันไป login ผ่าน /mcp (ห้ามขอ token หรือรหัสผ่านจากฉัน)
2. รัน `claude plugin marketplace add mydaylight01/finalrex-skills`
3. รัน `claude plugin install finalrex@finalrex-skills`
4. รัน `claude plugin list` ยืนยันว่ามี finalrex@finalrex-skills
5. บอกฉันให้เปิด session ใหม่ แล้วลองพิมพ์ /finalrex-report

ถ้าขั้นไหน error ให้หยุด แล้วแสดง error ให้ฉันดู
```

## อัปเดต

```
/plugin marketplace update finalrex-skills
```

แล้วเปิด session ใหม่

## Skills

| Skill | What it does |
|-------|--------------|
| `finalrex-report` | Logs the work of the session as tasks on your FinalRex board and writes the Daily Report note. Say `/finalrex-report` or "เขียน daily". |
| `finalrex-summary` | Writes Weekly / Monthly reports per person and the team summary, as tables, from the board's tasks and Daily Reports. Say `/finalrex-summary` or "สรุป weekly". |
| `finalrex-standup` | Records a standup: yesterday / today / blockers per person, updates tasks (blockers become Open risk tasks) and writes the "Standup: dd/mm/yyyy" note. Say `/finalrex-standup` or "บันทึก standup". |
| `finalrex-plan` | Turns a request or spec into parent tasks and subtasks, checking for duplicates first. Say `/finalrex-plan` or "แตก task". |
| `finalrex-status` | Read-only: open, overdue, stuck, Hold and open-risk tasks for you, a person, a project or the team. Say `/finalrex-status` or "งานฉันค้างอะไร". |
| `finalrex-kpi` | Planned (on hold until the FinalRex MCP has KPI tools). See [ROADMAP.md](ROADMAP.md). |

Every skill reads the rules note on your board first; its rules win over the skill text.
Skills that write show a dry run and ask once before writing.

The first run asks for your workspace, board, rules note, format and name.
Your settings are saved in `~/.claude/finalrex-report.json` on your machine and shared by all the skills. They are not part of this repo.
