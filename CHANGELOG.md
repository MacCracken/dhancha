# Changelog

All notable changes to dhancha are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [0.1.0] - unreleased

### Added
- Repo scaffolded: pure-Cyrius client-side widget toolkit / desktop app
  framework (Qt/GTK-equivalent) — buildable, link-checkable skeleton
  with the widget-tree / layout / event-loop+input-dispatch / surface
  module surfaces (`src/error.cyr`, `src/widget.cyr`, `src/layout.cyr`,
  `src/event.cyr`, `src/surface.cyr`), the `src/lib.cyr` include chain,
  and `programs/smoke.cyr` link-check. `cyrius = "6.4.7"`, GPL-3.0-only.
  Draw/present cross-deps (sadish + rekha + mabda) are deferred to v0.2.
