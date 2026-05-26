# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Static viewer that plays per-climber segments of IFSC World Cup broadcasts via the YouTube IFrame API. Deployed to GitHub Pages, served as-is — no build step, no bundler, no backend.

This is the **display half** of a two-repo workflow. The producer is `../ifsc-climbs-studio` (a Python CLI that downloads broadcasts and OCRs the countdown clock + name plates). The studio project is responsible for writing finalized, verified segment data into this repository's `events/` tree; index.html only reads it.

## Structure

- `index.html` — the entire app. Single file, vanilla JS, no dependencies beyond the YouTube IFrame API loaded at runtime.
- `events/index.json` — manifest of all available events, written by `ifsc-climbs-studio`. Drives the event picker dropdown.
- `events/<event-slug>/segments.json` — segment data placed here by `ifsc-climbs-studio`. Each event lives in its own subdirectory; `stream.mp4` and `name_debug/` from the studio's working dir are **not** copied over.

The viewer discovers events purely from `events/index.json` at runtime — no event list is hardcoded in `index.html`. The selected event slug is reflected in the `?event=<slug>` query string. Events are sorted newest-first by `upload_date`, with `title` as the tiebreaker. Climb labels: event slugs starting with `mens-` get M1, M2…; everything else gets W1, W2….

## Data contract with ifsc-climbs-studio

The viewer reads two files: `events/index.json` (manifest) and `events/<slug>/segments.json` (per-event data).

`events/index.json`:

```
{
  events: [
    { slug: string, title: string, upload_date: "YYYY-MM-DD" },
    ...
  ]
}
```

`slug` must match the directory name under `events/`. `upload_date` is used as the primary sort key (descending); `title` is the tiebreaker and the dropdown label.

`segments.json` (current, `schema_version: 2`):

```
{
  schema_version: 2,
  video_id: string,                              // YouTube ID, fed to YT.Player
  athletes: [{ name: string, country: string|null }],
  segments: [{ start: number, end: number|null, athletes: number[] }]   // athletes = indices into athletes[]
}
```

`segments[].end` may be `null` for the last segment (and is currently `null` for all segments in v2 dumps — the viewer only reads `start`). v1 used `countries: string[]` and could carry optional `url` / `clock_regions` / `clock_region` metadata; the viewer no longer looks at any of those.

**If you change this schema, update `segment.py` in `ifsc-climbs-studio` in lockstep** — the studio writes it and the viewer reads it, with no shared validation layer. Same for the directory naming convention under `output/`.

## Deployment

GitHub Pages serves the repo root, so `index.html` and `events/**/segments.json` must both be committed and reachable as static assets. The YouTube IFrame API is loaded from `https://www.youtube.com/iframe_api` at runtime — no API key, but the page will not work offline.

## Conventions

- Keep the viewer dependency-free and single-file unless there's a strong reason — the deployment story is "commit and push," and adding a build step breaks that.
- All timestamps in `segments.json` are float seconds; `YT.Player.seekTo(seconds, true)` consumes them directly.
