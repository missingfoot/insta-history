# Instagram History

A Tampermonkey userscript that keeps a running log of the reels you come across on Instagram, split by where you saw them and grouped by day. It adds a small floating button to instagram.com that opens a panel of copyable reel links.

Everything stays in your browser. The script makes no network requests of its own and sends nothing anywhere.

## What it logs

Each source has its own tab in the panel.

| Tab | Page | What gets logged |
| --- | --- | --- |
| Home | `instagram.com/` | Each reel that crosses the middle of your screen as you scroll the feed. Photos and carousels are skipped. |
| Reels | `instagram.com/reels/...` | Each reel that snaps into view in the Reels feed. |
| Saved | `instagram.com/<you>/saved/...` | Every reel tile that loads in your saved posts grid. |
| Likes | Your activity > Interactions > Likes | Every video tile that loads in your likes grid. |

Other pages (profiles, explore, search and so on) are not logged. The panel shows "Not logging on this page" there, and you can still browse and copy what you already have.

All links are written in the form `https://www.instagram.com/reel/<shortcode>/`.

## Requirements

- A desktop browser with the [Tampermonkey](https://www.tampermonkey.net/) extension. Built and tested in Chrome. Other Chromium browsers should work. Firefox is untested.
- Instagram set to English. The Likes tab relies on English labels to tell videos from photos. The other tabs have fallbacks and should work in any language.

## Install

1. Install Tampermonkey from your browser's extension store.
2. In Chrome, give Tampermonkey permission to run userscripts. Depending on your Chrome version this is either the "Allow User Scripts" toggle on Tampermonkey's details page (`chrome://extensions`, then Details), or the Developer mode switch at the top right of `chrome://extensions`. If scripts do not run, see the [Tampermonkey FAQ, Q209](https://www.tampermonkey.net/faq.php?locale=en&q=Q209).
3. Add the script, either way:
   - From GitHub: open `instagram-history.user.js` in the repo and click Raw. Tampermonkey spots the `.user.js` address and shows an install page. Click Install.
   - By hand: open the Tampermonkey dashboard, click the + tab to create a new script, replace the template with the contents of `instagram-history.user.js`, and save.
4. Open or reload instagram.com. A "History" button appears at the bottom right, just above the Messages pill.

### Updating

Paste the new version over the old one in the Tampermonkey editor and save, or reinstall from the Raw link. Your logged history is kept, because it is stored by the page and not by the script.

To let Tampermonkey update the script automatically from your repo, add these two lines to the header block at the top of the script, with your own username and repo name, and bump `@version` whenever you change the script:

```js
// @downloadURL  https://raw.githubusercontent.com/<username>/<repo>/main/instagram-history.user.js
// @updateURL    https://raw.githubusercontent.com/<username>/<repo>/main/instagram-history.user.js
```

## Using it

- History button: opens and closes the panel. The number is the count for the page you are on.
- Tabs: Home, Reels, Saved, Likes, each with its count. A green dot marks the source being logged right now, and the panel jumps to that tab when you move to a logged page.
- Day sections: each tab is split into Today, Yesterday, weekday names for the rest of the past week, Last week, "2 weeks ago", "3 weeks ago", then month and year. Click a section header to expand or collapse it.
- Order: newest first. The most recently logged link is at the top and is highlighted yellow for about 5 seconds.
- Copy one link: click it. "Copied" shows briefly on the right of the row.
- Open a link: middle click, or ctrl/cmd click.
- Copy a day: the Copy button on that section's header.
- Copy all: copies every link in the tab you are viewing, one per line.
- Auto-scroll: shown on Saved and Likes. It keeps scrolling to the bottom so the whole grid loads and gets logged, and stops by itself once nothing new has loaded for about 7 seconds. Click Stop to end it early. Keep the Instagram tab in the foreground, because Instagram pauses loading in background tabs.
- Clear: deletes the log for the tab you are viewing only. It asks for a second click within 3 seconds to confirm. Nothing is ever cleared automatically.

## How each source is detected

Instagram exposes reels differently on every page, so each tab uses its own method.

- Home: every feed post is an `<article>`, and reels carry a `/reels/<id>/` link inside it. Instagram preloads several posts below the fold, so the script only counts the post crossing the vertical middle of the window. It checks about every 0.4 seconds, so a reel you fling past faster than that may not be logged.
- Reels: the feed has no reel links in the page at all. Instagram rewrites the address bar to `/reels/<id>/` as each reel snaps into view, and the script logs that.
- Saved: every tile links to `/p/<id>/` whether it is a reel or not, so reels are picked out by the "Clip" badge icon on the tile. The grid removes rows as you scroll, which is why the script collects continuously instead of reading the page once.
- Likes: tiles are plain buttons with no link. The thumbnail address contains a cache key that is the post's numeric media id in base64, and an Instagram shortcode is the same number written in a 64 character alphabet. The script decodes the id and rebuilds the link. It cross-checks the date built into the id against the "shared" date on the tile to make sure it read the id correctly.

When a post is opened in the pop-up viewer over a grid or feed, logging carries on for the page underneath.

## What the times mean

- Home and Reels: the time shown on each row is when you saw the reel.
- Saved and Likes: the day section is when the script first saw the tile, not when you saved or liked the post. Instagram does not show those dates on the grid. The first time you scroll through an existing collection, everything lands under Today.
- On Saved and Likes, "newest first" means newest logged. Posts further down the grid load later, so after an Auto-scroll the list is in the reverse order of the page.

## Where the data lives

History is stored in the browser's localStorage for instagram.com, one key per tab:

```
igHistory.v2.home
igHistory.v2.reels
igHistory.v2.saved
igHistory.v2.likes
```

Each value is a JSON array of `[shortcode, unixSeconds]` pairs in the order they were logged.

- Capacity: about 190,000 reels in total. Each entry is around 27 characters and browsers allow roughly 5 MB per site. If storage ever fills up, the panel says so and keeps what it has.
- Several Instagram tabs open at once share the same history and stay in step.
- The history belongs to one browser profile on one computer. It is not synced.
- Clearing site data or cookies for instagram.com in the browser settings wipes the history. Uninstalling or updating the script does not.

### Backing up

Open DevTools on instagram.com (F12), go to the Console tab, and run:

```js
copy(JSON.stringify(Object.fromEntries(Object.entries(localStorage).filter(([k]) => k.startsWith('igHistory.v2.')))))
```

That puts the whole history on your clipboard as JSON. Paste it into a file to keep it. Chrome may ask you to type `allow pasting` before it accepts pasted code in the console.

To restore, run this with your saved JSON in place of `...`. It replaces whatever is currently stored for those tabs, so reload the page afterwards.

```js
Object.entries(...).forEach(([key, value]) => localStorage.setItem(key, value))
```

## Limitations

- The script reads Instagram's page structure, which Instagram changes without notice. If a tab stops logging, that page's markup has probably changed and its detection needs updating. The other tabs are independent and keep working.
- Likes depends on Instagram keeping the media id in the thumbnail address, and on the English UI.
- Likes and Saved only log reels and videos. Photos and carousels are skipped.
- Only what loads in the page can be logged. For a full Saved or Likes history you need to scroll through the grid once, or use Auto-scroll.
- Desktop browsers only. It does not run in the Instagram mobile app.

## Troubleshooting

- No History button: check the script is enabled in the Tampermonkey dashboard, that Tampermonkey is allowed to run userscripts (install step 2), and reload the page.
- Button is there but the count does not move: check the status text at the top right of the panel. "Not logging on this page" means the page is not one of the four sources.
- Auto-scroll stops early: bring the Instagram tab to the foreground and press Auto-scroll again.
- Likes tab stays at 0: check Instagram is set to English, and that the tiles on the page are videos.

## Adding another source

Sources are defined in the `SOURCES` array near the top of the script. Each entry is an object:

| Field | Purpose |
| --- | --- |
| `id` | Storage key suffix. Do not change it once data exists. |
| `label` | Tab name. |
| `test(path)` | Returns true when `location.pathname` belongs to this source. The first matching source wins. |
| `collect()` | Returns the shortcodes visible right now. Called about every 0.4 seconds while the source is live. |
| `timed` | Optional. Set to true when the log time is the watch time, so rows show it. |
| `scroller()` | Optional. Returns the element Auto-scroll should drive. Leave it out to hide Auto-scroll for this source. |

## Files

- `instagram-history.user.js`: the userscript.
- `README.md`: this file.
