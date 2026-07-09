# llm-traffic-order-to-geodata-pipeline

![Python](https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white)
![OpenAI GPT-4o](https://img.shields.io/badge/OpenAI-GPT--4o%20Vision-412991?logo=openai&logoColor=white)
![GeoPandas](https://img.shields.io/badge/GeoPandas-black)
![Shapely](https://img.shields.io/badge/Shapely-black)
![NetworkX](https://img.shields.io/badge/NetworkX-black)
![Folium](https://img.shields.io/badge/Folium-black)
![PyMuPDF](https://img.shields.io/badge/PyMuPDF-black)
![Pandas](https://img.shields.io/badge/Pandas-150458?logo=pandas&logoColor=white)
![QGIS](https://img.shields.io/badge/QGIS-black?logo=qgis&logoColor=white)
![Jupyter](https://img.shields.io/badge/Jupyter-F37626?logo=jupyter&logoColor=white)

A pipeline that reads scanned UK traffic regulation orders (legal PDFs with no shared format
between councils) and turns them into structured records and mapped street geometry.

## What a traffic regulation order actually is, and why it's a hard data problem

Every parking restriction, loading ban, or waiting limit on a UK street exists because a local
council published and sealed a legal PDF called a Traffic Regulation Order (TRO). That PDF is
the actual source of truth. Nothing downstream (a parking app, a curb management platform, a
council's own GIS system) has the restriction until someone turns that PDF into structured data.

That turns out to be a genuinely annoying problem to automate:

- Most TROs are scans or have a text layer that mixes underlined street names, tabular data, and
  legal prose in ways that break normal PDF text extraction.
- Every council writes its own version. Same legal concept, different layout, different wording.
- The street geometry is written as prose, not coordinates. A real line from the order I used:
  "From the extended line of the northeast kerb of Stirling Road eastwards for a distance of 58
  metres or thereby." That is precise enough for a surveyor and useless to a database.

The order I built this against was 33 pages, 18 schedules, about 120 individual restrictions.
Doing that by hand means someone reading the whole thing and typing out every street, every
restriction type, every distance, and then figuring out how to standardize it. I built this
because I kept running into that same problem in transportation and curb data work: the
regulation always exists as a PDF somewhere, and every project re-solves "how do we get this
into a database" from scratch.

## Why this matters

The order I built this against runs 33 pages and 18 schedules, and produces roughly 120
individual street restrictions once fully parsed. Doing that by hand means someone reading the
whole document and typing out every street, restriction type, and distance themselves, then
standardizing the wording so it fits a database. That is realistically most of a working day for
one document, and a single council publishes dozens of these over time, with more added or
amended every year.

This pipeline runs the same document from PDF to structured, geocoded output in a fraction of
that time, most of it unattended processing rather than someone sitting at a keyboard
transcribing. What is left afterward is a short list of rows to glance at and confirm, not a
document to retype from scratch. For a municipality trying to digitize its own regulations, or a
company offering that as a service to councils, the difference is not just convenience. It
changes how many orders one analyst can realistically get through, which changes what is
possible to offer and at what price.

## How it works

### Step one: there is no text to extract, so treat every page as an image

The first thing I ruled out was text extraction. It does not work reliably on these documents,
scanned or not. So every page gets rendered to an image and sent to a vision model (GPT-4o),
the same way a person would actually read it.

### Step two: figuring out where one schedule ends and the next begins

Once you are reading pages as images, a new problem shows up. A 33 page order is not one
document. It is roughly 18 independently numbered Schedules, each with its own restriction type
(no waiting, limited waiting, loading bans, and so on), and the streets listed under a Schedule
only make sense in the context of that Schedule's rules. Send pages to the model one at a time
or all at once and you lose that structure.

So there is a separate first pass that scans every page and asks the model whether it is the
start of a new Schedule, then stitches the results into page ranges, one per Schedule. This is
where the first real bug showed up. The model would occasionally flag a schedule number as a
heading when it was actually just mentioned inside a paragraph of legal text, something like
"as specified in Schedule VI" buried in the order's boilerplate. Since TROs always number their
schedules in order starting at I with no gaps, I added a plain sequence check on top of the
model's output: a detected heading only counts if its number is exactly one higher than the last
confirmed schedule. That single rule caught every false positive I tested against without
needing anyone to manually mark where the real content starts.

### Step three: extract each schedule as a batch

With the page ranges known, each Schedule's full set of pages goes to the model together in one
call, using a prompt that encodes the actual parsing rules for this document (how an underlined
title maps to an area, how several restrictions under one street become separate rows, how "31
metres or thereby" becomes the number 31). Batching by schedule instead of by page means the
model always has full context for the rule it is applying, and the output is structured CSV: 
area, schedule, street, side of street, length, plus the raw segment text and the raw
restriction text for traceability back to the source.

### Step four: clean it, and only flag what actually needs a person

Model output at this stage is mostly right but not perfectly consistent. A row here or there
drops a field or shifts a comma. Rather than trust every row blindly, the cleaning step checks
field counts against the expected schema, and where a malformed pattern is common and
predictable (a specific class of "revoked in its entirety" rows kept dropping one empty field in
a consistent way), it repairs the row deterministically instead of just logging a warning and
moving on. What is left after cleaning, forward filling schedule and area context down empty
rows, and merging in document metadata (title, effective date, penalties) is a short list of
rows that genuinely need a human glance, not a 120 row document someone would otherwise
transcribe by hand.

### Step five: turning prose into coordinates

A structured row is still not something a GIS system can use. "Airdriehill Street, north side,
58 metres from Stirling Road" is not a geometry yet. The second half of the pipeline turns that
description into an actual line segment, using OS Open Roads (Ordnance Survey's free national
road network) as ground truth.

The text gets parsed into a reference: which street the distance is measured from, which compass
direction, how far. When the order gives two reference streets instead of a direction ("from X
to Y, a distance of N metres"), the parser uses that instead, since tracing directly between two
known junctions is more reliable than doing direction math. From there, the pipeline finds the
junction where the restricted street meets its reference street using the road network's own
topology (two streets meet wherever their line segments share an endpoint node), then walks the
stated distance from that point, crossing multiple linked road segments if needed, since OS
splits long streets into many short pieces. At each junction it picks whichever next segment
best matches the compass bearing it is supposed to be heading in. Output is GeoJSON and an
interactive map, colored by restriction schedule.

## Getting sample data

This repo does not include a PDF, since TROs are council specific and licensing varies, but real
ones are free to download.

Recommended source: the [Transport Appeals TRO Library](https://tro.transportappeals.scot/authority_tro/?authority=North%20Lanarkshire%20Council),
a standing public archive of Scottish councils' traffic regulation orders. North Lanarkshire
Council alone has 9 scanned orders in this exact format.

| Ref | Order | Year |
|---|---|---|
| NL001 | [Airdrie Area](https://tro.transportappeals.scot/TRO/North%20Lanarkshire%20Council/20182f01-North-Lanarkshire-Council-Airdrie-Area-Traffic-Regulation-Consolidation-Order-2018_.pdf) | 2018 |
| NL002 | [Bellshill Area](https://tro.transportappeals.scot/TRO/North%20Lanarkshire%20Council/20182f02-North-Lanarkshire-Council-Bellshill-Area-Traffic-Regulation-Consolidation-Order-2018_.pdf) | 2018 |
| NL003 | [Coatbridge Area](https://tro.transportappeals.scot/TRO/North%20Lanarkshire%20Council/20182f03-North-Lanarkshire-Council-Coatbridge-Area-Traffic-Regulation-Consolidation-Order-2018_.pdf) | 2018 |
| NL004 | [Cumbernauld and Condorrat](https://tro.transportappeals.scot/TRO/North%20Lanarkshire%20Council/20182f04-North-Lanarkshire-Council-Cumbernauld-Condorrat-Traffic-Regulation-Order-2018_.pdf) | 2018 |
| NL005 | [Kilsyth Area](https://tro.transportappeals.scot/TRO/North%20Lanarkshire%20Council/20182f05-North-Lanarkshire-Council-Kilsyth-Area-Traffic-Regulation-Consolidation-Order-2018_.pdf) | 2018 |
| NL006 | [Motherwell and Ravenscraig Area](https://tro.transportappeals.scot/TRO/North%20Lanarkshire%20Council/2018_06_North_Lanarkshire_Council_Motherwell_and_Ravenscraig_Area_Traffic_Regulation_Consolidation_Order_2018.pdf) | 2018 |
| NL007 | [Various Streets, Shotts Area](https://tro.transportappeals.scot/TRO/North%20Lanarkshire%20Council/20182f07-North-Lanarkshire-Council-Various-Streets-Shotts-Area-Waiting-and-Laoding-Restrictions-Order-2018_.pdf) | 2018 |
| NL008 | [Wishaw Town Centre](https://tro.transportappeals.scot/TRO/North%20Lanarkshire%20Council/20182f08-North-Lanarkshire-Council-Wishaw-Town-Centre-Wishaw-Traffic-Management-Order-2014_.pdf) | 2014 |
| NL009 | [Various Locations, Wishaw](https://tro.transportappeals.scot/TRO/North%20Lanarkshire%20Council/20182f09-North-Lanarkshire-Council-Various-Locations-Wishaw-Traffic-Regulation-Variation-Order-2018_.pdf) | 2018 |

These are genuinely scanned, non text extractable PDFs, the real constraint this pipeline is
built to handle. Download NL001 (Airdrie Area), save it as `Inputs/NL001_airdrie_area_2018.pdf`,
and you are ready to run the first notebook.

Other sources worth knowing about if you want to extend beyond Scotland:
- [DfT Find Transport Data, TRO dataset](https://findtransportdata.dft.gov.uk/dataset/traffic-regulation-orders-177f4f31b7a) (England and Wales, links out to individual councils)
- [Brighton and Hove current TROs](https://www.brighton-hove.gov.uk/travel-and-road-safety/roads-and-highways/current-traffic-regulation-orders)
- [North East Lincolnshire TROs](https://www.nelincs.gov.uk/streets-travel-and-parking/traffic-and-road-safety/traffic-regulation-orders/)

## Pipeline

| Notebook | What it does |
|---|---|
| `notebooks/01_pdf_to_csv_extraction.ipynb` | PDF to per page images, schedule detection, bylaw row extraction, cleaned and metadata enriched CSV |
| `notebooks/02_geocoding_segments.ipynb` | CSV street descriptions parsed into direction and distance references, geocoded against OS Open Roads, exported as GeoJSON and an interactive Folium map |

## Setup

```bash
pip install pymupdf pillow openai python-dotenv pandas geopandas shapely networkx folium pyproj
```

Create a `.env` file in the project root:

```
OPENAI_API_KEY=your-key-here
```

### Road network data, for notebook 2

Download OS Open Roads (free, official UK road network) from the
[OS Data Hub](https://osdatahub.os.uk/downloads/open/OpenRoads), GeoPackage format, the tile
covering North Lanarkshire (National Grid square NS), and save it to
`Geodata/os_open_roads_north_lanarkshire.gpkg`. This file is large. See the notebook for how to
clip a whole of Great Britain download down to just the area you need.

## Project structure

```
Inputs/            source TRO PDFs, not committed
Geodata/           OS Open Roads GeoPackage, not committed
Processed_Bylaws/  raw and cleaned per schedule CSV output
Schedules/         page to schedule detection logs
Final/             final structured CSV, geocoded segments as GeoJSON, and the Folium map per order
notebooks/         the pipeline itself
```
