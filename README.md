# Spark Calendar API

A small, public, **static JSON feed of Spark program dates** — session start/end,
withdrawal deadlines (per meeting pattern), Learning Coach orientations, and
program-specific no-class windows (e.g. "no class during state testing").

This is the **program** calendar. School-wide dates (holidays, breaks, the school
testing window itself) live in the separate [School Dates API](https://dates.warner.click)
— consumers that need both fetch both.

## Endpoint

| URL | Use |
| :-- | :-- |
| `https://raw.githubusercontent.com/warnerwes/spark-calendar/main/v1/spark-calendar.json` | **Server-side** (GAS, Node) — preferred, no custom-domain TLS dependency. |
| `https://spark.warner.click/v1/spark-calendar.json` | Browser apps (once the custom domain is wired). |

CORS-open, no auth, cacheable. ISO `YYYY-MM-DD` dates; ranges inclusive; all
date-only values are local dates in `America/Los_Angeles`; `generatedAt` is UTC.

## Shape (v1)

```jsonc
{
  "version": "1",
  "schemaVersion": 1,
  "generatedAt": "2026-06-19T17:46:33Z",
  "timezone": "America/Los_Angeles",
  "years": {
    "2026-2027": {
      "label": "2026-2027",
      "sessions": [
        { "id": "s1", "label": "Session 1", "begins": "2026-09-08", "ends": "2026-12-18",
          "withdrawal": [ {"pattern":"MW","date":"2026-09-22"}, {"pattern":"TT","date":"2026-09-21"} ] }
      ],
      "orientations": [ {"label":"Learning Coach Orientation","date":"2026-09-01"} ],
      "noClassWindows": [ {"label":"State Testing","start":"2027-04-19","end":"2027-04-30"} ]
    }
  }
}
```

`withdrawal[].pattern` is a class **meeting pattern** id (`MW` = Mon/Wed, `TT` =
Tue/Thu). A class resolves its own deadline by its `meetingPatternId`.

## Resolve "current" (copy-paste recipe)

```js
const within = (a, b, iso) => a <= iso && iso <= b;
const yearKeyFor = (cal, iso) => Object.keys(cal.years).find(k =>
  cal.years[k].sessions.some(s => within(s.begins, s.ends, iso))) || null;
const currentSession = (cal, iso) => {
  const y = yearKeyFor(cal, iso); if (!y) return null;
  return cal.years[y].sessions.find(s => within(s.begins, s.ends, iso)) || null;
};
const withdrawalFor = (session, meetingPatternId) =>
  (session.withdrawal.find(w => w.pattern === meetingPatternId) || {}).date || null;
```

## How it's produced

**Source of truth:** the `programCalendar` config edited by admins at
`pca.roboteamup.com/admin?tab=calendar` (Firestore `appSettings/websiteConfig`).
The publisher (`scripts/publishSparkCalendar.cjs` in the roboteamup.com repo) reads
that config, validates it, and writes `v1/spark-calendar.json` here. GitHub
Pages/CDN serves it. Reader/writer split = free, cacheable, un-spammable.

Versioned: `/v1/` is permanent. Breaking changes ship as `/v2/` alongside `/v1/`.
