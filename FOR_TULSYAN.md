# The Shaadi Site, Explained

*A plain-language tour of `index.html` — what it is, how it hangs together, and the
things it taught us along the way.*

---

## 1. What this thing actually is

One file. `index.html`, about 43 KB, and that's the entire application.

No React. No build step. No `npm install`. No bundler, no framework, no
`node_modules` folder quietly eating 300 MB of your disk. You double-click the file
and a wedding invitation opens.

This is worth sitting with for a second, because the instinct in 2026 is to reach for
Next.js the moment someone says "website." But ask what this site actually needs to
do: show some words and pictures, remember one choice, play a song, count down to a
date. That's it. A framework would add a deployment pipeline, a dependency tree with
a shelf life, and a build that breaks in eighteen months when you want to fix a
typo — in exchange for conveniences this page never needed.

The test I'd apply: **will the person maintaining this in a year be able to open it
and change something?** With one HTML file, yes — even if that person is a cousin who
knows a little HTML. With a Next.js app, they'd first need the right Node version.

The trade-off is real, though, and you should see it clearly: everything lives in one
file, so the CSS is 60-odd lines of dense minified-looking rules, and finding the one
you want means `grep`. That's the tax. For a project this size, it's cheap.

**Stack:**

| Layer | Choice | Why |
|---|---|---|
| Markup | Hand-written HTML5 | It's a document. HTML is for documents. |
| Styling | Inline `<style>`, CSS custom properties | One file. `--maroon` in one place, used 40 times. |
| Behaviour | ~90 lines of vanilla JS | Nothing here needs a virtual DOM. |
| Fonts | Google Fonts (Cormorant Garamond, DM Sans, Yatra One) | The only external dependency. |
| Persistence | `localStorage` | Client-side, zero backend. |
| Hosting | Any static host | It's files. Netlify, GitHub Pages, S3, a USB stick. |

---

## 2. The shape of the page

The page is a vertical journey. Each section is a full-height "screen," and dividers
with little ✦ and ❈ glyphs sit between them like the pause between courses at a
wedding dinner.

```
<body>
 ├─ <audio id="bgMusic">            the instrumental, loaded only on demand
 ├─ <button id="musicControl">      floating ♫ toggle, fixed bottom-right
 ├─ <header class="hero">           names, date, live countdown, "Enter the Shaadi"
 ├─ <section id="big-news">         "Hum dono ki shaadi hai!"
 ├─ <section id="choose">           ← THE GATE. Team Shreya / Team Aditya
 │    └─ #vibe                        excitement slider, revealed after you pick
 ├─ <section id="itinerary">        6-card swipeable carousel + Save the dates
 ├─ <section class="marriage-game"> the quiz
 ├─ <section class="venue">         Parklane Resort, travel notes, map
 ├─ <section class="family-details"> both families
 ├─ <section class="share">         WhatsApp share
 └─ <footer>
```

Every one of those is a **direct child of `<body>`**. That flat structure looks
naive — no wrapper divs, no layout container — but it turned out to be the single
most useful architectural property of the whole file. More on that in a moment.

### The three moving parts

**The carousel** (`#eventTrack`) is a flexbox row with `overflow-x: auto` and
`scroll-snap-type: x mandatory`. That's it. The browser does the swiping, the
momentum, the snapping — all natively, all at 60fps, with no JavaScript touching the
scroll. The JS only *listens*: a debounced `scroll` handler works out which card is
nearest and lights up the right dot.

This is a pattern worth stealing. Ten years ago this needed a 40 KB carousel library
with touch handlers and inertia math. Now it's four CSS properties. A good engineer
periodically re-checks whether the library they reach for by habit has been absorbed
into the platform.

**The confetti** (`burst()`) creates 22 `<div>`s with random left positions and
animation delays, appends them to `<body>`, and removes them after 2.5 seconds. Crude,
effective, and it self-cleans. The `setTimeout(() => e.remove(), 2500)` is the
important half — without it you'd leak DOM nodes every time someone tapped a button.

**The family cards** quietly reorder themselves. Pick Team Shreya and the Drolia
family card slides to the front; pick Team Aditya and the Tulsyans lead. If nobody
has picked, `arrangeFamilyCards()` flips a coin. Small touch, and exactly the sort of
detail that makes a page feel considered — nobody's side is permanently second.

---

## 3. The gate, and why it works the way it does

The ask: **you shouldn't be able to scroll past the picker without choosing a team.**

The obvious implementation is to fight the scroll — listen for `wheel` and
`touchmove`, call `preventDefault()`, maybe pin `body { overflow: hidden }`. Every
part of that is misery. iOS Safari has its own rubber-band physics. Passive event
listeners mean `preventDefault()` sometimes doesn't fire. `overflow: hidden` on
`<body>` scrolls you back to the top on some browsers and loses your position. And a
keyboard user pressing `End` sails straight past your careful touch handling.

So we didn't fight the scroll. We removed the destination:

```css
body.gated #choose ~ *:not(.confetti) { display: none }
```

The `~` is the *general sibling combinator*: "every element after `#choose` that
shares its parent." Because every section is a direct child of `<body>`, that one
selector reaches the itinerary, the game, the venue, the families, the share block,
the footer, and all the dividers between them. `display: none` takes them out of
layout entirely, so the document genuinely ends at the picker. There is nothing below
to scroll to.

**No scroll to block, because there's nowhere to go.** No touch handlers, no
`preventDefault`, no platform quirks. It works identically on an iPhone, a
ten-year-old Android, and a keyboard.

The analogy I keep coming back to: you can hire a bouncer to stop people walking down
a corridor, or you can not build the corridor yet. The bouncer needs training,
supervision, and will eventually let someone through. The missing corridor never
fails.

Two details that make it hold up:

- **`:not(.confetti)`** — `burst()` appends confetti to `<body>`, which makes each
  piece a later sibling of `#choose`. Without the exclusion, the celebration for
  picking a team would be invisible. A one-line escape hatch for a one-line rule.
- **A visible reason.** A page that just *stops* reads as broken. `#gateHint` says
  "Pick your team to unlock the rest of the invitation ✦", and if you try to scroll
  or press `↓` at the bottom, it bounces. **Never let a constraint look like a bug.**

---

## 4. Bugs we hit, and what each one teaches

### 4.1 The "default selection" that wasn't in the HTML

**Symptom:** Team Shreya appeared pre-selected on every visit.

**The hunt:** `grep` for `class="choice active"` in the HTML — nothing. No card was
marked selected anywhere in the markup.

**The culprit**, on the last line of the script:

```js
const s = localStorage.getItem('weddingSide');
if (s) { side = s; document.querySelector(`[data-side="${CSS.escape(s)}"]`)?.classList.add('active'); }
```

A previous visit had saved `"Bride's Side"`, and the page dutifully restored it —
forever. The "default" wasn't in the code at all. It was in *state left over on that
one machine.*

**The lesson, and it's a big one:** when something is wrong on screen but you can't
find it in the source, **stop reading the source and ask where the state lives.**
`localStorage`, cookies, IndexedDB, a service worker cache, a stale CDN edge. This
class of bug has a nasty signature: *it doesn't reproduce for the person who didn't
click the thing.* You'd have shipped this, and every guest who ever picked a side
would see it pre-picked forever while it looked perfect on a fresh browser.

Whenever a colleague says "works on my machine," persisted client state is one of the
first three suspects.

### 4.2 The `height` attribute that ate the layout

Adding `width` and `height` attributes to the event images is a genuine best practice
— it lets the browser reserve the right space before the image arrives, so the page
doesn't lurch as things load (that lurch has a name: Cumulative Layout Shift).

So I added them. And the cards ballooned, with big empty cream bands above and below
each photo.

**Why:** the CSS said `.event-art { width: 100%; aspect-ratio: 5/4; object-fit: contain }`.
`aspect-ratio` only computes a dimension when the *other* one is `auto`. My
`height="800"` attribute became a presentational hint — a real, computed height —
so both dimensions were now definite, `aspect-ratio` had nothing to solve for, and
quietly stepped aside. The image then sat letterboxed inside an 800px-tall box.

**The fix is one line**, and it's the line every CSS reset has shipped for a decade:

```css
.event-art { height: auto }
```

**The lesson:** two correct techniques can be wrong *together*. Nobody would call
"add width/height to images" a mistake, and nobody would call `aspect-ratio` a
mistake. The bug lives in the seam between them. This is why you look at the result
after a change that "obviously" can't break anything — I only caught it because I
screenshotted the page instead of trusting that an attribute couldn't affect layout.

### 4.3 The ticking screen-reader

I turned the static "83 days to go" into a live clock updating every second. The
element already had `aria-live="polite"` from before — perfectly sensible when it
changed once a day.

With a per-second update, `aria-live` means **a screen reader announces the countdown
every single second, forever, over everything else on the page.** For a blind guest,
the invitation becomes unusable.

Fix: drop the live region, and mark the ticking clock `aria-hidden="true"` so the day
count still reads normally.

**The lesson:** accessibility attributes are not decorations you sprinkle on for
points — they're *behaviour*. `aria-live` is a promise to interrupt the user. When you
change how often something updates, you've changed what that promise costs. Any time
you increase an element's update frequency, go and check what's subscribed to it.

### 4.4 The bug that wasn't: a 500px floor

Screenshotting at `--window-size=390,844` produced a page with everything clipped on
the right. Looked like a serious mobile overflow bug.

Before "fixing" it, I measured — injected a probe that logged
`document.documentElement.clientWidth`. It said **500**.

Headless Chrome enforces a 500px minimum viewport. The layout was fine; I'd been
handed a 390px-wide crop of a 500px-wide page.

**The lesson, and maybe the most valuable one here:** *verify your instrument before
you trust its reading.* I was about thirty seconds from "fixing" a layout that was
never broken — and that fix would have been a real bug, shipped in response to an
imaginary one. When a tool reports something surprising, the tool is a suspect too.

---

## 5. Making it fast: 12 MB → about 250 KB

The site weighed 12 MB. For guests in Dhanbad on patchy mobile data, that's not a
slow site, that's a site that never finishes loading. Here's where it went and how it
came down. Note that almost none of this was clever:

| Asset | Before | After | How |
|---|---|---|---|
| `ishq-bulaava.mp3` | 2.9 MB | **0** | Referenced nowhere. Deleted. |
| `jashn-e-bahaaraa` | 4.9 MB MP3 | 3.1 MB M4A, **not downloaded until tapped** | `preload="none"` + AAC via `afconvert` |
| `hero-invitation.png` | 2.0 MB | 131 KB JPEG | It renders at `opacity: .12`. It's texture. |
| 6 event photos | 1.8 MB | ~950 KB, **lazy** | Resized to 800px, `loading="lazy"` |

**Actual first load is now roughly 250 KB** — the HTML, the hero image, and fonts.
Everything else arrives only if the guest goes looking for it.

Three things worth internalising:

**The biggest win was deleting a file.** 2.9 MB of the 12 was an audio track nothing
referenced. Before optimising anything, check what you're shipping that you don't
even use. `grep -c "filename" index.html` returning `0` is the cheapest performance
win in existence.

**`preload="auto"` on a 5 MB audio file is a hostile default.** The browser was
downloading five megabytes of music before the guest had done anything, on the *hope*
they might press play — and mobile browsers block autoplay anyway, so it was usually
five megabytes spent on nothing. `preload="none"` costs a moment of delay when
someone actually taps.

**Match the pixels to the display size.** Those photos were 1145×1374 rendering into
a ~370px-wide card. We were shipping nine times the pixels anyone could see. Resized
to 800px wide (still 2× for retina), they look identical.

Originals are all preserved in `originals/` — **delete that folder before you
deploy**, or you'll upload the 12 MB you just removed.

---

## 6. Link previews: the highest-leverage 20 lines

This site spreads by WhatsApp forward. Somebody sends it to the family group, that
group forwards it to another. And every one of those forwards was rendering as a bare
grey rectangle, because the page had no Open Graph tags.

Open Graph is a small set of `<meta>` tags that WhatsApp, iMessage, Facebook and the
rest read to build the preview card. No tags, no card.

We added `og:title`, `og:description`, and `og:image` — the last pointing at
`og-image.jpg`, a 1200×630 crop of the invitation art showing the names, the date and
Dhanbad. Now a forward arrives as an invitation instead of a link.

**Think about it as conversion, because that's what it is.** People decide whether to
tap based on the preview. Twenty lines of `<meta>` probably move that number more than
any amount of work on the page itself. The most valuable part of your site can be the
part that renders somewhere else entirely.

⚠️ **One thing you must do at deploy time:** `og:image` is currently the relative path
`og-image.jpg`. Some scrapers won't resolve that. Once you have a domain, change it to
the full `https://yourdomain.com/og-image.jpg`. There's a comment in the `<head>`
saying so. After deploying, paste the URL into
[Facebook's Sharing Debugger](https://developers.facebook.com/tools/debug/) to force a
re-scrape — previews are cached hard, and if the first scrape sees a broken image,
that's what everyone gets for a long time.

---

## 7. Accessibility: the day the gate locked people out

The team cards started as `<div onclick="chooseSide(this)">`. That works with a mouse
and works with a finger, so it looks finished.

A `<div>` is not focusable. You cannot Tab to it. Enter and Space do nothing. A screen
reader announces it as a lump of text, not something you can activate.

Before the gate, that was rude. **After the gate, it was a locked door.** Picking a
team became mandatory to see any of the page — and keyboard and screen-reader guests
had no way to pick. A change intended to improve the experience had, for those guests,
destroyed it entirely.

This is the thing about accessibility gaps: they're usually latent. The `div` had been
"fine" for months. Adding a hard dependency on an inaccessible control turned a small
sin into a total lockout, in a change that had nothing to do with accessibility.
**When you make something mandatory, re-audit how it's reached.**

The fix was `<button type="button">`, which brings focusability, Enter/Space, and the
right announcement for free — no JavaScript. Plus `aria-pressed` toggling so a screen
reader says "pressed."

One wrinkle: `<button>` may only contain *phrasing content*, and the cards had `<h3>`
and `<p>` inside. Browsers tolerate it; validators don't. So the `<h3>` became
`<span class="choice-title">` and the `<p>` became `<span class="choice-copy">`, with
the CSS retargeted and `display: block` restored. Visually identical — the desktop
screenshot before and after is pixel-for-pixel the same — and now it's valid.

Also fixed in the same pass: an `aria-label` on the excitement slider (it was an
unlabelled range input — "slider, 92" and nothing else), `role="status"` on the quiz
answer so it's announced, `rel="noopener"` on the `target="_blank"` map link, and
`lang="hi"` on the Devanagari so screen readers don't try to pronounce शुभ विवाह as
English.

And `prefers-reduced-motion`: floating petals, confetti, pulsing button and smooth
scroll all stand down for anyone who's asked their OS for less movement. For people
with vestibular disorders, decorative motion causes actual nausea. One media query.

---

## 8. Habits worth stealing from this project

**Verify by looking.** Every claim in this document was checked — screenshots at
phone and desktop widths, a script that clicks a team and reports whether sections
became visible, `file` on the transcoded audio, a link-checker for dangling `src`
paths. Two of the four bugs above were caught by looking at a rendered page, not by
reading code. Reading code tells you what you *meant*.

**Assert on your edits.** Every change to `index.html` went through a script that did
`assert s.count(old) == 1` before replacing. Not once, not twice, exactly once. Twice
means the pattern is ambiguous and you're about to corrupt something; zero means
you're editing a file you don't understand. Cheap, and it turns silent corruption into
a loud stop. (This is exactly what `sed -i` doesn't give you.)

**Keep the originals.** `originals/` holds the untouched PNG, MP3s, JPEGs and a
backup of `index.html`. Lossy conversion is one-way. The five seconds to `cp` first
is insurance against re-sourcing artwork you can't regenerate.

**Fix the cause, not the symptom.** Pre-selected team? Don't add code to clear it —
find out it came from `localStorage`. Can't scroll past a section? Don't intercept
scroll — remove what's below.

**Say when you broke it.** The `aspect-ratio` bug was mine, introduced two steps
earlier by a change I'd have defended as a best practice. Naming that plainly is
faster than quietly patching it, and it's the difference between a colleague who can
be trusted with a codebase and one who can't.

---

## 9. If you pick this up again

**The one that matters: nobody receives an RSVP.** "Lock it in" writes to
`localStorage` — that is, to the guest's own browser, where you will never see it. And
"Send my vibe on WhatsApp" opens WhatsApp's *contact chooser*, so it only reaches you
if the guest happens to pick you. Right now the site collects nothing.

Two ways out, in ascending order of effort:

1. **Prefill the number:** `https://wa.me/91XXXXXXXXXX?text=...` sends straight to you.
   Five-minute change.
2. **A real RSVP form** — attending, headcount, arrival date, veg/Jain — posting to a
   Google Form or Formspree. Still no backend, and you end up with a spreadsheet of
   who's actually coming. In November you will want that spreadsheet.

Other things guests will ask for regardless: dress code per function (especially haldi
colours), room and accommodation details, a tap-to-call help-desk number, the wedding
WhatsApp group link, and a line about gifts.

**Before you deploy:**

- [ ] Delete `originals/` (12 MB of backups)
- [ ] Make `og:image` an absolute `https://` URL
- [ ] Run the URL through Facebook's Sharing Debugger
- [ ] Open it on a real phone on mobile data, not just desktop Chrome
- [ ] Tab through the page with the keyboard — you should reach both team buttons

---

*Built as one HTML file, and it should stay that way.*
