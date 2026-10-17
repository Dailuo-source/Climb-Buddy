# Climb Buddy: handoff notes

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

- Tabs: **Session** (date picker, totals, per-grade ✓/Rest/✕), **Projects**, **Map**, **Progress** (month view).
- **Whose view**: one person switcher at the top of every tab (`whoSeg`), shared across tabs (`WHO` in localStorage `cb.who`; `BOTH` flag `cb.both`). Map, Projects and Progress also offer **Both**; Session always logs for one person. Swiping left/right anywhere switches (cycles Gary → Cecilia → Both, wraps): the screen follows the finger (`paintSwitch`): the new tint sweeps in as a soft gradient on body + header, the black highlight (`.wseg .pill`, positioned by fractional `--i`) glides between names, and the page drifts; past 35% or a quick flick it commits (`animateSwitch`), otherwise springs back. Taps animate the same way. Ignored while a panel is open or on anything that scrolls sideways (zoomed map, long chart). A person's projects = open ones they've tried and not yet sent (`openFor`). "This wall was reset" still resets everyone's projects on that wall.
- **Map**: `HUB_MAP` traces Gary's floor plan (1227×1402 coordinates); each zone's `area` + `type` must match a Lists value. Projects view shows dots in hold colour; Activity view shades walls by climbs in the last 30 days; tapping a wall opens stats, its projects and "This wall was reset". Details panels include a mini map to tap the wall, and typing an anchor number picks the wall from the Anchors ranges and pins its estimated spot. `HUB_MAP.anchorWalls` holds each rope wall's face as a path plus a number span (Pit 77→109 north→south down the west face; Upper 1→53 from the south end up the east face then west along the north edge — Claude's estimate, Gary to confirm); numbers are spread evenly along the path (`anchorPoint`), widened by any Anchors ranges in Lists. Map tab projects with an anchor sit at that spot instead of the wall's cluster.
- "Last tried" on a project comes only from its climbs' dates (Added is just a fallback), and new projects take the climb's date as Added, so backfilled sessions show the right day.
- Sync: same pattern as Pantry Pal (optimistic, batched ops, 60 s polling). Forms in open panels are never redrawn by a background sync.
- Always a light theme, even in phone dark mode (Gary's choice). Background tints by whose stats you're viewing: very light green for Gary, very light lavender for Cecilia, cream for Both (`data-who` on `<html>`, also sets the status-bar `theme-color`).
- Icon: simple side-on grey rock with a face climbing an orange rope, feet on a cream wall, on lavender. Gary deleted the maskable icon; keep it out of the manifest and `sw.js`.

## Changing things

- App files: edit, bump `CACHE` in `sw.js` (currently `climb-buddy-v11`), commit and push to `main`. Cloudflare deploys in about a minute.
- Code.gs: paste into Apps Script and redeploy (**Deploy → Manage deployments → pencil → New version**; URL stays the same). Claude did this through Claude in Chrome last time: open the sheet → Extensions → Apps Script, set `monaco.editor.getModels()[0]` to the raw GitHub Code.gs, Ctrl+S, redeploy. A new sheet column also needs adding to the live sheet (the header is matched by name).
- Sheet edits through the Google Sheets connector are fine. Insert rows rather than rewriting blocks in Lists, because the colour cells are painted.
- Commits use author `Claude <noreply@anthropic.com>`; GitHub rejects Gary's private email. Push with `https://github.com/Dailuo-source/Climb-Buddy.git`.
- Testing: a Node mock of SpreadsheetApp plus Playwright (route `script.google.com` to `api()`) worked well for end-to-end checks.

## Ideas mentioned but not built

- Tracking speed climb or Boards & Training sessions (they're labels on the map for now).
