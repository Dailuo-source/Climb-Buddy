# Climb Buddy — Brief

> **Claude: read this first.** Open only the files a change touches; use the code map to jump to a function.
> `web/index.html` is ~102 KB: search for a function name (e.g. `grep -n "function render" web/index.html`) and read just that function.
> At the end of a work session, update **Status** and add one line to **Changelog** (keep about 5).

Gary and Cecilia's climbing log, mainly for **The Hub Mississauga**. Pick a category tile and fill in one panel (result, grade, wall, anchor, hold colour, tries, rests, who, video links, note, date); tap any climb to change any of it. Falls become projects; there's a gym map. Samsung Android phones, Chrome PWA.

## Where things live

| Piece | Location |
| --- | --- |
| Live app | https://climbbuddy.pages.dev/ (Cloudflare Pages from `main`, build output directory `web`, so only `web/` is published). Gary wants pages.dev only; the old workers.dev Worker "climbbuddy" (has his name in the URL) can go |
| Database | Google Sheet **Climb Buddy**, id `1AmIjYEFQOTm5U4ZjXuo2UK5LXney1yc8b8zvyANcJu4`. Tabs: Climbs, Projects, Notes, Lists |
| Backend | Apps Script bound to the sheet; source is `Code.gs`. Web app version 3 |
| Connecting a phone | Menu **Climb Buddy → Show app key and phone link** → `https://climbbuddy.pages.dev/#api=<exec>&key=<key>` (stored as `cb.api`, `cb.key`); `APP_SITE` in Code.gs builds it |
| Setup guide | `SETUP.md` |

## Data model

- **Climbs**: When, Climber, Type (Bouldering / Top rope / Auto belay / Kilter Board), Grade, Result (Sent / Rested / Fell), Gym, Where, Anchor #, Hold colour, Tries, Rests, Note, Project ID, Video links (Google Photos share links, one per line), Angle (Kilter), Problem name (Kilter).
- **Projects**: Name, Added by (`owner`: who made it with + New; a project nobody has tried yet shows only for them), Type, Grade, Hold colour, Where, Anchor #, Gym, Note, Status (Open / Sent / Reset), Sent by, Added, Updated. Closes when everyone who tried it has sent it, or when reset. "Last tried" comes only from its climbs' dates; new projects take the climb's date as Added. Never Kilter Board.
- **Notes** (beta notes): Project ID, Climber, When, Note. Each climber's notes are their own; never show one person's notes in the other's view.
- **Lists** (List / Value / Colour / Only at gym): Climber, Gym, Boulder grade (tag colour), Rope grade, Kilter grade (V0–V16), Boulder/Rope/Auto belay area, Anchors (`"Upper: 1-43"`, ranges allowed), Hold colour. Gym-specific rows win over blank-gym rows.
- Code.gs adds a missing column header the first time the app writes that field, so new fields work before **Set up / repair sheets** is run. Run it anyway after deploying: it also adds the Kilter grades.
- The Hub: boulder circuits yellow V0-1, pink V1-2, blue V2-3, green V3-4, orange V4-5, red V5-6, black V6+. Rope grades 5.9, then 5.10- / 5.10+ through 5.13- / 5.13+ (no plain 5.10–5.13). Areas — Bouldering: Cave Left, Cave Right, Comp Wall Left, Comp Wall Right, Learners, Corner North, Corner East, Stage 3. Top rope: Upper, Upper Back, Upper Entrance, Pit, Pit Small Wall. Auto belay: Upper Back, Upper Learners, Upper Entrance. Anchors: Upper 1–43, Pit 75–87 and 92–109, Pit Small Wall 71–74 and 88–91 (Gary gave some numbers; the rest are spread along the walls).

## App design rules (`index.html`, one file)

- Tabs: **Climb**, **Projects**, **Progress**, with kawaii tab icons. Climb is internally `session` (`renderSession`): date bar; map card (compact `mapSVG` with `noLabels`, the person's walls glow for the day in their colour with a count bubble each (`numFill`/`numInk`/`numR`); no project flags here, beside count, sent/rested/fell bar, hardest, open projects); tapping a wall picks it (`homeSel`, `cb.homesel`, gold) and **Full map ⤢** opens the full map. **Log row** (2026-10-31): "Log a climb" text on the left, a compact **+** button with the four category icons on the right (`data-act="logNew"`: opens the panel for the picked wall's category, else the last used `TYPE`); with a wall picked it reads "Log a climb at <Wall>". **Today's climbs**: badge cards (`climbCard`: result-coloured left edge, `idBadges` grade · holds (Kilter: board + angle) · wall map, result icon + where, then time · tries · rests · project · note; play tile for a video), with **Show** (All + only the categories climbed that day; `dayFilter`, `cb.dayf`) and **Sort** Recent / Grade / Type + Hardest/Easiest (`daySort`, `dayHard`, `cb.days`, `cb.dayh`); Recent shows 8 then Show all.
- **Log / edit panel** (`climbSheet`, sheet type `climb`; compact since 2026-10-31, `.lsheet` on the sheet): header has the title, a who toggle (`.lwho`) and a date chip (`.dchip`, the date input `#fDay` sits invisibly on top); category row and result row are short (38 / 36px); grades are one sideways-scrolling row (`gradeFill` colours, ropes on the ramp); wall = small map (`miniSVG(..., small)`, `.minimap.wm`) beside wall pills + Anchor #; hold colours one row; Tries / Rests (Rested only) / Video on one row (video inputs appear below once added); Note is a single input; **Cancel / Save stay pinned** (sticky `.foot`), Save says "Save to project", "Save · project sent!" or "Save + add to Projects". Kilter: board-angle buttons 20–50° + problem name instead of wall and holds. Older notes on this panel: one panel for new (`newClimb(type, area)`, from a tile or Log a climb here, with the picked wall) and existing climbs (`editClimb(c)`, from any climb row). Category (4 buttons), Result (Sent / Fell; Rested only for Top rope, or an older climb already saved as Rested), grade chips in rows of 4, wall: mini map + area + anchor (Kilter: Angle and Problem name instead), hold colour, tries / rests steppers, who climbed it, Google Photos video links (several; `video` column, one per line), project link (`projMatches`/`matchList`: open projects at the gym with the same category and grade as `idBadges` cards, best match first (wall, holds, anchor, then yours), labelled "Probably a repeat" / "Same grade and wall" / …; a new climb whose grade, wall and holds all match is linked for you (`d.autoLink`) until you pick something else (`d.noAuto`); the panel redraws when the wall, holds or anchor change; or "Add to Projects" for a fall; never Kilter), note, date, Delete or Cancel / Save. Everything is editable. The form lives in a draft, `sheet.d`: `readForm()` copies typed fields in before any button (`data-act="cf"` → `setField`) redraws the panel. Changing category clears a grade, wall or project that doesn't fit. A new climb is stored only on Save (`saveClimb`); Back or Cancel drops it. No grade → "Pick a grade first". Save shows Undo (`undoSave`: a new climb is removed, an edit gets its old values back, projects too).
- Selected state: the chosen result and grade are solid with a white outline and a tick badge, the others fade; the chosen category uses the person's colour.
- **Whose view**: one switcher on Climb, Projects and Progress (`whoSeg`, `cb.who`, `cb.both`): a big button per climber with a kawaii avatar (`AVATAR` by theme key) and "N sent today"; the picked one gets the person's colour (`PCOL`: Gary cyan #3DC2EC, Cecilia pink #F7A1C4). **Both** only on Progress (`canBoth`); the full map shows the picked person (no switcher, no swipe). Swiping switches with a finger-following animation (`paintSwitch`, commit past 35% or a flick via `animateSwitch`); ignored with a panel open or on sideways-scrolling content. A person's projects = open ones they tried and haven't sent (`openFor`).
- **Projects tab** (`renderProjects`, redone 2026-10-31): one person (no Both); "<Name>'s projects" + **+ New**; **Projects map** first (`projMap`): walls with projects glow, each project is a tag in its grade colour with a hold-colour dot inside (`marksSVG` `hd`; dashed ring when no colour), routes with an anchor also get a dot at the anchor; tapping a wall or tag picks that wall (`projWall`, `projSel`, gold; other tags fade) and the list narrows, with a gold "Wall <Wall> · N projects ×" chip. Under the map: **Show** chips All / Boulder / Top rope / Auto (`PF`, `projFilter`, `cb.pf`; also filters the map) and **Sort** Recent / Grade / Type (`PSORT`, `projSort`, `cb.psort`) with a **Hardest / Easiest** flip (`projHard`, `cb.phard`) when not Recent. Grade groups by grade (swatch in the grade colour, boulders then ropes); Type groups Boulder / Top rope / Auto belay, hardest first inside. Gary asked for no Wall or Holds sort (the map is the wall filter). Cards (`projCard`, `.pc3`, ~60px): three badges from `idBadges` (grade tile in `gradeFill`: boulder circuit colour, ropes a light-to-dark `ROPE_RAMP`; a kawaii hold `holdBlob` in the hold colour with its name; `wallThumb` mini gym map with the wall lit and the anchor pinned), then wall (+ #anchor, name first if set), a dot per try and the date, chevron. Beta notes are not on the cards (they're on the project page). A **twin** (`twinOf`/`sameRoute`: same category, grade and wall, same holds and anchor when both set, not Reset) gets an orange "<Name> has one just like this · Merge?" strip; "Sent or reset recently" below.
- **Merge** (`mergeSheet`, sheet type `merge`, `doMerge`): pick which to keep (radio `keep`); all climbs and notes move to it, Sent by is combined, status recomputed, empty name/holds/anchor/note filled from the other, the other is deleted; Undo restores everything. **They're different** stores the pair in `cb.nottwins` (this phone only).
- **Project page** (`renderProject`, `tab==="proj"`, `projId` / `cb.projid`; not in the tab bar, Projects stays highlighted; phone Back returns to Projects; no swipe): ‹ Projects, the person, ✎ Edit; hero with hold ball, grade, name, category + where, status pill, mini map with the anchor pin (`act:"noop"`); stats (tries, sessions, rests last time or last tried); quick log Fell / Rested (Top rope only) / Sent it! (`projLog`, for WHO on DAY, Undo); **Beta notes** (`myNotes(p, WHO)`, Notes tab, add inline with `noteForm`, × deletes with Undo; a half-typed note survives sync redraws); **Every try** (this person's climbs on it as `climbRow`, tap to edit); Route was reset / Reopen, Delete project (also removes its notes; climbs stay).
- **Edit / New project panel** (`editProject(p|null)`, `projSheet`, sheet type `project`, draft in `sheet.d`, `readProjForm` / `setProjField` via `data-act="pjf"`, `saveProject`): name, status (Open / Sent / Route reset; edit only: Sent adds WHO to Sent by, Open again takes them off), category (3, no Kilter), grade chips, mini map + wall + anchor, hold colour; Delete or Cancel / Save with Undo. A new project gets `owner: WHO` and opens its page. New only: if an open project looks the same (`projDups`, `sameRoute`) a "You're already projecting this one" box (`dupWarning`) offers **Open that one** or **No, a new route** (`d.dupOk`); Save waits for an answer.
- **Progress map** (`progressMap`, above the charts, follows the month and Gary / Cecilia / Both): walls shaded by sends that month, a tag with the hardest send per wall (grade colour for boulders), a gold ★ when it beats everything sent on that wall before the month (only if there were earlier sends there); tap a wall (`progWall`, `progSel`) for its line: sends, hardest, new best, rested, falls.
- `marksSVG` keeps tags inside the gym and nudges overlapping ones down.
- **Videos:** links only, nothing uploaded. Climbs keep Google Photos share links (`video`, one per line); rows show a play tile that opens the first. **Share → Climb Buddy**: the manifest's `share_target` (GET `./?title&text&url`) opens the app; `readShare` (in `boot`) pulls the first link out and clears the address; after the first sync `showShare` opens the `share` sheet (`shareSheet`): the 12 latest climbs at this gym, the picked person's latest preselected; **Add to this climb** (`attachShare`, Undo) or **Log a new climb** (new climb panel with the link filled in). `sw.js` doesn't cache addresses with a query.
- **"Didn't finish"**: the sheet still stores `Fell`; the app shows it as "Didn't finish" everywhere (`resLabel`, `RES_KEYS`, result buttons, quick log, Progress legends and tables).
- **After-save pop-up** (`showCheer`, `#cheer`): a new climb (and a project-page quick log) gets a pop-up with a kawaii animation (`CHEER_ART`: star + confetti, teacup with steam, wobbling rain cloud, trophy) and a cheeky British line from `CHEER` (sent 30, rested 20, dnf 30, proj 12; `cheerKind`). Each kind is a shuffled deck on the phone (`cb.deck.<kind>`), so no repeat until it runs out. Undo and "Ta very much"; hides after 5 s. Editing a climb keeps the plain toast.
- Icons: `catIcon(type)` (Bouldering, Top rope, Auto belay, Kilter Board) and `resIcon(result)` (Sent in `--sent`, Rested deep violet, Fell light purple), SVGs from `design/`. Shown on type buttons, project cards and the day's climb rows.
- **Map**: `HUB_MAP` (1227×1402); each zone's `area` + `type` must match a Lists value. Traced from Gary's floor plan of 2026-10-25 (image px → map: x·0.945, y·0.946, see the anchor comment). `HUB_MAP.anchorRuns` holds the anchor numbers where Gary drew them, along each wall face; `anchorPoint` finds a number on its run and spreads in-between numbers along the wall (area-agnostic). `anchorAreas` says which wall a number belongs to when Lists has no Anchors row (Upper 1–43; Pit Small Wall 71–74 and 88–91; Pit 75–87 and 92–109; the Lists Anchors rows say the same and win). Speed climb is Pit Small Wall 71–74 (label only). The **bridge** joining the small wall to the Pit is overhead (you walk under it): drawn as a dashed feature, not a wall; its anchors (75–82, 92–98) count as Pit. Anchor numbers are not printed on the map; only a picked anchor shows as a pin in the log panel's mini map. Kilter Board is a zone (`any:true`: every Kilter climb counts there, no anchors or projects); Training is a plain feature. **Full map** (`renderMap`, `tab==="map"`, not in the tab bar; ‹ Climb and phone Back return): one session for the picked person (`mapSession`): their climbs on the day the Climb tab is on (`DAY`), walls glow in their colour with a count bubble each; **no project flags, no Week/Month** (Gary's call). ‹ › step through that person's session days and move `DAY` with them. Tapping a wall opens `zonePanel`: sent bar and line for the session, that wall's open projects (tap to edit), **Log a climb here** (`logHere`: picks the wall, back to Climb, opens the picker), **Wall history** (all-time table per person, `wallHist`), This wall was reset.
- **Always dark** (`isDark()` true, `<html data-dark="1">`). Palette: indigo #402E7A, violet #4C3BCF, blue #4B70F5, cyan #3DC2EC. Background tone shows whose stats: teal-navy Gary (`data-who="green"`), plum Cecilia (`"lav"`), midnight indigo Both; also sets `theme-color`. `--me` is the person's colour (cyan / pink); `--sent` and `--accent` follow it. Fell is light purple #CDBFFB everywhere (`--fell`, dark text `--fell-ink`); no coral left. Light tokens (`THEME_LIGHT`, `RES_LIGHT`, `MAP_LIGHT`) are kept in case light returns.
- Charts separate sent / rested / fell by lightness (colour-blind safe): sent mid (Gary #2A9FD0, Cecilia #D9609A, `CHART_SENT` / `--chart-sent`), rested deep violet #5B45D6, fell light #CDBFFB (`RES_DARK`, `--rest-bar`, `--fell-bar`).
- Map in dark: black floor, palette outlines with faint fills (`MAP_DARK`), tapped wall gold, anchor pin cyan.
- Press feedback: `TAP_SEL` dips while held (`.tapping`), then a cyan ring (`.tapped`); `replayTap()` at the end of `render`/`renderSheet`.
- **Back button** (`popstate`, `reconcileBack`, `navDepth`, `backTokens`): each Back goes up one level (closes a panel, project page → Projects, any tab → Climb, then "Press Back again to close" and the next Back leaves). History entries are only Back tokens `{cbTrap:n}`; the app keeps one more token than levels deep. **Chrome skips entries added without a tap** (that skipped page made Back jump to an older load of the app), so a token is only added in a click/keydown listener (one per tap, checked with `navigator.userActivation`), never from `popstate` or `boot`; extra tokens left when something closes in code are dropped with `history.go` (`dropPops` ignores that pop).
- Sync: optimistic, batched ops, 60 s polling. Forms in open panels are never redrawn by background sync.
- Icon: kawaii coral chalk bag on sky blue. Keep the maskable icon out of the manifest and `sw.js`.

## Changing things

- App files: edit, bump `CACHE` in `web/sw.js` (currently `climb-buddy-v31`), commit and push to `main`.
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
| `Code.gs` | 524 | `doGet`/`doPost`/`api`; read/write (`readAll_`, `applyOp_`, `table_`, `headerMap_`, `fromCell_`/`toCell_`); menu (`onOpen`, `setup`, `showKey`, `setWebAppUrl`, `resetKey`); tabs (`buildTab_`, `seedLists_`, `paintSwatch_`/`paintAllSwatches_`) |
| `web/index.html` | 1744 | CSS, then the markup, then one `<script>` (search for names, line numbers drift) |
| `web/sw.js`, `manifest.webmanifest` | small | offline cache (`CACHE`), PWA manifest |

Inside the `index.html` script, search for:
- Lists and grades: `listRows`, `areaRows`, `gradeIdx`, `gradeColour`, `normType`, `holdHex`, `holdDot`, `tagHTML`
- Sync: `readLinkParams`, `callApi`, `enqueue`, `flush`, `normalize`, `settleDefaults`, `boot`
- Theme and person switch: `themeFor`, `applyTheme`, `whoSeg`, `setWho`, `paintSwitch`, `animateSwitch`, `tapWho`
- Screens: `setTab`, `render`, `onData`
- Climb tab: `renderSession`, `homeSel`, `homeWall`, `climbCard`, `dayFilter`, `daySort`, `showCheer`, `CHEER`, `resLabel`, `climbRow`, `videoLinks`, `hardest`
- Projects: `renderProjects`, `PF`, `PSORT`, `idBadges`, `gradeFill`, `holdBlob`, `wallThumb`, `twinOf`, `sameRoute`, `mergeSheet`, `doMerge`, `projMatches`, `matchList`, `projDups`, `dupWarning`, `projCard`, `renderProject`, `projLog`, `myNotes`, `projInfo`, `triedBy`, `openFor`, `markProjectSent`, `editProject`, `projSheet`, `saveProject`
- Map: `HUB_MAP`, `marksSVG`, `projMap`, `progressMap`, `anchorRanges`, `areaForAnchor`, `anchorPoint`, `mapSVG`, `miniMap`, `renderMap`, `mapLand`, `mapSession`, `mapStep`, `zonePanel`, `FLAG_SVG`
- Progress: `renderProgress`, `gradeCounts`
- Back button: `reconcileBack`, `navDepth`, `backTokens`, `dropPops`
- Share: `readShare`, `showShare`, `shareSheet`, `attachShare`, `recentClimbs`
- Panels: `openSheet`, `renderSheet`, `newClimb`, `editClimb`, `climbSheet`, `readForm`, `setField`, `linkProject`, `saveClimb`, `undoSave`, `projSheet`, `daySheet`, `gymSheet`, `areaField`, `holdField`
- Tap feedback: `TAP_SEL`, `tapKey`, `replayTap`, `scrollsSideways`

## Redesign (done 2026-10-26)

The October 2026 redesign is finished; the rules above describe it. The approved mockup is in `design/` (six `.dc.html` screens, not published) and holds the exact icon SVGs. It fixed what Gary found clunky: hard-to-fix logging, projects that were hard to track or edit, a map stuck on its own tab with an Activity/Projects toggle, his and Cecilia's views looking alike, and an unused **Both** view. Kept from it: selected states must be obvious (solid, white outline, tick badge; the rest faded), kawaii icons in the Karate-Ka Companion style, Fell is light purple, Rested only for top rope, Kilter Board is never a project, beta notes are per person.

## Status
- Redesign: done (all six stages).
- Working: Projects tab with ID badges, Show / Sort, repeat matching and merge (2026-10-31); Climb home with map card and tiles, one log / edit panel (Kilter, video links) with Undo, Projects with a page per project and beta notes, Share → Climb Buddy from Google Photos, full map of one session, Progress (with Both), per-person views with swipe, always-dark theme.
- Open: remove the old workers.dev Worker.
- Ideas: speed climb or Boards & Training sessions (map labels for now).

## Changelog (latest first, keep about 5 lines)
- 2026-10-31: Quicker logging: compact log panel with pinned Save, one Log row on home, day climbs as badge cards with Show + Sort, "Didn't finish" wording, cheeky British pop-up after saving
- 2026-10-31: Projects easier to tell apart: grade / hold / wall badges on compact cards, Show + Sort under the map, map tags with hold dots, repeat matching when logging, duplicate warning on New, merge twins
- 2026-10-27: Full map shows only the picked person's climbs for one session (no project flags, no Week/Month); ‹ › step sessions
- 2026-10-26: Projects map: tapping a wall or tag picks the wall and narrows the cards below, instead of opening a project
- 2026-10-26: Phone Back fixed: no more jumping out to an older load of the app (Back tokens are only added on a tap)

## Working efficiently
- Find code with search first, then read only the relevant lines. Don't read index.html top to bottom.
- Make small, targeted edits. Never regenerate a whole file.
- Don't open a browser to test unless I ask. I'll test on my phone.
- No subagents unless I ask.
- One task per session. I'll /clear between tasks.
