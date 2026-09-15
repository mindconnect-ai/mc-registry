---
name: db-timetables
description: Use when asked about Deutsche Bahn trains at a German station — departures, arrivals, delays, platform changes or cancellations — via the DB Timetables API with the installation's API key
tools: bash
---
# Deutsche Bahn timetables

The DB API Marketplace's **Timetables API** answers three questions for one
station: what is planned for an hour, what has changed, and what changed in
the last two minutes. It speaks XML and needs two credentials.

## Credentials

The credentials sit in the environment `bash` inherits — never in a prompt,
never in code you write:

| Variable | Header it goes into |
|----------|---------------------|
| `DB_CLIENT_ID` | `DB-Client-Id` |
| `DB_API_KEY` | `DB-Api-Key` |

Check they exist before anything else, without printing them:

```bash
test -n "$DB_CLIENT_ID" && test -n "$DB_API_KEY" && echo ok || echo "missing DB_CLIENT_ID / DB_API_KEY"
```

If they are missing, stop and tell the user: the operator registers at
https://developers.deutschebahn.com/, creates an application, subscribes it to
the free *Timetables* plan and exports the two values. Do **not** use
`code_execute` for these calls — its container has neither the variables nor,
by default, network access, and pasting a key into code would put it into the
conversation.

## Endpoints

Base URL: `https://apis.deutschebahn.com/db-api-marketplace/apis/timetables/v1`

| Call | What it returns |
|------|-----------------|
| `GET /station/{pattern}` | stations whose name matches; gives the `eva` number every other call needs |
| `GET /plan/{evaNo}/{YYMMDD}/{HH}` | the planned stops in that **one hour slice** (local time) |
| `GET /fchg/{evaNo}` | every known change for the station — delays, platforms, cancellations, messages (large) |
| `GET /rchg/{evaNo}` | only the changes of the last two minutes |

Every call sends the two headers and `Accept: application/xml`. The free plan
allows 60 calls a minute.

```bash
DB="https://apis.deutschebahn.com/db-api-marketplace/apis/timetables/v1"
curl -sS "$DB/station/Frankfurt%20Hbf" \
  -H "DB-Client-Id: $DB_CLIENT_ID" -H "DB-Api-Key: $DB_API_KEY" -H "Accept: application/xml"
```

URL-encode the pattern (spaces as `%20`). The answer is
`<stations><station name="Frankfurt(Main)Hbf" eva="8000105" ds100="FF" .../></stations>`
— take `eva`. A plan for one hour, and the changes to merge into it:

```bash
EVA=8000105; DAY=$(TZ=Europe/Berlin date +%y%m%d); HOUR=$(TZ=Europe/Berlin date +%H)
curl -sS "$DB/plan/$EVA/$DAY/$HOUR" -H "DB-Client-Id: $DB_CLIENT_ID" -H "DB-Api-Key: $DB_API_KEY" -H "Accept: application/xml" > /tmp/plan.xml
curl -sS "$DB/fchg/$EVA"            -H "DB-Client-Id: $DB_CLIENT_ID" -H "DB-Api-Key: $DB_API_KEY" -H "Accept: application/xml" > /tmp/fchg.xml
```

The hour slice is the station's local time, hence `TZ=Europe/Berlin`. A
departure board for "the next hour" usually needs the current and the
following slice.

## The XML

Both plan and changes are a `<timetable>` of stops `<s>`; each stop is one
train at this station, identified by `id`, with an arrival `<ar>`, a
departure `<dp>` (a starting train has no `<ar>`, a terminating one no
`<dp>`) and the train's label `<tl>`:

```xml
<timetable station="Frankfurt(Main)Hbf">
  <s id="-1234567890-2409151230-5">
    <tl f="F" t="p" o="80" c="ICE" n="595"/>
    <ar pt="2409151228" pp="7" ppth="Berlin Hbf|Braunschweig Hbf|Kassel-Wilhelmshöhe" l=""/>
    <dp pt="2409151233" pp="7" ppth="Mannheim Hbf|Stuttgart Hbf|München Hbf" l=""/>
  </s>
</timetable>
```

| Attribute | Meaning |
|-----------|---------|
| `tl/@c`, `tl/@n` | train category and number — `ICE 595`, `RE 20`, `S 8` |
| `dp/@l`, `ar/@l` | the line, when there is one (`S8`, `RE20`) |
| `pt`, `pp`, `ppth` | planned time (`YYMMDDHHmm`, local), platform, path (stations `|`-separated; the last of `dp/@ppth` is the destination, the first of `ar/@ppth` the origin) |
| `ct`, `cp`, `cpth` | the **changed** time, platform, path — only in `fchg`/`rchg`, only when changed |
| `cs="c"` | the stop is **cancelled**; `ps`/`cs` are the planned / changed status |
| `<m>` | messages on the stop or on `ar`/`dp`: `t` is the kind (`d` delay reason, `h` general, `q` quality), `c` a reason code, `ts` when it was issued |

Merge by `s/@id`: a change element carries only what changed, so a stop in
`fchg` with `<dp ct="…"/>` and no `pt` is the planned stop plus a new time.
Delay = `ct - pt` in minutes.

## Turning it into an answer

Parse with the Python that comes with the host — nothing beyond the standard
library — and print a board; read the XML yourself when there is no
`python3`:

```bash
python3 - <<'PY'
import xml.etree.ElementTree as ET
plan = {s.get("id"): s for s in ET.parse("/tmp/plan.xml").getroot()}
chg = {s.get("id"): s for s in ET.parse("/tmp/fchg.xml").getroot()}
def t(v): return f"{v[6:8]}:{v[8:10]}" if v else ""
rows = []
for sid, s in plan.items():
    dp = s.find("dp")
    if dp is None: continue
    tl = s.find("tl"); c = chg.get(sid); cdp = c.find("dp") if c is not None else None
    ct = cdp.get("ct") if cdp is not None else None
    cancelled = cdp is not None and cdp.get("cs") == "c"
    train = (dp.get("l") or f'{tl.get("c")} {tl.get("n")}')
    dest = dp.get("ppth", "").split("|")[-1]
    platform = (cdp.get("cp") if cdp is not None and cdp.get("cp") else dp.get("pp"))
    rows.append((dp.get("pt"), t(dp.get("pt")), t(ct), train, dest, platform, cancelled))
for _, pt, ct, train, dest, platform, cancelled in sorted(rows):
    status = "CANCELLED" if cancelled else (f"now {ct}" if ct and ct != pt else "on time")
    print(f"{pt}  {train:<8} {dest:<28} pl. {platform or '-':<4} {status}")
PY
```

Report what the user asked for — the next departures, one train's delay, the
platform — not the whole board, and say which hour slice you looked at. When
the plan for an hour is empty, the hour is further out than the API holds,
or the station serves no trains then; say so rather than guessing.
