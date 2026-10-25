<p align="center"><a href="https://apify.com/DevilScrapes/meetup-group-scraper"><img src="assets/banner.svg" alt="Meetup Group Scraper — Python example" width="100%"></a></p>

# Meetup Group Scraper in Python: event history and RSVPs for the groups you follow

<p align="center">
<a href="https://apify.com/DevilScrapes/meetup-group-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/meetup-group-scraper-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/meetup-group-scraper-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

Most Meetup research starts with a shortlist: the ten user groups in your city, the chapters of one organisation, the communities a competitor sponsors. This repository is a tested Python client for the **[Meetup Group Scraper](https://apify.com/DevilScrapes/meetup-group-scraper)** on Apify that works from exactly that kind of list. Give it group slugs or links and it brings back one record per event with the date and offset, the venue down to latitude and longitude, how many people RSVP'd, any ticket cap, any fee and who organises it. Apify does the fetching; you get `results.json` and `results.csv` on disk, shaped for a DataFrame.

> **Actor:** [Meetup Group Scraper](https://apify.com/DevilScrapes/meetup-group-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [meetup-group-scraper-javascript](https://github.com/devilscrapes/meetup-group-scraper-javascript)

## 🔥 What you get

- **Groups you name, not search luck**: each entry in `groupUrls` can be a full `meetup.com/<group>/` link, a link with `/events/` on the end, or just the bare group slug; the Actor reduces all of them to the slug before fetching.
- **Pick the side of today you care about**: `eventWindow` keeps events still to come, events already held, or both, judged against the moment the run started.
- **Gaps filled from the event page**: when a listing lacks an end time, venue coordinates or a host, the Actor opens that event's own page to complete the row, and skips the extra request when nothing is missing.
- **Counts, not people**: `rsvp_count` is a single total per event; attendee names and profiles never enter the dataset.
- **Optional discovery mode**: leave the group list empty and give `location` and/or `topics` to search Meetup's find page instead; `matched_via` tells you which route produced each row.
- **One row per event across the run**: an event that turns up twice, say from a group link and from a search, is written once, and `maxEvents` caps the run total.
- **Tolerant of hiccups**: throttled or erroring pages are retried up to five times with growing pauses on a fresh proxy session, and a group slug Meetup does not recognise is logged and skipped.

## 🎯 Use cases

### Check whether a community is alive before you sponsor it

Set `eventWindow` to `past`, run the groups on your sponsorship shortlist and count meetings per month. A group that met twice last year with single-digit turnout reads very differently from one that fills a 60-seat room every fortnight, and the numbers are in front of you before anyone signs an invoice.

### Snapshot RSVP momentum on a schedule

Run the script from cron each morning with a dated `--out` folder. Join the snapshots on `event_id` and you can watch how quickly an announced event fills, which tells a DevRel or field team which formats actually pull people in.

### Map where a scene physically meets

`venue_lat` and `venue_lon` drop straight into a notebook scatter plot or a folium map. Clustering a city's tech groups by venue shows which neighbourhoods and co-working spaces host most of the activity.

### Compare free and paid formats

With `is_free`, `fee_amount` and `fee_currency` on every record, a short groupby shows whether charging a few dollars changes turnout for the groups you track.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/meetup-group-scraper-python.git
cd meetup-group-scraper-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [Meetup Group Scraper](https://apify.com/DevilScrapes/meetup-group-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

Stripped of file handling and error reporting, the integration fits in five lines. `call()` waits for the Meetup run to stop, and `iterate_items()` streams the saved events back without loading the whole dataset first.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/meetup-group-scraper").call(run_input={'groupUrls': ['chapter-community'], 'eventWindow': 'upcoming', 'maxEvents': 10, 'includeDescription': True, 'proxyConfiguration': {'useApifyProxy': True}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

`main.py` forwards `input.json` untouched. The sample asks for the upcoming calendar of one public group, Chapter Community, and stops at 10 events, so a first run costs well under a dollar. Swap in the slugs of the communities you follow, and note that `maxEvents` limits the whole run rather than each group.

```json
{
  "groupUrls": [
    "chapter-community"
  ],
  "eventWindow": "upcoming",
  "maxEvents": 10,
  "includeDescription": true,
  "proxyConfiguration": {
    "useApifyProxy": true
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `groupUrls` | array | no | `["chapter-community"]` | — |
| `location` | string | no | — | — |
| `topics` | array | no | — | — |
| `eventWindow` | string | no | `"upcoming"` | `upcoming`, `past`, `both` |
| `maxEvents` | integer | no | `50` | — |
| `includeDescription` | boolean | no | `true` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true}` | — |

## 📤 Output

Each record is one event. `date_time_start` keeps the local offset Meetup shows, `timezone` holds the group's IANA zone, and the venue fields are `null` for online events or when the organiser hides the address. `organizer_name` comes from the group's organiser or the event's first host.

| Field | Type | Always present |
|---|---|---|
| `event_id` | string | yes |
| `event_url` | string | yes |
| `event_name` | string | yes |
| `group_name` | string | yes |
| `group_url` | string | yes |
| `organizer_name` | string or null | no |
| `date_time_start` | string | yes |
| `date_time_end` | string or null | no |
| `timezone` | string or null | no |
| `is_online` | boolean | yes |
| `venue_name` | string or null | no |
| `venue_address` | string or null | no |
| `venue_city` | string or null | no |
| `venue_state` | string or null | no |
| `venue_country` | string or null | no |
| `venue_lat` | number or null | no |
| `venue_lon` | number or null | no |
| `rsvp_count` | integer | yes |
| `max_tickets` | integer or null | no |
| `is_free` | boolean | yes |
| `fee_amount` | number or null | no |
| `fee_currency` | string or null | no |
| `description` | string or null | no |
| `matched_via` | string | yes |
| `scraped_at` | string | yes |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result row emitted | $0.002 |

Prices as configured on 2026-08-20. The [Actor page](https://apify.com/DevilScrapes/meetup-group-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

This pandas snippet (`pip install pandas`) turns the rows into a per-group, per-month cadence table: how many events ran, median RSVPs, median fill rate against the ticket cap, and the share that were paid or online. It writes `group_cadence.csv` for a spreadsheet or a slide.

```python
import json

import pandas as pd

events = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))
events["starts"] = pd.to_datetime(events["date_time_start"], utc=True)
events["month"] = events["starts"].dt.strftime("%Y-%m")
events["fill_rate"] = events["rsvp_count"] / events["max_tickets"]

cadence = (
    events.groupby(["group_name", "month"])
    .agg(
        events=("event_id", "nunique"),
        median_rsvps=("rsvp_count", "median"),
        median_fill=("fill_rate", "median"),
        paid_share=("is_free", lambda free: 1 - free.mean()),
        online_share=("is_online", "mean"),
    )
    .reset_index()
    .sort_values(["group_name", "month"])
)
cadence.to_csv("group_cadence.csv", index=False)
print(cadence.to_string(index=False))
```

## ❓ FAQ

### Is there an official Meetup API I could use instead?

Meetup's own API expects an OAuth client registered with Meetup. This Actor reads the same public group and event pages a logged-out visitor sees, so an Apify token is the only credential involved.

### How far back does the past-events window reach?

It reads what Meetup embeds on the first page of a group's listing, which was 30 upcoming and 10 past events when the Actor was built. For a longer history, schedule regular runs and keep every snapshot.

### What if one of my group slugs is wrong?

That group is skipped with a warning in the run log and the rest continue. If no group or search yields any event at all, the run ends as failed with the message `Scraped 0 events.`, and `main.py` exits with code 1 so cron or Airflow can alert.

### Can I get the list of attendees?

No, and that is deliberate. You get the RSVP total and the capacity, which are enough for turnout analysis without collecting anyone's personal profile.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/meetup-group-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Meetup Group Scraper on Apify Store](https://apify.com/DevilScrapes/meetup-group-scraper)
- [JavaScript version of this example](https://github.com/devilscrapes/meetup-group-scraper-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [Meetup Events Scraper](https://apify.com/DevilScrapes/meetup-events-scraper) — keyword and city searches across all of Meetup when you do not yet know which groups matter.
- [Lu.ma Event Discovery Scraper](https://apify.com/DevilScrapes/luma-event-discovery) — the Luma calendars where many startup and AI communities now publish.
- [Eventbrite Events Scraper](https://apify.com/DevilScrapes/eventbrite-events-scraper) — ticketed workshops and conferences with organizer profiles.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/meetup-group-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Meetup or Meetup.com.
