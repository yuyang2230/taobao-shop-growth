# Taobao Shop Growth Skill (EN)

> Language: [English](README.en.md) | [中文](README.md)

> Turn "customer-acquisition optimization for a solo-run Taobao store" into a **standard SOP that an AI agent can execute**: diagnosis → product scoring → content traffic → batch automation.
> Every method was validated end-to-end on a real, live lab-equipment Taobao store. All conclusions are marked ✅ / ⚠️ / ❌ from actual field tests — no armchair talk.

## Why this repo exists

Most public "Taobao operations" content talks in concepts and gives no **executable workflow**; even less of it teaches an **AI agent to do the work for you**.

This repo breaks down the day-to-day growth work of a small shop into SOPs an AI can follow, packaged as a skill for Claude Code / ZCode / Cursor and similar coding assistants. You say one line — "optimize all products in my store" — and the AI runs the diagnosis, rewrites titles, fills required attributes, and clears the "traffic restricted" flags, then reports back.

## Eight field-tested findings (read these first)

1. **"Traffic restricted" is usually caused by empty required attributes, not the title.**
   The red **Error(N)** in the "Optimization Suggestion" panel on the product edit page mostly comes from required attribute fields (rated voltage, drive type, material…) being empty. Fill them and resubmit: the "traffic restricted" notice on the success page disappears. Field-tested to be *more* effective than title rewrites. → [references/03-attributes.md](references/03-attributes.md)

2. **The platform pops an AI-suggested attribute dialog — clicking "confirm" is free completion.**
   The "Product attribute info update" dialog (platform-AI suggested values, pre-checked) appears occasionally on the edit page. One temperature probe went from 27% → 36% attribute completion just by confirming. Zero cost. → [references/03-attributes.md](references/03-attributes.md)

3. **The short "guide title" (≤15 Chinese chars) is a massively overlooked slot.**
   It's the title actually shown in the mobile recommendation feed; most small stores leave it empty across the whole shop. → [references/02-title-optimization.md](references/02-title-optimization.md)

4. **Title / attribute edits can be fully automated.**
   Qianniu's built-in "Product Manager" (商品管家) does conversational batch edits over official APIs (it won't click the wrong row); a dual-channel agent setup (browser agent + visual re-verification) covers what it can't. → [references/05-automation.md](references/05-automation.md)

5. **"Missing product video" needs no filming — product images become videos.**
   Main image → 3D card carousel (zero-failure fallback) or AI image-to-video (crop the text band + visual QC; fall back to the carousel on fail). Field test: 5 products rendered in a day, all uploaded and confirmed. → [references/07-product-video.md](references/07-product-video.md)

6. **Agents routinely report false success — "three instruction lines" make batch writes reproducible.**
   Put these three lines into every write task: "read the field back and self-verify after writing", "assign the whole field, never use Ctrl+A / hotkeys", "finish in one go, don't pause for confirmation". With them: 2/2 real writes in the same session; without the verify line, one silent lie ("written" while the screenshot still showed the old value). → [references/05-automation.md](references/05-automation.md)

7. **Taobao's own AI search (Qianwen) quotes structured product specs directly into its answers — low sales is not a barrier.**
   Answers are built as "scenario branches + quoted spec phrases": load demand words into the title, write spec phrases on their own lines, keep each FAQ block 134–167 characters as a standalone block — a store with single-digit sales was quoted over listings with ~100 sales. ⚠️ Red line: changing the first-level category wipes the last 30 days of sales, irrecoverably. → [references/08-geo-qianwen.md](references/08-geo-qianwen.md)

8. **For React admin pages, CDP direct connection beats coordinate-clicking.**
   After lock screen / RDP disconnect, screenshot+coordinate schemes fail **silently**; connectOverCDP needs no window focus and reads real DOM state. Four iron rules: locator.fill instead of native setters, real locator clicks instead of el.click(), insertText for long CJK text, hide overlay layers before clicking. → [references/05-automation.md](references/05-automation.md)

## Repository layout

```
taobao-shop-growth/
├── SKILL.md                     # Main skill file (AI operating instructions)
├── LICENSE                      # MIT
├── references/                  # Deep-dive modules
│   ├── 01-diagnosis.md          # Shop diagnosis: experience score → homepage templates
│   ├── 02-title-optimization.md # Title rewrite: 60-byte formula + real before/after
│   ├── 03-attributes.md         # Attribute completion: the root cause of "traffic restricted"
│   ├── 04-content-traffic.md    # Content traffic: Guangguang / Ask Everyone / detail FAQ / off-site GEO
│   ├── 05-automation.md         # AI automation: Product Manager + browser agent + scheduled jobs
│   ├── 06-b2b-playbook.md       # High-ticket B2B play (lab instruments as example)
│   ├── 07-product-video.md      # No-filming product video pipeline
│   └── 08-geo-qianwen.md        # In-site GEO: Taobao AI search (Qianwen) quoting mechanics & optimization
├── templates.md                 # Title formula / attribute specs / FAQ / content calendar / CS scripts / comment auto-replies
└── publish/                     # Ready-to-post content-platform copy
    ├── xiaohongshu-notes.md     # Xiaohongshu (RED) notes ×4 (incl. pinned-comment templates)
    └── douyin-scripts.md        # Douyin voiceover scripts ×3 (incl. comment-section hooks)
```

## Quick start

**Option A: install as an AI-agent skill (recommended)**

```bash
git clone https://github.com/yuyang2230/taobao-shop-growth.git
```

Copy the whole `taobao-shop-growth` folder into your assistant's skills directory (Claude Code / ZCode / Cursor), then, in a browser where Qianniu seller center is already logged in, tell your AI:

> **EN:** "Using the taobao-shop-growth skill: run a full shop diagnosis first, then optimize the top 10 products (titles + required attributes + video), ledger-driven, and report back."
>
> **中文:** "帮我按 taobao-shop-growth 技能，先给店铺做一次诊断，再优化前 10 件商品。"

**Option B: use it as a human runbook**

Read [SKILL.md](SKILL.md) top-down, follow `references/` in order, and execute step by step in the Qianniu seller console — no AI required.

## Who it's for

- Solo / two-person Taobao stores (C-store) without time to study operations systematically
- Older stores with many products but traffic stuck at the same level for years
- Especially B2B / high-ticket / model-number-search categories (instruments, equipment, industrial parts, hardware)
- Efficiency-minded sellers who want an AI agent to take over repetitive back-office work

## Case store

Everything in this skill was validated on a real, live **lab-instruments shop** (glass reactors, rotary evaporators, magnetic stirrators and accessories; buyers are university labs, pharma and chemical companies). The no-filming video pipeline (07) was also validated there: nine products across two batches now carry videos, all uploaded and confirmed. After the in-site GEO work (08), key products rank #2–6 on their core keywords.

> If you buy lab equipment / instruments: search **"予明仪器"** on Taobao. Contact details are in the Chinese [README.md](README.md).

## Disclaimer

- This is a personal experience write-up, **not an official Taobao/Alibaba product**. UI and features follow the current Qianniu version.
- Nothing here involves fake orders or violations; follow platform rules and execute within your capacity.
- All "field-tested" conclusions come from one store, one category, one point in time — reference only, no results promised.

## Version history

| Version | Date | Highlights |
|---------|------|------------|
| v1.2 | 2026-09-20 | New 08 in-site AI search (Qianwen) GEO module: quoting mechanics, 134–167-char FAQ blocks, spec-phrase slots, ⚠️ category-change sales-wipe red line; 05 upgraded: CDP direct channel as the primary route + React iron rules + Qianniu v2 field notes + judge-model assist + local-OCR fallback, dropdown attributes revised to "automatable via CDP"; 07 CDP upload SOP; 04 marketing-copy red line / no-edit-after-publish / cold-start self-reply; templates spec-phrase template |
| v1.1 | 2026-09-18 | 07 no-filming video pipeline; "three reliable agent instruction lines" + human-handoff list; Guanghe review consistency rule; "Ask Everyone" client-only marker; comment auto-reply scripts; bilingual README |
| v1.0 | 2026-09-17 | First release: diagnosis / titles / attributes / content / automation + templates + social publishing packs |

## License

MIT — fork, modify, and republish freely; keep attribution.
