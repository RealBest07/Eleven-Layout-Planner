# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Read This First

**Read `work.md` before touching code.** It is a hand-maintained context cache (architecture, function index, layout-object schema, coordinate system, zone-drawing order, changelog) that lets you skip re-reading the ~1030-line source. After any change, update the Changelog table in `work.md`.

## Project

Single-file web app: `index.html`. Vanilla HTML/CSS/JS + Canvas 2D. No build step, no dependencies, no tests, no package manager.

A 7-Eleven store layout planner: user enters plot/store/setback/parking dimensions, app computes a zoned site plan and renders it to `<canvas>`. UI text is Thai.

## Running

Open `index.html` directly in a browser, or use the configured preview server (see `.claude/launch.json`). There is no compile/lint/test toolchain — changes are verified by reloading the page and inspecting the canvas.

## Architecture

Pipeline is `generate()` → builds a plain layout object `L` → `draw(L)` → renders. All `draw*` helpers take `(ctx, cx, cy, cs, L)` and are pure functions of `L`.

Key invariants (full detail in `work.md`):
- **Coordinate flip**: real `y=0` is the road (canvas bottom), `y=plotD` is the back (canvas top). `cy(y) = OY + (plotD - y) * SC` does the flip; `cx`/`cs` scale only.
- **Zones stack bottom→top**: Free Space → ระยะถอย (rearClear) → ช่องจอด (parking slots) → ทางเดิน (storeClear) → ร้าน (store). `freeSpace` absorbs leftover depth so `rearClear` stays exactly equal to its input.
- **Plot clipping**: `draw()` calls `ctx.save()/clip()` to the plot rect before drawing zones, then `ctx.restore()` before drawing the plot boundary, labels, road, and dimensions (which render outside the plot). Don't move that `restore()`.
- **Store overflow**: when the store exceeds the plot, it is drawn dimmed with a "⚠ ร้านเกินขนาดที่ดิน" label and an error bar — not hidden.

## Conventions

- `aisleW = 6` is hardcoded in `generate()`, not a user input.
- Store always faces the road (south/bottom); there is no facing-direction input (removed). `sbFront` was also removed — only `sbBack`/`sbLeft`/`sbRight` exist.
- When adding a user input: add the DOM element in the left `.panel`, read it in `generate()`, thread it into `L`, then consume it in the relevant `draw*` helper.
