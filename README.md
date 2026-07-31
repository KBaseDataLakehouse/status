# KBase Lakehouse Status

Status page + machine-readable feed. Follows
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
history; ended entries collapse into a "N resolved" section on the page.

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

## Two axes, kept apart

Announcements say what is *planned*; the service list says what is *broken*.
Conflating them makes a scheduled-upgrade notice read as an outage.

- **Banner colour is service health alone** — green/yellow/red, never touched by
  announcements. No probe configured, or the probe not yet answered: neutral,
  never green. Active announcements appear under it as a plain count.
- **Announcement lifecycle** (Current / Scheduled / Resolved) is blue and
  neutral, outside the health scale.
- **Unknown is neutral**, not a severity rung.

## Layout

| Path | Role |
|---|---|
| `_data/status.yml` | The only file you edit: `announcements` + `services_url`. |
| `index.html` | Page markup and script. |
| `_layouts/default.html` | Shell and styles. No theme, no external assets. |
| `status.json` | `_data/status.yml` verbatim as JSON. |
| `_config.yml` | Title and description. |

Announcement states and the open/resolved split are computed twice: in Liquid at
build time so the page works without JavaScript, and again in the browser so
labels track the reader's clock rather than the last commit. An entry whose
window opened or closed since the build moves between the two lists on load.

Per-service rows are built entirely in the browser from `services_url` —
`GET /hub/status` on JupyterHub, the only component both publicly reachable and
in-cluster. There is no static service list to keep in sync; names from the last
good response are cached in `localStorage`. If the fetch fails, JupyterHub reads
Down (it serves that endpoint) and every remembered service Unknown.

Built by GitHub Pages' own Jekyll — no Actions, no build step.

## Custom domain

Currently on the default `github.io` URL. To move it, add a `CNAME` file and the
DNS record, then update `BERDL_STATUS_URL` in the JupyterHub deployment config.
The consumer reads that variable, so the move is a config change.
