# KBase Lakehouse Status

Source for the KBase Lakehouse status page and its machine-readable
announcement feed. Follows the pattern of
[`kbase/status`](https://github.com/kbase/status) — a small Jekyll site on
GitHub Pages, edited directly in the browser — with the addition of a
`status.json` that KBase Lakehouse JupyterLab reads.

It lives outside the platform on purpose. A status page earns its keep exactly
when the platform is down, so it must not depend on anything it reports on.

| | |
|---|---|
| Page | `https://kbasedatalakehouse.github.io/status/` |
| Feed | `https://kbasedatalakehouse.github.io/status/status.json` |

## Publishing an announcement

Edit [`_data/status.yml`](_data/status.yml) and commit. That is the entire
process — it can be done from the GitHub web UI on a phone during an incident.
GitHub Pages rebuilds the page and the feed from that one file.

Add a new entry at the top of the list:

```yaml
announcements:
  - id: 2026-08-02-spark-restart
    message: >-
      Spark clusters restart Saturday 09:00–13:00 PT. Save your work;
      running jobs will be interrupted.
    starts_at: "2026-08-02T09:00:00Z"
    ends_at: "2026-08-02T13:00:00Z"
```

**Quote the timestamps.** Unquoted YAML timestamps are parsed as date objects
and re-serialized into `status.json` without the `T`/`Z`, in a form consumers
cannot read. This is the one way to silently break the feed:

```yaml
ends_at: 2026-08-02T13:00:00Z     # WRONG -> re-serialized as a date
ends_at: "2026-08-02T13:00:00Z"   # RIGHT -> "2026-08-02T13:00:00Z"
```

To retract something early, set `ends_at` to a time in the past — don't delete
the entry. The list is a history; the page renders resolved items as a record.

### Fields

| Field | Required | Meaning |
|---|---|---|
| `id` | yes | Stable and never reused. Dismissal is remembered per id, so editing an entry's text without changing its id will not re-notify anyone who dismissed it. Use a new id to re-notify. |
| `message` | yes | Plain text, shown verbatim in the JupyterLab popup. |
| `starts_at` | no | Quoted ISO 8601. Absent means already started, so an announcement can be committed ahead of time and appear on schedule. |
| `ends_at` | no | Quoted ISO 8601. Absent means no expiry — it shows until given an end time. Exclusive: at `ends_at` exactly, it is over. |

## How it reaches users

JupyterLab's CoreUI extension polls `status.json` through its server extension
and shows a dismissible popup for any announcement inside its window. The
`BERDL_STATUS_URL` environment variable points notebook pods at this site's
base URL; unset means no announcements and no errors.

Two consequences worth knowing:

- **Propagation is bounded by the CDN, not the poll.** GitHub Pages serves
  `cache-control: max-age=600`, so a new announcement can take roughly ten
  minutes plus build time to reach users. Fine for scheduled maintenance,
  mediocre for a live incident — post early.
- **Malformed entries fail toward visible.** An unparseable timestamp leaves an
  announcement showing rather than hiding it. A stuck notice is obvious and
  retractable; a silently suppressed outage notice is not. A structurally
  broken entry is skipped individually so it cannot blank the whole feed.

## Layout

| Path | Role |
|---|---|
| `_data/status.yml` | The only file you edit. Single source for both outputs. |
| `index.html` | Renders the history as a page. |
| `_layouts/default.html` | Page shell and styles. No theme, no external assets. |
| `status.json` | Renders the same data as JSON for machines. |
| `_config.yml` | Site title and description. |

State labels and the banner are computed twice: at build time so the page is
correct without JavaScript, and again in the browser so they are correct at
view time rather than frozen at the last commit.

Built by GitHub Pages' own Jekyll — no Actions, no build step, nothing to break
between editing and publishing.

## Custom domain

The site is currently on the default `github.io` URL. To move it to something
like `status.lakehouse.kbase.us`, add a `CNAME` file and the DNS record, then
update `BERDL_STATUS_URL` in the JupyterHub deployment config. The consumer
reads that variable rather than a hardcoded URL, so the move is a config change.
