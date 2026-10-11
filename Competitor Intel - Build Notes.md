# Competitor Intel — Build Notes

**Owner:** Lauren (RevOps) · **Built:** 11 Oct 2026 · **Skill:** `competitor-intel` v1 (proto, `dry_run`)
**Destination:** Janet's Claude, as two scheduled tasks

---

## What it does

| Mode | When | What Janet sees |
|---|---|---|
| **Add** | Someone sends a domain ("track airops.com") | Confirmation: surfaces watched, today's pricing, recent High-priority context |
| **Detect** | Daily, 7:27am CT | One line: new competitor mentions and proposals. No digest |
| **Monitor** | Mondays, 7:52am CT | Weekly digest (High → Medium → Pending review), or a quiet-week note |

**Pricing is the top priority.** Every confirmed pricing change is High, appears first in the digest, and is compared against daydream's **$15K/month, 12-month** floor.

## Where things live

| | Location | Role |
|---|---|---|
| Registry | Attio → **Competitors** list (`competitors`) | Who's tracked, their status, surfaces and linked deals |
| Log | [daydream — Competitor Intel Log](https://docs.google.com/spreadsheets/d/1gtC7zc27ioWEbCK75e6P8qWutzL1n_9AItvP2lKX3Oc/edit) | Intel Log (one row per item), Snapshots (last-known page state), Runs |
| Skill | `competitor-intel/` | `SKILL.md` + 6 references + evals |

## File structure

| File | Purpose |
|---|---|
| `SKILL.md` | Config table, hard rules, the three workflows, failure handling |
| `references/registry-and-log.md` | Attio slugs and option titles, sheet columns, `item_id` formats, row mechanics |
| `references/detect.md` | Sources, prospect gates, mention rules, ignore list, competitor test |
| `references/monitor.md` | Surface discovery, fingerprints, pricing confirmation, sitemap, news search |
| `references/relevance-rubric.md` | daydream's positioning and pricing anchor, tier rules H1–L3, talk-track rules |
| `references/digest-template.md` | Every chat output |
| `references/examples.md` | Real calibration cases: Tapestry, Assort, SignLab, VideoGen, JISR, Plact, AirOps |

All workspace values sit in the Config table at the top of `SKILL.md`. Moving the skill to Janet means editing that table, not the logic.

---

## Design decisions

| Decision | Why |
|---|---|
| **Fingerprints, not hashes** | WebFetch returns a summary, and its wording drifts each fetch, so a hash would "change" every week. Each surface stores the facts that matter (latest changelog date, plan grid, product URLs). |
| **Pricing must survive a second, targeted fetch** | Re-reads of AirOps' pricing dropped add-ons and surfaced unlabeled blocks between runs. Without confirmation, H1 would fire on noise. |
| **Sitemap compares product paths, not `lastmod`** | AirOps bulk-updated ten pages on one afternoon. Only a stored path list tells new from edited. |
| **One mention per call, keyed on the Circleback meeting ID** | Circleback posts two notes per call; the first dry run double-counted every mention. |
| **Prospect gate on every source** | Most notes and emails aren't prospects: activity logs, vendor pitches, partner and contractor calls, newsletters. Only records tied to a deal are read. |
| **The prospect's competitors are never mentions** | The most common false positive in testing (Hyro for Assort, HeyGen for VideoGen, ZenHR for JISR). It's hard rule 6, and calibrated with real notes. |
| **Type comes from what they sell; Incumbent from the relationship** | "Evaluating Profound" is still an Alternative: it's software the client runs. |
| **ICP feed is judged, not trusted** | `Out of ICP - Competitor` catches SEO suppliers too (SerpProxies), so each one goes through the competitor test. |
| **Nothing before the baseline is "new"** | A competitor's first Monitor run would otherwise log weeks of history. |
| **Related items are grouped in the digest** | Ten landing pages for one offer are one item to Janet, but ten rows in the log so dedupe stays exact. |
| **Write to the first empty row, never append** | Dropdowns run to row 1000, so `append` lands on row 1001. Found and fixed during setup. |
| **Attio is read-only except the Competitors list** | Per your call. Category and status changes are suggested in the digest, never written. |
| **Ships in `dry_run`** | Same phased rollout as the other skills. |

---

## Test results (11 Oct, dry run on live data)

| Run | Result |
|---|---|
| **Detect, 28–31 Jul** | Found AirOps and Profound on the **open Tapestry deal**, and Growth X and Profound on Assort Health. Proposed Plact from the ICP feed and skipped SerpProxies. Skipped every prospect-rival, Sunbeam, and the vendor, partner and contractor calls correctly. |
| **Monitor, AirOps** | Pricing unchanged (still no public prices). Found a cluster of new **managed link-building landing pages**. Logged the changelog and blog items with the correct tiers. |
| **Consistency review** | 11 issues across the files, all fixed. |

22 friction points from the two dry runs were fixed before packaging: per-call dedupe, prospect gates on notes, exact domain matching, sitemap path lists, pricing confirmation, the baseline rule, digest grouping, the call cap (8 → 12), and the deal-link format.

---

## Rollout

| Phase | Where | Mode | Exit when |
|---|---|---|---|
| 1 | Lauren's workspace | `dry_run`: Detect on the last 2–4 weeks, one Monitor run | You agree with the mentions, tiers and talk tracks |
| 2 | Lauren's workspace | `live`: seed list added, both schedules on | Two weekly digests Janet doesn't need to correct |
| 3 | Janet's workspace | `live` | First digest lands in her Claude |

## Janet's install checklist

| # | Item |
|---|---|
| 1 | Connect **Attio** and **Google Sheets** on her account, and enable both in the chat that runs the schedules |
| 2 | Edit access on the Intel Log sheet (Lauren shares it) |
| 3 | Install the skill; set `MODE: live` in the Config |
| 4 | Create both scheduled tasks (below) with **automatic approval**, so writes don't stall waiting for a confirmation |

### Scheduled task prompts

**Daily, Detect:** `CRON_TZ=America/Chicago 27 7 * * *`
> Run the competitor-intel skill in Detect mode for its default lookback in America/Chicago. Write per its Config mode, then end with the one-line daily summary.

**Weekly, Monitor:** `CRON_TZ=America/Chicago 52 7 * * 1`
> Run the competitor-intel skill in Monitor mode for all Active competitors using its default window. Write per its Config mode, then send the weekly digest, or the quiet-week note if nothing is High or Medium.

The times are a few minutes before 7:30 and 8:00 because runs on the hour or half hour can be delayed. Detect still runs after the 7am call sweep.

---

## Open items

| # | Item | Owner |
|---|---|---|
| 1 | **Price floor conflict.** You gave $15K/month for 12 months. One July call note records $17K/month as "the floor", and recent proposals ran $20K with a 6-month opt-out. The skill uses $15K and treats the others as context only. Confirm | Lauren / Janet |
| 2 | **AirOps category.** The 2 Sep repositioning and the new managed-service pages make it Hybrid, not Software. Change it in Attio if you agree | Lauren |
| 3 | Send the seed list. Each competitor gets the Add treatment: surfaces, a pricing baseline and a search qualifier | Lauren |
| 4 | Phase 1: dry-run Detect over the last 2–4 weeks and review the Proposed list it would create (expect Growth X, Profound, Plact) | Lauren |
| 5 | Confirm the ignore list in `detect.md` (Ahrefs, Semrush, Lumar and others are never proposed) | Janet |
| 6 | Share the Intel Log sheet with Janet before phase 3 | Lauren |
| 7 | **v2 candidates:** write `{{competitor_2}}` on the contact; LinkedIn company posts via Fuse; G2 reviews; job postings | — |
