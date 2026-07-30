---
layout: default
---

{% assign entries = site.data.status.announcements %}
{% assign now = site.time | date: '%s' | plus: 0 %}

{% if entries == nil or entries.size == 0 %}

No announcements. The KBase Lakehouse is operating normally.

{% else %}
{% for a in entries %}
{% assign starts = nil %}
{% if a.starts_at and a.starts_at != '' %}{% assign starts = a.starts_at | date: '%s' | plus: 0 %}{% endif %}
{% assign ends = nil %}
{% if a.ends_at and a.ends_at != '' %}{% assign ends = a.ends_at | date: '%s' | plus: 0 %}{% endif %}
{% assign state = 'Current' %}
{% if starts and now < starts %}{% assign state = 'Scheduled' %}{% endif %}
{% if ends and now >= ends %}{% assign state = 'Resolved' %}{% endif %}
<div class="announcement" data-starts-at="{{ a.starts_at }}" data-ends-at="{{ a.ends_at }}">
<h3 class="announcement-state">{{ state }}</h3>
<p>{{ a.message }}</p>
<p class="announcement-window">{% if starts and ends %}{{ a.starts_at | date: '%d %b %Y, %H:%M UTC' }} &mdash; {{ a.ends_at | date: '%d %b %Y, %H:%M UTC' }}{% elsif ends %}Until {{ a.ends_at | date: '%d %b %Y, %H:%M UTC' }}{% elsif starts %}From {{ a.starts_at | date: '%d %b %Y, %H:%M UTC' }}{% else %}No end time set{% endif %}</p>
</div>
<hr>
{% endfor %}
{% endif %}

<p id="build-note">Current / Scheduled / Resolved below were computed when this
page was last built, {{ site.time | date: '%d %b %Y, %H:%M UTC' }}, and may have
moved on since. The <a href="status.json">machine-readable feed</a> carries the
raw times.</p>

<script>
// Recompute the state labels against the reader's clock. Liquid can only
// evaluate them at build time, which goes stale as the build ages; this makes
// them true at view time. The build-time labels remain as the no-JS fallback,
// so the page is never blank or wrong-by-omission.
(function () {
  var announcements = document.querySelectorAll('.announcement');
  if (!announcements.length) {
    return;
  }
  var now = Date.now();
  announcements.forEach(function (el) {
    var starts = Date.parse(el.dataset.startsAt || '');
    var ends = Date.parse(el.dataset.endsAt || '');
    // An absent or unparseable bound means unbounded on that side, matching
    // how consumers of status.json treat it: fail toward visible.
    var state = 'Current';
    if (!isNaN(starts) && now < starts) {
      state = 'Scheduled';
    }
    if (!isNaN(ends) && now >= ends) {
      state = 'Resolved';
    }
    var heading = el.querySelector('.announcement-state');
    if (heading) {
      heading.textContent = state;
    }
  });
  // The labels are now live, so the staleness caveat no longer applies.
  var note = document.getElementById('build-note');
  if (note) {
    note.remove();
  }
})();
</script>

For KBase-wide outages beyond the Lakehouse, see
[status.kbase.us](https://status.kbase.us/).
