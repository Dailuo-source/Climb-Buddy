# Climb Buddy — Brief

> **Claude: read this first.** Open only the files a change touches, and use the code map below to jump to a function instead of reading whole files.
> `index.html` is about 102 KB (~26k tokens): don't read it whole; search for a function name (e.g. `grep -n "function render" index.html`) and read just that function.
> At the end of a work session, update **Status** and add one line to **Changelog** (keep about 10).

Gary and Cecilia's climbing log, mainly for **The Hub Mississauga**. Tap ✓ / Rest / ✕ per grade to log sends, finishes with rests (ropes only) and falls. Every tap opens a details panel (wall, anchor, hold colour, tries, rests, note, date), routes they fall on become projects, and there's a map of the gym. Used on their Samsung Android phones (Chrome PWA).

## Where things live

| Piece | Location |
| --- | --- |
| Live app | https://climbbuddy.pages.dev/ (Cloudflare Pages from `main` of `Dailuo-source/Climb-Buddy`). There's also an old Cloudflare **Worker** named climbbuddy (workers.dev, has Gary's name in the URL); Gary wants pages.dev only |
| Database | Google Sheet **Climb Buddy**, id `1AmIjYEFQOTm5U4ZjXuo2UK5LXney1yc8b8zvyANcJu4`. Tabs: Climbs, Projects, Lists |
| Backend | Apps Script bound to the sheet; source is `Code.gs` here. Web app deployment version 3 is live |
| Connecting a phone | Sheet menu **Climb Buddy → Show app key and phone link** gives `https://climbbuddy.pages.dev/#api=<exec>&key=<key>`; the app stores both in localStorage (`cb.api`, `cb.key`). `APP_SITE` in Code.gs builds that link |
| Setup guide | `SETUP.md` |

## Data model

- **Climbs**: one row per climb: When, Climber, Type (Bouldering / Top rope / Auto belay), Grade, Result (Sent / Rested / Fell), Gym, Where, Anchor #, Hold colour, Tries, Rests, Note, Project ID.
- **Projects**: Type, Grade, Hold colour, Where, Anchor #, Gym, Note, Status (Open / Sent / Reset), Sent by (list), Added, Updated. A project closes when everyone who tried it has sent it, or when it's marked reset.
- **Lists** (List / Value / Colour / Only at gym): Climber (Gary, Cecilia), Gym, Boulder grade with tag colour, Rope grade, Boulder area, Rope area, Auto belay area, Anchors (`"Upper: 1-43"`, comma-separated ranges allowed), Hold colour. Rows with a gym only apply at that gym; for each list, gym-specific rows win over blank-gym rows.
- The Hub's boulder circuits: yellow V0-1, pink V1-2, blue V2-3, green V3-4, orange V4-5, red V5-6, black V6+. Rope grades: 5.9, then 5.10- / 5.10+ through 5.13- / 5.13+ (no plain 5.10–5.13).
- Hub areas. Bouldering: Cave Left, Cave Right, Comp Wall Left, Comp Wall Right, Learners, Corner North, Corner East, Stage 3. Top rope: Upper, Upper Back, Upper Entrance, Pit, Pit Small Wall. Auto belay: Upper Back, Upper Learners, Upper Entrance. Known anchors: Upper 1–43, Pit 87–109; Gary will add the rest (whiteboard hints: Upper 49–53, Pit 77/80/83).

## App structure (`index.html`, one file)

- Tabs: **Session** (date picker, day summary card `daySummary`, drawn rather than numbers (Gary prefers visuals): a ring of sent / rested / fell with the total in the middle and a legend with counts and send rate, then a stacked bar per climb type (Bouldering, Top rope, Auto belay) on one shared scale with counts inside the segments and that type's top send; tapping a type row switches the grade list below. Chart colours `RES_HEX` (#1F4FB8 / #6C8CEB / #DD6B26) are deeper than the UI's soft fills and pass the colour-blind/contrast validator; then per-grade ✓/Rest/✕), **Projects**, **Map**, **Progress** (month view).
- **Whose view**: one person switcher at the top of every tab (`whoSeg`), shared across tabs (`WHO` in localStorage `cb.who`; `BOTH` flag `cb.both`). Map, Projects and Progress also offer **Both**; Session always logs for one person. Swiping left/right anywhere switches (cycles Gary → Cecilia → Both, wraps): the screen follows the finger (`paintSwitch`): the new tint sweeps in as a soft gradient on body + header, the black highlight (`.wseg .pill`, positioned by fractional `--i`) glides between names, and the page drifts; past 35% or a quick flick it commits (`animateSwitch`), otherwise springs back. Taps animate the same way. Ignored while a panel is open or on anything that scrolls sideways (zoomed map, long chart). A person's projects = open ones they've tried and not yet sent (`openFor`). "This wall was reset" still resets everyone's projects on that wall.
- **Map**: `HUB_MAP` traces Gary's floor plan (1227×1402 coordinates); each zone's `area` + `type` must match a Lists value. Projects view shows dots in hold colour; Activity view shades walls by climbs in the last 30 days; tapping a wall opens stats, its projects and "This wall was reset". Details panels include a mini map to tap the wall, and typing an anchor number picks the wall from the Anchors ranges and pins its estimated spot. `HUB_MAP.anchorWalls` holds each rope wall's face as a path plus a number span (Pit 77→109 north→south down the west face; Upper 1→53 from the south end up the east face then west along the north edge — Claude's estimate, Gary to confirm); numbers are spread evenly along the path (`anchorPoint`), widened by any Anchors ranges in Lists. Map tab projects with an anchor sit at that spot instead of the wall's cluster.
- "Last tried" on a project comes only from its climbs' dates (Added is just a fallback), and new projects take the climb's date as Added, so backfilled sessions show the right day.
- Sync: same pattern as Pantry Pal (optimistic, batched ops, 60 s polling). Forms in open panels are never redrawn by a background sync.
- **Always dark** (Gary's choice; `isDark()` returns true, `<html data-dark="1">` so there's no light flash). Built on Gary's palette: indigo #402E7A, violet #4C3BCF, blue #4B70F5, cyan #3DC2EC. Background tone shows whose stats you're viewing: deep teal-navy for Gary (`data-who="green"`), deep plum for Cecilia (`"lav"`), midnight indigo for Both; also sets the status-bar `theme-color`. Name highlight is a cyan→blue gradient; selected chips/pills violet; primary buttons blue; Sent buttons cyan; ✕ buttons and small "fell" counts stay coral. The light tokens (`THEME_LIGHT`, `RES_LIGHT`, `MAP_LIGHT`) are still in the file if light ever comes back.
- Overview charts (Session ring + per-type bars, Progress "Climbs per session") use only blue/purple shades, separated by brightness so they pass the colour-blind checks: sent #6FD6F6, rested #5B7BF3, fell #6236C9 (`RES_DARK`, `--chart-sent`, `--rest-bar`, `--fell-bar`).
- Map in dark: black floor, walls as palette-coloured outlines with faint fills (`MAP_DARK`: violet boulders, blue top rope, cyan auto belay), near-white labels, tapped wall gold, anchor pin cyan.
- Press feedback: anything tappable (`TAP_SEL`) dips and brightens while held (`.tapping`), then pulses a cyan ring (`.tapped`). Taps usually redraw the screen, so `replayTap()` (end of `render`/`renderSheet`) finds the same button again by its data-* attributes plus index and plays the ring there.
- The name switcher's buttons must stay transparent (`.seg.wseg button`) so the gliding highlight shows; a plain `.wseg button` rule loses to `.seg button[aria-pressed]`.
- Icon: simple side-on grey rock with a face climbing an orange rope, feet on a cream wall, on lavender. Gary deleted the maskable icon; keep it out of the manifest and `sw.js`.

## Changing things

- App files: edit, bump `CACHE` in `sw.js` (currently `climb-buddy-v15`), commit and push to `main`. Cloudflare deploys in about a minute.
- Code.gs: paste into Apps Script and redeploy (**Deploy → Manage deployments → pencil → New version**; URL stays the same). Claude did this through Claude in Chrome last time: open the sheet → Extensions → Apps Script, set `monaco.editor.getModels()[0]` to the raw GitHub Code.gs, Ctrl+S, redeploy. A new sheet column also needs adding to the live sheet (the header is matched by name).
- Sheet edits through the Google Sheets connector are fine. Insert rows rather than rewriting blocks in Lists, because the colour cells are painted.
- Commits use author `Claude <noreply@anthropic.com>`; GitHub rejects Gary's private email. Push with `https://github.com/Dailuo-source/Climb-Buddy.git`.
- Testing: a Node mock of SpreadsheetApp plus Playwright (route `script.google.com` to `api()`) worked well for end-to-end checks.

## Ideas mentioned but not built

- Tracking speed climb or Boards & Training sessions (they're labels on the map for now).

## Code map
API: op-batched like Pantry Pal (`{key, ops: [{op: add|update|del, coll, id, data}]}` → whole state).

| File | Lines | What's in it |
|---|---|---|
| `Code.gs` | 488 | `doGet`/`doPost`/`api`; read/write (`readAll_`, `applyOp_`, `table_`, `headerMap_`, `fromCell_`/`toCell_`); sheet menu (`onOpen`, `setup`, `showKey`, `setWebAppUrl`, `resetKey`); tabs (`buildTab_`, `seedLists_`, `paintSwatch_`/`paintAllSwatches_` for Lists colours) |
| `index.html` | 1281 | CSS (lines ~15–339), markup (~341–366), one `<script>` (~367–1279) |
| `sw.js`, `manifest.webmanifest` | small | offline cache (`CACHE`), PWA manifest |

Inside the `index.html` script, search for:
- Lists and grades: `listRows`, `areaRows`, `gradeIdx`, `gradeColour`, `normType`, `holdHex`, `holdDot`, `tagHTML`
- Sync: `readLinkParams`, `callApi`, `enqueue`, `flush`, `normalize`, `settleDefaults`, `boot`
- Theme and person switch: `themeFor`, `applyTheme`, `whoSeg`, `setWho`, `paintSwitch`, `animateSwitch`, `tapWho`
- Screens: `setTab`, `render`, `onData`
- Session tab: `renderSession`, `daySummary`, `hardest`, `logClimb`
- Projects: `renderProjects`, `projCard`, `projInfo`, `triedBy`, `openFor`, `markProjectSent`
- Map: `HUB_MAP`, `anchorRanges`, `areaForAnchor`, `anchorPoint`, `mapSVG`, `miniMap`, `renderMap`, `zonePanel`
- Progress: `renderProgress`, `gradeCounts`
- Panels: `openSheet`, `renderSheet`, `climbSheet`, `saveClimb`, `projSheet`, `daySheet`, `gymSheet`, `areaField`, `holdField`
- Tap feedback: `TAP_SEL`, `tapKey`, `replayTap`, `scrollsSideways`

## Status
- Working: logging (✓ / Rest / ✕ with details), projects, Hub map with anchors, Progress, per-person views with swipe, always-dark theme.
- Open: Gary to confirm the anchor numbering estimate and add the remaining anchors; the old workers.dev Worker (has Gary's name in the URL) can be removed, since he wants pages.dev only.
- Ideas: speed climb or Boards & Training sessions (labels on the map for now).

## Changelog (latest first, keep about 10 lines)
- 2026-10-18: Turned the handoff notes into a brief (read-first rules, code map, status, changelog)
- 2026-10-17: Always-dark theme in Gary's palette, black map, blue/purple charts, press feedback
- 2026-10-17: Session summary as a chart: day ring plus stacked bar per climb type
- 2026-10-16: Session summary: Total row plus a row per climb type with top send
- 2026-10-16: Session summary: total row with mix bar, plus Sent/Rested/Fell rows with top grades
- 2026-10-16: Fix last-tried dates, finger-following swipe, anchor spots on the map
- 2026-10-16: Per-person views on every tab, swipe to switch, green/lavender tint
