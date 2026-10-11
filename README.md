# competitor-intel

A Claude skill that tracks daydream's competitors and tells Janet what changed. It keeps one competitor list in Attio, watches each competitor's site and the news, picks up competitors that prospects name on calls and in email, and sends a weekly digest. **Pricing is the top priority.**

**Status:** v1 · `dry_run` · built 11 Oct 2026 · owner Lauren (RevOps)

---

## How it works

| Mode | Trigger | Writes | Output |
|---|---|---|---|
| **Add** | A domain is sent to Claude ("track airops.com") | Attio Competitors entry, Snapshots baseline, Runs row | Add confirmation |
| **Detect** | Daily scheduled task, 7:27am CT | Proposed entries, linked deals, mention counts, Intel Log mention rows | One-line summary |
| **Monitor** | Weekly scheduled task, Mondays 7:52am CT (or "scan competitors now") | Intel Log items, Snapshots, `last_scanned` | Weekly digest, or a quiet-week note |

**Detect reads three sources:**

1. Circleback call notes on Attio records
2. Prospect emails synced to Attio
3. Deals that icp-qualification closed as `Out of ICP - Competitor`

**Monitor checks five surfaces per competitor:** pricing, changelog, blog/press, sitemap (new product pages), and news.

Every item is tiered **High / Medium / Low** by rule. Pricing changes, offering overlap, funding and competitor mentions on live deals are always High.

## Repo structure

```
competitor-intel/
├── README.md                       # this file
├── SKILL.md                        # Config table, hard rules, the three workflows, failure handling
├── references/
│   ├── registry-and-log.md         # Attio slugs and option titles, sheet columns, item_id formats
│   ├── detect.md                   # Sources, prospect gates, mention rules, ignore list, competitor test
│   ├── monitor.md                  # Surface discovery, fingerprints, pricing confirmation, sitemap, news
│   ├── relevance-rubric.md         # daydream's positioning and price floor, tier rules H1–L3, talk tracks
│   ├── digest-template.md          # Every chat output Janet sees
│   └── examples.md                 # Calibration cases from real Attio data
└── evals/
    └── evals.json                  # Test prompts and assertions
```

## Dependencies

| System | Used for | Access needed |
|---|---|---|
| **Attio** | Competitors list (registry), deals, Circleback notes, synced email | Read across the workspace; write to the `competitors` list only |
| **Google Sheets** | [daydream — Competitor Intel Log](https://docs.google.com/spreadsheets/d/1gtC7zc27ioWEbCK75e6P8qWutzL1n_9AItvP2lKX3Oc/edit): Intel Log, Snapshots, Runs tabs | Edit |
| **Web search / fetch** | Competitor sites and news | Built into Claude |

**Attio setup:** the Competitors list (`competitors`, parent object Companies) has 15 list attributes. The spec is in `references/registry-and-log.md`. Option titles are case-sensitive, so the skill matches them exactly.

## Configuration

Every workspace-specific value is in the **Config** table at the top of `SKILL.md`. Moving the skill to a new workspace means editing that table, not the logic.

| Key | Default | Notes |
|---|---|---|
| `MODE` | `dry_run` | Prints every write instead of making it. Set to `live` after review |
| `TIMEZONE` | `America/Chicago` | |
| `ATTIO_WORKSPACE_SLUG` | `daydreamlabs` | Used to build deal links |
| `ATTIO_LIST` | `competitors` | |
| `SHEET_ID` | Intel Log sheet ID | |
| `DETECT_LOOKBACK_DAYS` | `3` | The overlap covers weekends; the Intel Log dedupes repeats |
| `MONITOR_WINDOW_DAYS` | `8` | |
| `DAYDREAM_PRICE_FLOOR` | `$15K/month, 12-month` | Anchor for pricing comparisons and talk tracks |
| `MAX_COMPETITORS_PER_RUN` | `15` | |
| `MAX_CALLS_PER_COMPETITOR` | `12` | Pricing is always checked first |

## Install

1. Connect **Attio** and **Google Sheets** to the Claude account that will run the skill, and enable both in the chat that runs the schedules.
2. Confirm that account has edit access to the Intel Log sheet.
3. Upload the skill. Zip the `competitor-intel/` folder, or use the packaged `competitor-intel.skill`.
4. Run it once interactively in `dry_run` (for example, "run competitor-intel Detect for the last 2 weeks") and review the output.
5. Set `MODE: live` in `SKILL.md`.
6. Create the two scheduled tasks below with **automatic approval** turned on. Otherwise the Attio and Sheets writes stall waiting for a confirmation that nobody is there to give.

### Scheduled tasks

**Daily, Detect:** `CRON_TZ=America/Chicago 27 7 * * *`

> Run the competitor-intel skill in Detect mode for its default lookback in America/Chicago. Write per its Config mode, then end with the one-line daily summary.

**Weekly, Monitor:** `CRON_TZ=America/Chicago 52 7 * * 1`

> Run the competitor-intel skill in Monitor mode for all Active competitors using its default window. Write per its Config mode, then send the weekly digest, or the quiet-week note if nothing is High or Medium.

## Usage

| Say | What happens |
|---|---|
| `track profound.com` | Add mode: surfaces discovered, pricing baselined, entry set to Active |
| `what's new with AirOps?` | Monitor for that one competitor |
| `scan competitors now` | A full Monitor run (logged as `scan_now`) |
| `check for competitor mentions` | Detect for the default lookback |
| `run Detect for 1–14 Sept` | Detect with an explicit date window |

**Reviewing proposals:** Detect adds unknown competitors as `Proposed`. Janet sets them to `Active` or `Retired` in Attio → Competitors. An activated entry is baselined on the next Monday run and scanned from the following week.

## Design principles

- **Attio is read-only except the Competitors list.** Category and status changes are suggested in the digest, never written.
- **The sheet is the memory.** Each scheduled run starts fresh. Every Intel Log row has a deterministic `item_id`, so re-runs never duplicate.
- **Fingerprints, not hashes.** Fetched pages come back as summaries that drift between runs. Each surface stores the facts that matter instead.
- **Pricing changes must be confirmed.** A difference has to survive a second, targeted fetch before it's reported as High.
- **The prospect's competitors are not daydream's.** A company named as the prospect's market rival is never filed as a mention.
- **Nothing from before the baseline is "new."**

## Testing

`evals/evals.json` has six test prompts with assertions covering Detect precision, ICP-feed judgment, Monitor output, pricing-change handling and Add mode. The calibration cases in `references/examples.md` come from real daydream calls and emails.

v1 was dry-run against live data on 11 Oct 2026. 22 friction points from those runs and 11 cross-file inconsistencies were fixed before packaging. Details are in the build notes (Notion / project docs).

## Roadmap

| Version | Scope |
|---|---|
| **v1** (current) | Add, Detect, Monitor; Attio + Sheets; weekly digest in chat |
| **v2 candidates** | Write `{{competitor_2}}` to the Attio contact; LinkedIn company posts via Fuse; G2 reviews; job postings as a signal |

## Changelog

| Date | Change |
|---|---|
| 2026-10-11 | v1 built and dry-run tested. Fixed: per-call dedupe, prospect gates on notes, exact domain matching, sitemap path lists, pricing confirmation, the baseline rule, digest grouping |

---

> **Keep this repo private.** The skill files contain Attio list IDs, the Intel Log sheet ID, daydream's price floor, and calibration excerpts from real prospect calls.
