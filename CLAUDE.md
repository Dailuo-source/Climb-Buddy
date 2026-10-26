# Climb Buddy — Brief

> **Claude: read this first.** Open only the files a change touches; use the code map to jump to a function.
> `web/index.html` is ~102 KB: search for a function name (e.g. `grep -n "function render" web/index.html`) and read just that function.
> At the end of a work session, update **Status** and add one line to **Changelog** (keep about 5).

Gary and Cecilia's climbing log, mainly for **The Hub Mississauga**. Pick a category tile, then result (Sent / Fell, Rested on top rope) and grade; a details panel follows (wall, anchor, hold colour, tries, rests, note, date). Falls become projects; there's a gym map. Samsung Android phones, Chrome PWA.

> **A redesign is in progress.** The approved mockup is in `design/` (six `.dc.html` screens, not published). Follow **Redesign** below, one stage per session. Until a stage is done, the rest of this brief describes the current app.

## Where things live

| Piece | Location |
| --- | --- |
| Live app | https://climbbuddy.pages.dev/ (Cloudflare Pages from `main`, build output directory `web`, so only `web/` is published). Gary wants pages.dev only; the old workers.dev Worker "climbbuddy" (has his name in the URL) can go |
| Database | Google Sheet **Climb Buddy**, id `1AmIjYEFQOTm5U4ZjXuo2UK5LXney1yc8b8zvyANcJu4`. Tabs: Climbs, Projects, Lists |
| Backend | Apps Script bound to the sheet; source is `Code.gs`. Web app version 3 |
| Connecting a phone | Menu **Climb Buddy → Show app key and phone link** → `https://climbbuddy.pages.dev/#api=<exec>&key=<key>` (stored as `cb.api`, `cb.key`); `APP_SITE` in Code.gs builds it |
| Setup guide | `SETUP.md` |

## Data model

- **Climbs**: When, Climber, Type (Bouldering / Top rope / Auto belay / Kilter Board), Grade, Result (Sent / Rested / Fell), Gym, Where, Anchor #, Hold colour, Tries, Rests, Note, Project ID, Video links (Google Photos share links, one per line), Angle (Kilter), Problem name (Kilter).
- **Projects**: Name, Type, Grade, Hold colour, Where, Anchor #, Gym, Note, Status (Open / Sent / Reset), Sent by, Added, Updated. Closes when everyone who tried it has sent it, or when reset. "Last tried" comes only from its climbs' dates; new projects take the climb's date as Added. Never Kilter Board.
- **Notes** (beta notes): Project ID, Climber, When, Note. Each climber's notes are their own; never show one person's notes in the other's view.
- **Lists** (List / Value / Colour / Only at gym): Climber, Gym, Boulder grade (tag colour), Rope grade, Kilter grade (V0–V16), Boulder/Rope/Auto belay area, Anchors (`"Upper: 1-43"`, ranges allowed), Hold colour. Gym-specific rows win over blank-gym rows.
- Code.gs adds a missing column header the first time the app writes that field, so new fields work before **Set up / repair sheets** is run. Run it anyway after deploying: it also adds the Kilter grades.
- The Hub: boulder circuits yellow V0-1, pink V1-2, blue V2-3, green V3-4, orange V4-5, red V5-6, black V6+. Rope grades 5.9, then 5.10- / 5.10+ through 5.13- / 5.13+ (no plain 5.10–5.13). Areas — Bouldering: Cave Left, Cave Right, Comp Wall Left, Comp Wall Right, Learners, Corner North, Corner East, Stage 3. Top rope: Upper, Upper Back, Upper Entrance, Pit, Pit Small Wall. Auto belay: Upper Back, Upper Learners, Upper Entrance. Anchors known: Upper 1–43, Pit 87–109.

## App design rules (`index.html`, one file)

- Tabs: **Climb**, **Projects**, **Progress**, with kawaii tab icons. Climb is internally `session` (`renderSession`): date bar; map card (compact `mapSVG` with `noLabels`/`noNums`, the person's walls glow for the day in their colour, flags on their open projects, beside count, sent/rested/fell bar, hardest, open projects); tapping a wall picks it (`homeSel`, `cb.homesel`, gold) and **Full map ⤢** opens the full map. **Log a climb**: four tiles (`TYPES`, incl. Kilter Board) with the day's counts; the picked wall's category tile is highlighted. **Today's climbs**: `climbRow` per climb (result icon, category, grade, wall/anchor or Kilter angle, hold dot, tries/note, play tile for the first video link), first 5 then Show all; tap to edit.
- Logging: a tile opens `pickSheet` (Result: Sent / Fell, plus Rested for Top rope only; grade chips in rows of 4). Tapping a grade calls `logClimb` (with the picked wall when its type matches) and opens `climbSheet` for the details. Saving shows a toast with Undo (`undoSave`: a new climb is removed, an edit gets its old values back, projects too). Stage 4 replaces both panels with one.
- **Whose view**: one switcher on Climb, Projects and Progress (`whoSeg`, `cb.who`, `cb.both`): a big button per climber with a kawaii avatar (`AVATAR` by theme key) and "N sent today"; the picked one gets the person's colour (`PCOL`: Gary cyan #3DC2EC, Cecilia pink #F7A1C4). **Both** only on Progress (`canBoth`); the full map is always both (`cur()` is `*` there, no switcher, no swipe). Swiping switches with a finger-following animation (`paintSwitch`, commit past 35% or a flick via `animateSwitch`); ignored with a panel open or on sideways-scrolling content. A person's projects = open ones they tried and haven't sent (`openFor`).
- Icons: `catIcon(type)` (Bouldering, Top rope, Auto belay, Kilter Board) and `resIcon(result)` (Sent in `--sent`, Rested deep violet, Fell light purple), SVGs from `design/`. Shown on type buttons, project cards and the day's climb rows.
- **Map**: `HUB_MAP` (1227×1402); each zone's `area` + `type` must match a Lists value. `HUB_MAP.anchorWalls` spreads anchor numbers along each rope wall path (`anchorPoint`), widened by Lists ranges (numbering is Claude's estimate, Gary to confirm). **Full map** (`renderMap`, `tab==="map"`, not in the tab bar; ‹ Climb and phone Back return): one view, no toggle. Walls glow with both climbers' climbs for the period (count chip on each wall), flags mark open projects (`o.flags`, nudged off the count). Opens on Day at the latest session (`mapLand` in `setTab`); Day / Week / Month chips (`cb.mapper`) and ‹ › stepping (`mapPeriod("*")`, `mapStep`): Day steps through session days only, Week is Mon–Sun, Month is calendar. Tapping a wall opens `zonePanel`: sent bar and line for the period, that wall's open projects (tap to edit), **Log a climb here** (`logHere`: picks the wall, back to Climb, opens the picker), **Wall history** (all-time table per person, `wallHist`), This wall was reset.
- **Always dark** (`isDark()` true, `<html data-dark="1">`). Palette: indigo #402E7A, violet #4C3BCF, blue #4B70F5, cyan #3DC2EC. Background tone shows whose stats: teal-navy Gary (`data-who="green"`), plum Cecilia (`"lav"`), midnight indigo Both; also sets `theme-color`. `--me` is the person's colour (cyan / pink); `--sent` and `--accent` follow it. Fell is light purple #CDBFFB everywhere (`--fell`, dark text `--fell-ink`); no coral left. Light tokens (`THEME_LIGHT`, `RES_LIGHT`, `MAP_LIGHT`) are kept in case light returns.
- Charts separate sent / rested / fell by lightness (colour-blind safe): sent mid (Gary #2A9FD0, Cecilia #D9609A, `CHART_SENT` / `--chart-sent`), rested deep violet #5B45D6, fell light #CDBFFB (`RES_DARK`, `--rest-bar`, `--fell-bar`).
- Map in dark: black floor, palette outlines with faint fills (`MAP_DARK`), tapped wall gold, anchor pin cyan.
- Press feedback: `TAP_SEL` dips while held (`.tapping`), then a cyan ring (`.tapped`); `replayTap()` at the end of `render`/`renderSheet`.
- **Back button** (`armBack`, `popstate`): a spare `{cbTrap:1}` entry keeps Back in the app; closes a panel (a just-logged climb's panel acts like Skip details, with Undo), then goes to Climb, then "Press Back again to close" (2.5 s). Chrome skips entries added without a tap, so the trap is re-added on every `pointerdown`.
- Sync: optimistic, batched ops, 60 s polling. Forms in open panels are never redrawn by background sync.
- Icon: kawaii coral chalk bag on sky blue. Keep the maskable icon out of the manifest and `sw.js`.

## Changing things

- App files: edit, bump `CACHE` in `web/sw.js` (currently `climb-buddy-v20`), commit and push to `main`.
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
| `Code.gs` | 523 | `doGet`/`doPost`/`api`; read/write (`readAll_`, `applyOp_`, `table_`, `headerMap_`, `fromCell_`/`toCell_`); menu (`onOpen`, `setup`, `showKey`, `setWebAppUrl`, `resetKey`); tabs (`buildTab_`, `seedLists_`, `paintSwatch_`/`paintAllSwatches_`) |
| `web/index.html` | 1446 | CSS (~15–346), markup (~348–373), one `<script>` (~374–1362) |
| `web/sw.js`, `manifest.webmanifest` | small | offline cache (`CACHE`), PWA manifest |

Inside the `index.html` script, search for:
- Lists and grades: `listRows`, `areaRows`, `gradeIdx`, `gradeColour`, `normType`, `holdHex`, `holdDot`, `tagHTML`
- Sync: `readLinkParams`, `callApi`, `enqueue`, `flush`, `normalize`, `settleDefaults`, `boot`
- Theme and person switch: `themeFor`, `applyTheme`, `whoSeg`, `setWho`, `paintSwitch`, `animateSwitch`, `tapWho`
- Screens: `setTab`, `render`, `onData`
- Climb tab: `renderSession`, `homeSel`, `homeWall`, `climbRow`, `videoLinks`, `hardest`, `pickSheet`, `logClimb`, `undoSave`
- Projects: `renderProjects`, `projCard`, `projInfo`, `triedBy`, `openFor`, `markProjectSent`
- Map: `HUB_MAP`, `anchorRanges`, `areaForAnchor`, `anchorPoint`, `mapSVG`, `miniMap`, `renderMap`, `mapLand`, `mapPeriod`, `mapStep`, `setMapPer`, `zonePanel`, `FLAG_SVG`
- Progress: `renderProgress`, `gradeCounts`
- Back button: `armBack`, `trapped`, `exitArmed`
- Panels: `openSheet`, `renderSheet`, `climbSheet`, `saveClimb`, `projSheet`, `daySheet`, `gymSheet`, `areaField`, `holdField`
- Tap feedback: `TAP_SEL`, `tapKey`, `replayTap`, `scrollsSideways`

## Redesign (approved 2026-10-24, mockup in `design/`)

Gary's problems with the current app: logging is clunky and hard to fix; projects are hard to track or edit; the map is useful but stuck on its own tab, and its Activity/Projects toggle splits things; his and Cecilia's views look alike; nobody uses **Both**.

**Decisions**
- **Tabs:** Climb (home), Projects, Progress. The Map tab is removed; its code moves into Climb plus a full-screen map panel.
- **Person switch:** two big buttons with kawaii avatars (Gary: chalk bag, cyan `#3DC2EC`; Cecilia: hold with a bow, pink `#F7A1C4`). The selected one is filled in that person's colour; their colour carries through the page (keep the per-person background tones). **Both** is removed from Climb and Projects but **stays on Progress**.
- **Climb (home)** (`design/Main.dc.html`): date bar; map card (compact `mapSVG` beside today's stats: count, sent/rested/fell bar, hardest, open projects); tapping a wall selects it (gold) and the wall fills in when logging; **Full map ⤢** opens the full map. Then **Log a climb**: four tiles (Boulder, Top rope, Auto belay, Kilter Board) with counts for the day; the selected wall's category tile is highlighted. The per-grade ✓/Rest/✕ rows are gone. Then **Today's climbs** (kept): one row per climb with result badge, category icon, grade, wall/anchor, hold dot, tries/note, and a play tile when it has a video. Tapping a row opens it for editing. Toast after save: Undo.
- **Log / edit a climb** (`design/Edit-climb.dc.html`): one panel for new and existing climbs. Category (4 icon buttons), Result (Sent / Rested / Fell; **Rested only for Top rope**), Grade chips in rows of 4 (ropes include 5.12− / 5.12+), wall + anchor + mini map (Kilter: angle and problem name instead), hold colour, tries stepper, who climbed it, Google Photos video links, project link, note, date, Delete / Save. **Everything is editable**, including grade, result, category and climber.
- **Selected state must be obvious**: the chosen result / grade / status is solid, bright, outlined in white, with a tick badge; the rest are plain outlines at reduced opacity.
- **Result colours:** Sent cyan (Gary) or pink (Cecilia), Rested deep violet, **Fell light purple `#CDBFFB`** (replaces coral everywhere, including the Progress charts; keep chart colours separated by lightness). Bubbly round icons for all three: white circle, `#1E2235` outline, rounded thick ✓ / ✕ / pause bars.
- **Category icons** (kawaii, Karate-Ka Companion style: white fill, `#1E2235` outline, dot eyes, pink blush, small smile): boulder (rock with warm hold spots), top rope (hold on a cyan rope), auto belay (box with a lavender strap), Kilter Board (tilted board with four coloured holds). Used on the home tiles, the log panel, Today's climbs, project cards and project pages. Exact SVGs are in `design/`.
- **Projects** (`design/Projects.dc.html`): heading "<Name>'s projects" + **+ New** (add a project without falling first); filter chips All / Boulder / Rope (no Kilter); cards with hold-colour strip, category icon, grade, name, where, a dot per try, last note, last tried; a big chevron, centred vertically on the right. "Sent recently" list with Reopen.
- **Project page** (`design/Project.dc.html`): back, person, ✎ Edit; hero (hold colour, grade, name, category icon, where, status, mini map with pin); stats (tries, sessions, rests last time); quick log Fell / Rested / Sent it!; **Beta notes** (this climber only, add a note inline); **Every try** (each try, tap to edit, video tile); Route was reset / Delete project.
- **Edit project** (`design/Edit-project.dc.html`): name, status (Open / Sent / Route reset), category (no Kilter), grade, wall + anchor + map, hold colour. Tries keep their own details.
- **Full map** (`design/Map.dc.html`): one view, no Activity/Projects toggle: walls glow with climbs for the Day / Week / Month period, flags mark open projects, always both. Legend uses the same flag pin as the map. Tapping a wall opens a panel: climbs and sent bar for the period, that wall's projects, **Log a climb here**, Wall history. Kilter lives in the Boards & Training area.
- **Videos:** links only, nothing uploaded. Paste a Google Photos share link (several allowed) on a climb; the row shows a play tile that opens the link. Also a **share target**: in Google Photos, Share → Climb Buddy opens the app with the link and asks which climb to attach it to (`share_target` in the manifest, GET with `url`/`text` params; pick from recent climbs, default the latest).

**Build plan (one stage per session)**
1. ~~Backend~~: Code.gs columns (Video links, Angle, Problem name, project Name), Notes tab, Kilter Board type and grades. *Done 2026-10-24; Gary deploys with clasp and runs Set up / repair sheets.*
2. ~~Shell + person switch + icons~~: 3 tabs, new person switcher (no Both outside Progress), kawaii category and result icons, Fell colour change. *Done 2026-10-25; the map stays reachable from Full map ⤢ on Climb (with Both) until stage 3.*
3. ~~Climb home~~: map card on home (wall tap selects), Log a climb tiles, Today's climbs list; remove grade rows and the Map tab; full-map panel. *Done 2026-10-25; tiles open a stop-gap result + grade picker, then the old details panel, until stage 4.*
4. **Log / edit panel:** the new climb panel (all fields editable, Rested only for Top rope, Kilter fields, video links).
5. **Projects:** list redesign, + New, project page, edit project, beta notes (Notes tab).
6. **Video share target** (manifest + handler), polish, update this brief (design rules, code map) and remove the redesign notice.

## Status
- Redesign: stages 1–3 (backend; shell, switcher, icons; Climb home and full map) done; next is stage 4 (log / edit panel).
- Working: Climb home with map card and tiles, logging with details and Undo, projects, full map with anchors and Day/Week/Month, Progress, per-person views with swipe, always-dark theme.
- Open: Gary to confirm anchor numbering and add the remaining anchors (whiteboard hints: Upper 49–53, Pit 77/80/83); remove the old workers.dev Worker.
- Ideas: speed climb or Boards & Training sessions (map labels for now).

## Changelog (latest first, keep about 5 lines)
- 2026-10-25: Redesign stage 3: Climb home (map card, Log a climb tiles with result + grade picker, Today's climbs, Undo on save); full map is always both with flags and a wall panel; grade rows removed
- 2026-10-25: Redesign stage 2: Climb / Projects / Progress tabs, avatar person switcher (Both only on Progress), kawaii category and result icons, Fell light purple
- 2026-10-24: Redesign approved (`design/`); backend for Kilter Board, video links, project names and beta notes
- 2026-10-22: Map Activity by Day / Week / Month with ‹ › stepping; Map opens on Activity at the latest session
- 2026-10-21: Moved the site files into `web/`; only that folder is published

## Working efficiently
- Find code with search first, then read only the relevant lines. Don't read index.html top to bottom.
- Make small, targeted edits. Never regenerate a whole file.
- Don't open a browser to test unless I ask. I'll test on my phone.
- No subagents unless I ask.
- One task per session. I'll /clear between tasks.
