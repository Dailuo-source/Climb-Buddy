# Climb Buddy setup

Climb Buddy keeps everything in a Google Sheet. The app itself is hosted on Cloudflare Pages.
It goes on Cloudflare rather than GitHub Pages because Chrome on Android won't install two
apps from the same website, and Pantry Pal already uses `dailuo-source.github.io`.

## 1. The Google Sheet (on a computer)

1. Make a new sheet at sheets.new and name it **Climb Buddy**.
2. Open **Extensions → Apps Script**, delete what's in `Code.gs`, paste in this repo's `Code.gs`, and save.
3. Pick **setup** in the function list at the top and press **Run**. Allow the permissions it asks for.
   This makes the **Climbs**, **Projects** and **Lists** tabs and fills Lists with The Hub's grades,
   tag colours, wall areas and hold colours.
4. Choose **Deploy → New deployment**, pick type **Web app**, set *Execute as: Me* and
   *Who has access: Anyone*, then **Deploy**. Copy the Web app URL (it ends in `/exec`).
5. Back in the sheet, reload it, then choose **Climb Buddy → Set web app link…** and paste that URL.

## 2. Hosting (Cloudflare Pages)

1. In Cloudflare, go to **Workers & Pages → Create → Pages → Connect to Git** and pick the **Climb-Buddy** repo.
2. Name the project **climbbuddy**. Leave the build settings empty and choose **Save and Deploy**.
3. The app is then at `https://climbbuddy.pages.dev/`. If Cloudflare gives you a different
   address, change `APP_SITE` near the top of `Code.gs` to match and save.

## 3. Each phone

1. In the sheet, choose **Climb Buddy → Show app key and phone link**.
2. Open that link on the phone in Chrome (email or message it to yourself). It connects the app to
   your sheet and remembers it.
3. Chrome menu **⋮ → Add to Home screen → Install**.

## Changing things later

- **Lists tab**: add climbers, gyms, grades (with a hex colour for boulder tags), wall areas and hold
  colours. Leave *Only at gym* blank for things that apply everywhere. Fill it in for things that
  only exist at one gym, like The Hub's wall areas. The app picks up changes on its next sync.
- **Climbs and Projects tabs**: edit or delete rows freely. Rows you type in yourself get an ID
  automatically.
- **App updates**: bump `CACHE` in `sw.js` whenever the app files change, so phones load the new version.
