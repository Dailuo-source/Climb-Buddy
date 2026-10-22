# Climb Buddy — Brief

> **Claude: read this first.** Open only the files a change touches; use the code map to jump to a function.
> `web/index.html` is ~102 KB: search for a function name (e.g. `grep -n "function render" web/index.html`) and read just that function.
> At the end of a work session, update **Status** and add one line to **Changelog** (keep about 5).

Gary and Cecilia's climbing log, mainly for **The Hub Mississauga**. Tap ✓ / Rest / ✕ per grade to log sends, finishes with rests (ropes only) and falls; each tap opens a details panel (wall, anchor, hold colour, tries, rests, note, date). Falls become projects; there's a gym map. Samsung Android phones, Chrome PWA.

## Where things live

| Piece | Location |
| --- | --- |
| Live app | https://climbbuddy.pages.dev/ (Cloudflare Pages from `main`, build output directory `web`, so only `web/` is published). Gary wants pages.dev only; the old workers.dev Worker "climbbuddy" (has his name in the URL) can go |
| Database | Google Sheet **Climb Buddy**, id `1AmIjYEFQOTm5U4ZjXuo2UK5LXney1yc8b8zvyANcJu4`. Tabs: Climbs, Projects, Lists |
| Backend | Apps Script bound to the sheet; source is `Code.gs`. Web app version 3 |
| Connecting a phone | Menu **Climb Buddy → Show app key and phone link** → `https://climbbuddy.pages.dev/#api=<exec>&key=<key>` (stored as `cb.api`, `cb.key`); `APP_SITE` in Code.gs builds it |
| Setup guide | `SETUP.md` |

## Data model

- **Climbs**: When, Climber, Type (Bouldering / Top rope / Auto belay), Grade, Result (Sent / Rested / Fell), Gym, Where, Anchor #, Hold colour, Tries, Rests, Note, Project ID.
- **Projects**: Type, Grade, Hold colour, Where, Anchor #, Gym, Note, Status (Open / Sent / Reset), Sent by, Added, Updated. Closes when everyone who tried it has sent it, or when reset. "Last tried" comes only from its climbs' dates; new projects take the climb's date as Added.
- **Lists** (List / Value / Colour / Only at gym): Climber, Gym, Boulder grade (tag colour), Rope grade, Boulder/Rope/Auto belay area, Anchors (`"Upper: 1-43"`, ranges allowed), Hold colour. Gym-specific rows win over blank-gym rows.
- The Hub: boulder circuits yellow V0-1, pink V1-2, blue V2-3, green V3-4, orange V4-5, red V5-6, black V6+. Rope grades 5.9, then 5.10- / 5.10+ through 5.13- / 5.13+ (no plain 5.10–5.13). Areas — Bouldering: Cave Left, Cave Right, Comp Wall Left, Comp Wall Right, Learners, Corner North, Corner East, Stage 3. Top rope: Upper, Upper Back, Upper Entrance, Pit, Pit Small Wall. Auto belay: Upper Back, Upper Learners, Upper Entrance. Anchors known: Upper 1–43, Pit 87–109.

## App design rules (`index.html`, one file)

- Tabs: **Session** (date picker, `daySummary` drawn as a ring + stacked bar per type, since Gary prefers visuals; tapping a type switches the grade list; per-grade ✓/Rest/✕), **Projects**, **Map**, **Progress**.
- **Whose view**: one switcher on every tab (`whoSeg`, `cb.who`, `cb.both`). Map, Projects, Progress also offer **Both**; Session logs for one person. Swiping switches with a finger-following animation (`paintSwitch`, commit past 35% or a flick via `animateSwitch`); ignored with a panel open or on sideways-scrolling content. A person's projects = open ones they tried and haven't sent (`openFor`).
- The switcher buttons must stay transparent (`.seg.wseg button`); a plain `.wseg button` rule loses to `.seg button[aria-pressed]`.
- **Map**: `HUB_MAP` (1227×1402); each zone's `area` + `type` must match a Lists value. `HUB_MAP.anchorWalls` spreads anchor numbers along each rope wall path (`anchorPoint`), widened by Lists ranges (numbering is Claude's estimate, Gary to confirm).
- **Always dark** (`isDark()` true, `<html data-dark="1">`). Palette: indigo #402E7A, violet #4C3BCF, blue #4B70F5, cyan #3DC2EC. Background tone shows whose stats: teal-navy Gary (`data-who="green"`), plum Cecilia (`"lav"`), midnight indigo Both; also sets `theme-color`. ✕ buttons and fell counts stay coral. Light tokens (`THEME_LIGHT`, `RES_LIGHT`, `MAP_LIGHT`) are kept in case light returns.
- Charts use only blue/purple shades separated by brightness (colour-blind safe): sent #6FD6F6, rested #5B7BF3, fell #6236C9 (`RES_DARK`, `--chart-sent`, `--rest-bar`, `--fell-bar`).
- Map in dark: black floor, palette outlines with faint fills (`MAP_DARK`), tapped wall gold, anchor pin cyan.
- Press feedback: `TAP_SEL` dips while held (`.tapping`), then a cyan ring (`.tapped`); `replayTap()` at the end of `render`/`renderSheet`.
- **Back button** (`armBack`, `popstate`): a spare `{cbTrap:1}` entry keeps Back in the app; closes a panel (a just-logged climb's panel acts like Skip details, with Undo), then goes to Session, then "Press Back again to close" (2.5 s). Chrome skips entries added without a tap, so the trap is re-added on every `pointerdown`.
- Sync: optimistic, batched ops, 60 s polling. Forms in open panels are never redrawn by background sync.
- Icon: kawaii coral chalk bag on sky blue. Keep the maskable icon out of the manifest and `sw.js`.

## Changing things

- App files: edit, bump `CACHE` in `web/sw.js` (currently `climb-buddy-v17`), commit and push to `main`.
- Code.gs: paste into Apps Script and redeploy (**Deploy → Manage deployments → pencil → New version**). Via Claude in Chrome: sheet → Extensions → Apps Script, set `monaco.editor.getModels()[0]` to the raw GitHub Code.gs, Ctrl+S, redeploy. New columns must also be added to the live sheet.
- Sheet edits via the Google Sheets connector are fine; insert rows in Lists rather than rewriting blocks (colour cells are painted).
- Commits use author `Claude <noreply@anthropic.com>`. Push to `https://github.com/Dailuo-source/Climb-Buddy.git`.

## Deploying Code.gs

- Web app deployment ID: `AKfycbyZJS90k7Zpis9_o34t4vWSLGaIU6TUgyeSylhCwXLSuR8NK2ATQOEudUofVRXhlERI`. Always update this deployment so the /exec URL never changes.
- From this folder, `.clasp.json` points clasp at the script project (rootDir is the repo root, where `Code.gs` is):
  ```
  clasp push
  clasp deploy -i AKfycbyZJS90k7Zpis9_o34t4vWSLGaIU6TUgyeSylhCwXLSuR8NK2ATQOEudUofVRXhlERI -d "what changed"
  ```
- Never run `clasp deploy` without `-i`: that creates a new deployment with a new /exec URL. `clasp deployments` lists them.

## Code map
API: op-batched like Pantry Pal (`{key, ops: [{op: add|update|del, coll, id, data}]}` → whole state).

| File | Lines | What's in it |
|---|---|---|
| `Code.gs` | 488 | `doGet`/`doPost`/`api`; read/write (`readAll_`, `applyOp_`, `table_`, `headerMap_`, `fromCell_`/`toCell_`); menu (`onOpen`, `setup`, `showKey`, `setWebAppUrl`, `resetKey`); tabs (`buildTab_`, `seedLists_`, `paintSwatch_`/`paintAllSwatches_`) |
| `web/index.html` | 1281 | CSS (~15–339), markup (~341–366), one `<script>` (~367–1279) |
| `web/sw.js`, `manifest.webmanifest` | small | offline cache (`CACHE`), PWA manifest |

Inside the `index.html` script, search for:
- Lists and grades: `listRows`, `areaRows`, `gradeIdx`, `gradeColour`, `normType`, `holdHex`, `holdDot`, `tagHTML`
- Sync: `readLinkParams`, `callApi`, `enqueue`, `flush`, `normalize`, `settleDefaults`, `boot`
- Theme and person switch: `themeFor`, `applyTheme`, `whoSeg`, `setWho`, `paintSwitch`, `animateSwitch`, `tapWho`
- Screens: `setTab`, `render`, `onData`
- Session tab: `renderSession`, `daySummary`, `hardest`, `logClimb`
- Projects: `renderProjects`, `projCard`, `projInfo`, `triedBy`, `openFor`, `markProjectSent`
- Map: `HUB_MAP`, `anchorRanges`, `areaForAnchor`, `anchorPoint`, `mapSVG`, `miniMap`, `renderMap`, `zonePanel`
- Progress: `renderProgress`, `gradeCounts`
- Back button: `armBack`, `trapped`, `exitArmed`
- Panels: `openSheet`, `renderSheet`, `climbSheet`, `saveClimb`, `projSheet`, `daySheet`, `gymSheet`, `areaField`, `holdField`
- Tap feedback: `TAP_SEL`, `tapKey`, `replayTap`, `scrollsSideways`

## Status
- Working: logging with details, projects, Hub map with anchors, Progress, per-person views with swipe, always-dark theme.
- Open: Gary to confirm anchor numbering and add the remaining anchors (whiteboard hints: Upper 49–53, Pit 77/80/83); remove the old workers.dev Worker.
- Ideas: speed climb or Boards & Training sessions (map labels for now).

## Changelog (latest first, keep about 5 lines)
- 2026-10-21: Moved the site files into `web/`; only that folder is published
- 2026-10-22: Trimmed this brief
- 2026-10-18: Phone Back button stays in the app; double Back to exit
- 2026-10-18: New kawaii app icon (chalk bag on sky blue)
- 2026-10-17: Always-dark theme in Gary's palette, black map, blue/purple charts, press feedback

## Working efficiently
- Find code with search first, then read only the relevant lines. Don't read index.html top to bottom.
- Make small, targeted edits. Never regenerate a whole file.
- Don't open a browser to test unless I ask. I'll test on my phone.
- No subagents unless I ask.
- One task per session. I'll /clear between tasks.
