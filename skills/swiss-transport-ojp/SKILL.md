---
name: swiss-transport-ojp
description: Use when asked about Swiss public transport — SBB train connections, departures or arrivals at a stop, or finding a Swiss station — via the opentransportdata.swiss OJP 2.0 API with the installation's token
tools: bash
---
# Swiss public transport (OJP 2.0)

opentransportdata.swiss, the Swiss open-data platform for public transport,
answers journey questions through the **Open Journey Planner (OJP) 2.0** API:
find a stop, list its departures, plan a trip. Trains, trams, buses and boats
of every Swiss operator are in it, SBB included. The API is XML over POST and
needs a token.

## The token

The token sits in the environment `bash` inherits — never in a prompt, never
in code you write:

| Variable | Header it goes into |
|----------|---------------------|
| `OPENTRANSPORTDATA_API_KEY` | `Authorization: Bearer …` |

```bash
test -n "$OPENTRANSPORTDATA_API_KEY" && echo ok || echo "missing OPENTRANSPORTDATA_API_KEY"
```

If it is missing, stop and tell the user: the operator creates a free account
at https://api-manager.opentransportdata.swiss/, subscribes to the *OJP 2.0*
API and exports the key. Do **not** use `code_execute` for these calls — its
container has neither the variable nor, by default, network access. The free
plan allows 50 requests a minute.

## One endpoint, three requests

Everything is a `POST https://api.opentransportdata.swiss/ojp20` with
`Content-Type: application/xml` and an XML body. Write the body to a file,
then:

```bash
curl -sS https://api.opentransportdata.swiss/ojp20 \
  -H "Authorization: Bearer $OPENTRANSPORTDATA_API_KEY" \
  -H "Content-Type: application/xml" --data-binary @/tmp/request.xml > /tmp/response.xml
```

Every body has the same frame; only the inner request differs. Replace `NOW`
with the current UTC time (`date -u +%Y-%m-%dT%H:%M:%SZ`); `RequestorRef` is
any name for the caller.

### 1. Find a stop — LocationInformationRequest

Stops are addressed by their 7-digit number (`8507000` Bern, `8503000`
Zürich HB, `8500010` Basel SBB, `8505000` Luzern, `8501120` Lausanne,
`8506000` Winterthur, `8502113` Aarau). For any other name, ask:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<OJP xmlns="http://www.vdv.de/ojp" xmlns:siri="http://www.siri.org.uk/siri" version="2.0">
  <OJPRequest>
    <siri:ServiceRequest>
      <siri:RequestTimestamp>NOW</siri:RequestTimestamp>
      <siri:RequestorRef>mindconnect</siri:RequestorRef>
      <OJPLocationInformationRequest>
        <siri:RequestTimestamp>NOW</siri:RequestTimestamp>
        <InitialInput><Name>Interlaken</Name></InitialInput>
        <Restrictions><Type>stop</Type><NumberOfResults>5</NumberOfResults></Restrictions>
      </OJPLocationInformationRequest>
    </siri:ServiceRequest>
  </OJPRequest>
</OJP>
```

Answer: `OJPLocationInformationDelivery/PlaceResult/Place/StopPlace` with
`StopPlaceRef` (the number) and `StopPlaceName/Text`, plus a `Probability`
per hit — take the highest, or ask the user when two are close.

### 2. Departures at a stop — StopEventRequest

```xml
<?xml version="1.0" encoding="UTF-8"?>
<OJP xmlns="http://www.vdv.de/ojp" xmlns:siri="http://www.siri.org.uk/siri" version="2.0">
  <OJPRequest>
    <siri:ServiceRequest>
      <siri:RequestTimestamp>NOW</siri:RequestTimestamp>
      <siri:RequestorRef>mindconnect</siri:RequestorRef>
      <OJPStopEventRequest>
        <siri:RequestTimestamp>NOW</siri:RequestTimestamp>
        <Location>
          <PlaceRef><StopPlaceRef>8507000</StopPlaceRef><Name><Text>Bern</Text></Name></PlaceRef>
          <DepArrTime>NOW</DepArrTime>
        </Location>
        <Params>
          <NumberOfResults>10</NumberOfResults>
          <StopEventType>departure</StopEventType>
          <IncludeOnwardCalls>false</IncludeOnwardCalls>
          <UseRealtimeData>full</UseRealtimeData>
        </Params>
      </OJPStopEventRequest>
    </siri:ServiceRequest>
  </OJPRequest>
</OJP>
```

`StopEventType` is `departure`, `arrival` or `both`; `DepArrTime` is the
moment to start from. Answer: one `StopEventResult/StopEvent` per service:

| Path under `StopEvent` | Meaning |
|------------------------|---------|
| `ThisCall/CallAtStop/ServiceDeparture/TimetabledTime` | planned departure (UTC) |
| `ThisCall/CallAtStop/ServiceDeparture/EstimatedTime` | real-time departure, when known — the delay is the difference |
| `ThisCall/CallAtStop/PlannedQuay/Text`, `EstimatedQuay/Text` | platform, and the changed platform |
| `Service/PublishedServiceName/Text` | the line as printed — `IC 1`, `S 3`, `RE 8` |
| `Service/DestinationText/Text` | where it goes |
| `Service/Mode/PtMode` | `rail`, `bus`, `tram`, `water` |
| `Service/Cancelled` | `true` when the service does not run |

### 3. Plan a trip — TripRequest

```xml
<?xml version="1.0" encoding="UTF-8"?>
<OJP xmlns="http://www.vdv.de/ojp" xmlns:siri="http://www.siri.org.uk/siri" version="2.0">
  <OJPRequest>
    <siri:ServiceRequest>
      <siri:RequestTimestamp>NOW</siri:RequestTimestamp>
      <siri:RequestorRef>mindconnect</siri:RequestorRef>
      <OJPTripRequest>
        <siri:RequestTimestamp>NOW</siri:RequestTimestamp>
        <Origin>
          <PlaceRef><StopPlaceRef>8507000</StopPlaceRef><Name><Text>Bern</Text></Name></PlaceRef>
          <DepArrTime>2026-09-16T07:30:00Z</DepArrTime>
        </Origin>
        <Destination>
          <PlaceRef><StopPlaceRef>8503000</StopPlaceRef><Name><Text>Zürich HB</Text></Name></PlaceRef>
        </Destination>
        <Params>
          <NumberOfResults>4</NumberOfResults>
          <IncludeIntermediateStops>false</IncludeIntermediateStops>
          <UseRealtimeData>full</UseRealtimeData>
        </Params>
      </OJPTripRequest>
    </siri:ServiceRequest>
  </OJPRequest>
</OJP>
```

`DepArrTime` on the origin means "leave from"; put it on the destination
instead to mean "arrive by". Answer: `TripResult/Trip` with `Duration`
(ISO 8601, `PT56M`), `StartTime`, `EndTime`, `Transfers`, and one `Leg` per
segment — `TimedLeg` for a ride (`LegBoard` and `LegAlight` with
`StopPointName/Text`, `TimetabledTime`, `EstimatedTime`, `PlannedQuay`;
`Service/PublishedServiceName/Text` for the line) and `TransferLeg` for a
walk between platforms (`Duration`).

## Reading the answer

Times are UTC; convert to Europe/Zurich for the user. Parse with the host's
Python — standard library only:

```bash
python3 - <<'PY'
import xml.etree.ElementTree as ET
from datetime import datetime
from zoneinfo import ZoneInfo
ns = {"o": "http://www.vdv.de/ojp", "s": "http://www.siri.org.uk/siri"}
root = ET.parse("/tmp/response.xml").getroot()
def local(t): return datetime.fromisoformat(t.replace("Z", "+00:00")).astimezone(ZoneInfo("Europe/Zurich")).strftime("%H:%M") if t else ""
def text(el, path):
    f = el.find(path, ns); return (f.text or "") if f is not None else ""
for ev in root.iterfind(".//o:StopEvent", ns):
    dep = ev.find("o:ThisCall/o:CallAtStop/o:ServiceDeparture", ns)
    planned, est = local(text(dep, "o:TimetabledTime")), local(text(dep, "o:EstimatedTime"))
    line = text(ev, "o:Service/o:PublishedServiceName/o:Text"); dest = text(ev, "o:Service/o:DestinationText/o:Text")
    quay = text(ev, "o:ThisCall/o:CallAtStop/o:EstimatedQuay/o:Text") or text(ev, "o:ThisCall/o:CallAtStop/o:PlannedQuay/o:Text")
    status = "cancelled" if text(ev, "o:Service/o:Cancelled") == "true" else (f"now {est}" if est and est != planned else "on time")
    print(f"{planned}  {line:<7} {dest:<24} pl. {quay or '-':<4} {status}")
for trip in root.iterfind(".//o:TripResult/o:Trip", ns):
    print(f"{local(text(trip, 'o:StartTime'))} -> {local(text(trip, 'o:EndTime'))}  {text(trip, 'o:Duration')}  {text(trip, 'o:Transfers')} change(s)")
    for leg in trip.iterfind("o:Leg/o:TimedLeg", ns):
        print(f"   {text(leg, 'o:Service/o:PublishedServiceName/o:Text'):<7} {text(leg, 'o:LegBoard/o:StopPointName/o:Text')} {local(text(leg, 'o:LegBoard/o:ServiceDeparture/o:TimetabledTime'))}"
              f" -> {text(leg, 'o:LegAlight/o:StopPointName/o:Text')} {local(text(leg, 'o:LegAlight/o:ServiceArrival/o:TimetabledTime'))}")
PY
```

If `siri:Status` in the response is `false`, or an `ErrorCondition` is
present, quote its text to the user instead of an empty board. Report what
was asked — the next few departures, the best two connections — with local
times, platforms and the line names as the platform prints them.
