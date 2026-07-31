# KBase Lakehouse Status

Status page + machine-readable announcement feed. Follows
[`kbase/status`](https://github.com/kbase/status) — small Jekyll site on GitHub
Pages, edited in the browser — plus a `status.json` that JupyterLab reads.

Lives outside the platform on purpose: a status page earns its keep exactly when
the platform is down, so it must not depend on anything it reports on.

| | |
|---|---|
| Page | `https://kbasedatalakehouse.github.io/status/` |
| Feed | `https://kbasedatalakehouse.github.io/status/status.json` |

## Publishing an announcement

Edit [`_data/status.yml`](_data/status.yml) and commit — that is the whole
process, doable from the GitHub web UI on a phone during an incident. Add new
entries at the top:

```yaml
announcements:
  - id: 2026-08-02-spark-restart
    message: >-
      Spark clusters restart Saturday 09:00–13:00 PT. Save your work;
      running jobs will be interrupted.
    starts_at: "2026-08-02T09:00:00Z"
    ends_at: "2026-08-02T13:00:00Z"
```

**Quote the timestamps.** Unquoted, YAML parses them as dates and re-serializes
into `status.json` without the `T`/`Z`, in a form consumers cannot read. This is
the one way to silently break the feed:

```yaml
ends_at: 2026-08-02T13:00:00Z     # WRONG
ends_at: "2026-08-02T13:00:00Z"   # RIGHT
```

To retract early, set `ends_at` in the past — don't delete. The list is a
history; resolved items stay on the page as a record.

### Fields

| Field | Required | Meaning |
|---|---|---|
| `id` | yes | Stable, never reused. Dismissal is remembered per id, so editing text without changing the id will not re-notify. Use a new id to re-notify. |
| `message` | yes | Plain text, shown verbatim in the JupyterLab popup. |
| `starts_at` | no | Quoted ISO 8601. Absent = already started, so entries can be committed ahead of time. |
| `ends_at` | no | Quoted ISO 8601. Absent = no expiry. Exclusive: at `ends_at` exactly, it is over. |

## How it reaches users

CoreUI polls `status.json` through its server extension and shows a dismissible
popup per active announcement. `BERDL_STATUS_URL` points notebook pods at this
site's base URL; unset means no announcements and no errors.

- **Propagation is bounded by the CDN, not the poll.** GitHub Pages serves
  `cache-control: max-age=600`, so a new announcement takes ~10 min plus build
  time to land. Fine for scheduled maintenance, mediocre for a live incident —
  post early.
- **Malformed entries fail toward visible.** An unparseable timestamp leaves an
  announcement showing: a stuck notice is obvious and retractable, a silently
  suppressed outage notice is not. A structurally broken entry is skipped
  individually so it cannot blank the feed.

## Layout

| Path | Role |
|---|---|
| `_data/status.yml` | The only file you edit. Source for both outputs. |
| `index.html` | Renders the history as a page. |
| `_layouts/default.html` | Page shell and styles. No theme, no external assets. |
| `status.json` | Same data as JSON. |
| `_config.yml` | Title, description, status endpoint, expected service list. |

State labels and the banner are computed twice: at build time so the page works
without JavaScript, and in the browser so they are correct at view time.

Per-service state comes from `GET /hub/status` on JupyterHub — the only
component both publicly reachable and in-cluster. If that fetch fails,
JupyterHub reads Down and the rest Unknown.

Built by GitHub Pages' own Jekyll — no Actions, no build step.

## Custom domain

Currently on the default `github.io` URL. To move it, add a `CNAME` file and the
DNS record, then update `BERDL_STATUS_URL` in the JupyterHub deployment config.
The consumer reads that variable, so the move is a config change.
