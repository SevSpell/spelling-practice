# SevSpell (Spelling Practice) — master project reference

Single reference document for this project, superseding all previous
versions of this file. Read this before making changes, and start a
fresh chat from this file rather than an older one, this project has
grown a lot and older references are now meaningfully out of date.

## 1. What this is

A Spanish and English spelling practice app built for Harry's two
daughters (Rosie, 5, and Scarlett, 9). A parent photographs or types a
weekly word list, each child picks their own profile and practises
spelling, either Spanish (copying a shown word, or translating from
English) or English (copying a shown word). Correct, encouraging
practice earns coins, unlocks collectible creatures ("Sevlings"), and
unlocks arcade game time.

## 2. Live deployment

- **Hosted at:** `https://sevspell.github.io/spelling-practice/`
- **GitHub repo:** public, named `spelling-practice`, file must be
  named `index.html` at the repo root
- **Firebase project ID:** `spelling-app-83697`
- Files in the repo root: `index.html`, `logo-full.png`, `logo-mark.png`,
  `apple-touch-icon.png`, `favicon.png`, plus real Sevling art as
  `<id>.png`, `<id>-blink.png`, `<id>-happy.png` (Puggle, Lotyl, Sword,
  Pepper)

**Open issue:** Harry reported no icon appears on iOS when adding to
the home screen, despite the code being correctly set up. Two likely
causes flagged and awaiting his check: (1) the icon files may not
actually be present in the GitHub repo, or (2) iOS caches home-screen
icons per-URL very aggressively, if the page was ever added to a home
screen before the icons were live, deleting that old icon and clearing
Safari's site data for the domain is required, reloading alone won't
fix it. Not yet confirmed which.

## 3. Architecture

- **Version and update check (read this before every change).** The
  file carries a `const APP_VERSION = 'YYYY-MM-DD.n'` near the top of
  the script. **Bump it with every change to `index.html`**, otherwise
  devices won't know there's anything new. Built because the iOS
  home-screen copy kept running an old cached version: iOS often
  resumes a suspended copy rather than reloading, and GitHub Pages
  serves the file with a ~10 minute cache. How it works:
  - On launch, and whenever the app comes back to the front
    (`visibilitychange`/`pageshow`, throttled to once every 2 minutes),
    it fetches a fresh copy of itself with a unique `?v=` URL and
    `cache:'no-store'`, and reads the `APP_VERSION` inside it.
  - If it's different and the app is on the sign-in or profile picker
    screen, it reloads straight away (nothing to lose there). Anywhere
    else (mid-practice, mid-game) it shows a gold "A new version of
    SevSpell is ready, Update now" banner instead, so a child's session
    is never interrupted.
  - Loop guard: it won't auto-reload twice for the same version in one
    session, in case GitHub hasn't finished publishing yet.
  - Settings has an **App updates** card showing the running version,
    with **Check for updates** and **Reload latest version** (always
    reloads via a fresh URL, for when things still look stale).
  - No-cache `<meta>` tags were added too, but GitHub Pages' own headers
    win, so the version check is what actually does the work.
  - After pushing a change, allow a couple of minutes for GitHub Pages
    to publish before expecting devices to pick it up.

- **Single HTML file**, no build step, no server, all logic in one
  `<script type="module">` block, plain DOM manipulation
  (`app.innerHTML = ...` per view).
- **Firebase**: Email/Password auth (one login per family, see §4) and
  Firestore for all data storage.
- **Gemini API** (optional): photo → word list, via the
  **Interactions API** and **`gemini-3.6-flash`** model (this project's
  second Gemini API migration; if photo reading ever breaks again with
  a "model no longer available" style error, check
  https://ai.google.dev/gemini-api/docs, it's probably a third).
- **Hosting:** GitHub Pages.
- **Firebase config is baked into the file** (`DEFAULT_FIREBASE_CONFIG`),
  intentionally, it's a public identifier not a secret, the Firestore
  rules do the actual protecting.

## 4. Family accounts and profiles

**One shared Firebase login for the whole family**, not one account
per child. This was a deliberate redesign (children's separate
accounts hit Firebase's "email must be unique" wall awkwardly). Rosie
and Scarlett are **profiles** inside that one account, not separate
Firebase users:

- Sign in once (the family login) → lands on a **profile picker**
  ("Who's practising?") → tap a profile, no PIN, straight into that
  profile's own Home screen.
- Each profile has its own word lists, coins, Sevling collection,
  arcade time, and voice preference, completely private from the
  other, just switched with a tap via "Switch profile" on Home.
- **Adding a profile**: "+ Add profile" on the picker, either start
  fresh or **import an existing account** (a genuinely useful leftover
  from before this redesign: enter that old account's own
  username/password, and its coins/lists/Sevlings get copied into the
  new profile via a temporary secondary Firebase Auth session, the old
  account itself is untouched).
- **No PIN** on profiles, deliberately, they're just a fast switcher
  for siblings sharing one device, not a security boundary. The one
  real login (the family account) is the actual security boundary.

### Data model

```
spelling_users/{familyUid}/data/profiles          → the profile list itself (family-level)
spelling_users/{familyUid}/profiles/{profileId}/data/index   → that profile's word-list index
spelling_users/{familyUid}/profiles/{profileId}/data/game    → coins, Sevlings, arcade time, streaks
spelling_users/{familyUid}/profiles/{profileId}/data/prefs   → that profile's voice preference
spelling_users/{familyUid}/profiles/{profileId}/lists/{id}   → one doc per saved word list
family/invite, family/settings                     → shared invite code + Gemini key (unchanged)
```

The existing Firestore rule (`spelling_users/{userId}/{document=**}`)
already covers this nesting with no changes needed, the recursive
wildcard matches any depth.

`fsGet`/`fsSet`/`fsDelete` are family-level (only used for the
profiles list now). `fsGetP`/`fsSetP`/`fsDeleteP` are profile-scoped
(everything else), automatically injecting the current profile ID into
the path.

## 5. Word lists

**Spanish and English lists are completely separate**, a toggle on
Home switches which set of lists and modes you see. Spanish lists
store `{spanish, english}` pairs; English lists store just the
spelling word (kept in the same `english` field for code simplicity,
`spanish` left blank). Each list is tagged `lang:'es'|'en'` in its
index entry; untagged legacy lists are treated as Spanish for backward
compatibility.

Photo reading uses a different Gemini prompt per language
(`EXTRACT_PROMPT_ES` expects translation pairs, `EXTRACT_PROMPT_EN`
expects a plain word list), and the review table adapts to one column
for English lists.

**Pasting a list as text** (added because Gemini photo reading kept
hitting its usage limit): the Add list screen has an "Or paste the
words as text" box underneath the photo option, which works with no
API key. `parsePastedWords()` reads:
- markdown tables (`| 16 | dieciséis |`), skipping `|---|` lines and a
  header row if there is one;
- tab-separated rows copied from a spreadsheet;
- `Spanish - English` style lines (`-`, `=`, `:`, en/em dash, or a
  single comma);
- one word per line, for English lists (a numbering column or `1.`
  style numbering is ignored).

For Spanish lists it works out which column is Spanish: a header naming
the columns wins; otherwise each column is scored for Spanish signs
(accents, ñ, ¿¡, common Spanish words and endings) against English
signs, and a column of plain numbers is treated as the English side,
so `| 16 | dieciséis |` becomes English "16", Spanish "dieciséis"
(practice then shows/says "16" and asks for the Spanish). With three
or more columns, a leading numbering column is dropped. Pure numbers
are kept as they are (the usual word cleaner would strip them as
numbering). Everything lands on the normal review screen, which now
has a **Swap Spanish and English** button for when the guess is wrong.

**List mastery**: the first time a list is completed **100% correct**
(non-retry), it's flagged `mastered` on its index entry, permanently.

## 6. Practice modes and grading

Three modes, driven by a shared `PRACTICE_MODES` config
(`modeInfo(mode)`):

| Mode | Shows | Grades against | Translation shown | Reveal timer |
|---|---|---|---|---|
| `copy` (Copy the Spanish) | Spanish | Spanish | Yes (English, always visible) | Yes |
| `copy-en` (English spelling) | English | English | No | Yes |
| `translate` (Translate to Spanish) | English | Spanish | No | No (English is the permanent prompt, not hidden) |

**Grading** (`checkAnswer`, `normaliseBase`, `stripAccents`):
capitalisation and punctuation ignored entirely. A missing accent
still counts correct but shows a gentle note, "Correct!" and the note
line in **orange**, the correct spelling on its own line below with
only the actual accented letters (á é í ó ú ü) picked out in **blue**
(not bold, not a background box, just a colour change, so it reads as
one continuous word). `ñ` is never treated as an accent, a mismatch
there is marked properly wrong.

**Reveal sequence** (`copy` and `copy-en` modes only): on each new
word, audio plays automatically (no tap needed), English then Spanish
for `copy` mode, just the one language for `copy-en`. The word itself
stays hidden until that audio **actually finishes** (a real
`onend`/`onerror` event, not a fixed delay, with a 6s safety cap), then
fades in. Stays visible for **4 seconds** normally, or **2 seconds**
if that list has already been mastered once, then fades out and is
gone for that word, speaker button still works throughout. Autoplay
only works because `unlockSpeech()` fires a silent utterance
synchronously the instant a mode button is tapped, before any
Firestore `await` breaks the browser's gesture-triggered-audio
requirement.

**Voice**: `resolveVoice()` tries the profile's saved preference first
(exact name+lang match), falls back to `pickVoice()`'s automatic
best-match otherwise. All speech at 0.9x rate. Settings has a Voice
card (two dropdowns, populated from whatever's actually installed on
the current device) saved per-profile.

## 7. Coins, Sevlings, and the Trophy Case

**Coins/medals**: unchanged, every non-retry session earns trophy
(0 wrong, +£5) / gold (1, +£3) / silver (2, +£2) / bronze (3–4, +£1)
based on wrong-word count. Tapping the coins badge (top-right, any
screen) opens a dedicated **Trophy Case** page (medal counts + coin
total), separate from the Sevlings creature collection.

**Sevlings**: 34 creatures across 4 collections (Farm/Jungle/Sea/Pets,
8/8/8/10). Unlocking a new one needs **4–5 fully correct practices
(re-rolled each cycle) on the *same specific list*** (not just the
same language, tracked per `listId` in `game.listStreaks`), **and** a
cooldown since the last win, whichever finishes later. The cooldown
starts as a **2–3 week base** (`game.cooldownDays`, re-rolled 14–21 on
each win), but **shrinks by a day for every full-mark (0 wrong)
practice on any list** since that win (`game.fullMarksSinceLastWin`),
down to a **floor of 7 days/1 week**. An imperfect session doesn't
reset that list's streak progress, it just doesn't add to it, but it
also doesn't add to the cooldown-shrinking count, only a genuinely
perfect practice does. Winning resets every list's streak progress,
the cooldown base, and the full-marks counter together, and re-rolls
the streak goal and cooldown base.

The "N more days until your next Sevling" message (Sevlings page and
results screen, `sevlingsProgressLine()`) always shows the *effective*
wait, i.e. already reduced by perfect scores so far. Because that
wasn't obvious, it now adds a line explaining that every perfect score
on any list takes a day off, and how short the wait could get (days
left to the 7-day floor), or says the floor has already been reached.

4 of the 34 have real artwork so far: Puggle, Lotyl and Sword (set
via `image` fields on their `SEVLINGS` entries) and Pepper the rooster
(three cut-out PNGs produced from Harry's images, picked up
automatically by the hatching-egg art check, no `SEVLINGS` fields).
The rest are generated placeholder SVGs;
`sevlingIcon()` prefers real art when present and generates its
silhouette automatically via a CSS filter, no separate asset needed.

**"Hatching soon" eggs**: any *unlocked* Sevling with no artwork yet
shows a wobbling rainbow egg (`sevlingEggSvg()`, Foil ink outline and
hard shadow, gold/blue sparkles, a small crack) instead of the old
coloured placeholder silhouette, with a holographic "Hatching soon"
pill on the collection tile, the Care screen and the win screen. Locked
Sevlings still show the dark placeholder silhouette. **Art is picked up
automatically**: for any unlocked Sevling without an `image` field,
the app quietly checks GitHub for `<id>.png` (and `<id>-blink.png` +
`<id>-happy.png`), and as soon as they exist it swaps the egg for the
real art, including the idle animation if all three frames are there.
No code change or `SEVLINGS` edit needed, just upload the files with
the right names. Missing files are re-checked every time the app comes
back to the front. Display screens simply redraw; inside a game only
the egg itself is swapped, so play isn't interrupted. The wobble is off
for game pieces and for reduced-motion users.

**Real-artwork Sevlings also get an idle blink/happy animation** while
displayed unlocked on the collection grid, the Care screen, and the
win/reveal screen (not on small zoomed game-piece faces or on tokens
a game loop redraws continuously, e.g. the Maze Munch player marker,
the Brick Breaker paddle badge or the Overhead Racer driver face,
`sevlingIcon()`'s `faceOnly`/`noAnim` params exclude those). A Sevling
with both `imageBlink` and `imageHappy` fields set automatically picks
this up via `data-sevling-anim` on its `<img>`, wired by
`wireSevlingAnimations()` and cleared by `stopSevlingAnimations()` on
every `render()` so timers never pile up or target a removed element
across view changes.

The frame sequence is chosen by the Sevling's current happiness
(`game.sevlingCare[id].happiness`), read fresh on every tick so a
happiness change (from feeding, say) takes effect on the very next
frame:

- **Happiness 60% or above:** loops happy, blink, happy, mostly
  smiling with an occasional blink.
- **Happiness below 60%:** loops resting, blink, a plainer idle blink
  with no smile frame.

Each frame is held for a set duration (resting 2.5s, blink 180ms,
happy 1.4s). A small random stagger on start keeps multiple visible
Sevlings from blinking in unison.

**Producing new art for this treatment**: export on a plain white or
transparent background, 512×512px or larger (1024×1024px is fine).
Three frames per Sevling, resting/blink/happy, same pose, crop and
lighting throughout, only the face changes, generate the resting one
first then feed that exact image back for the blink/happy edits so
the pose locks across all three. Filenames follow `<id>.png`,
`<id>-blink.png`, `<id>-happy.png`, matching the fields on that
Sevling's `SEVLINGS` entry.

**Care screen**: spend coins on food (happiness, caps at 100, never
decays) or one-off cosmetic accessories, per Sevling. Every unlocked
Sevling also shows a small mood emoji (`sevlingMoodEmoji()`) reflecting
its current happiness, on the collection grid tile and next to the
numeric reading on the Care screen itself: 🤩 90%+, 😊 60–89% (the same
60% split the idle happy/normal animation above uses), 😐 20–59%, 😢
under 20%. Deliberately not shown on the win/reveal screen, a freshly
unlocked Sevling starts at 50% happiness (neutral face), which would
undercut the moment of winning it.

## 8. Arcade

Reached via its own nav tab. **Unlock mechanic**: score 90%+ three
times (cumulative, non-punishing, same philosophy as Sevling streaks)
on the *same list* → +5 minutes of arcade time, tracked separately
from the Sevling-unlock streak (`game.gameUnlockStreaks`, different
threshold). Time **stacks** across different lists, and across
sessions if unused. A game already in progress is always allowed to
finish even if time runs out mid-session, only *starting a new one* is
blocked until more time's earned.

Five games, each with an optional Sevling (or "classic") character
selector, the choice saved per profile on the game doc
(`selectedConnect4Sevling`, `selectedSnakeSevling`, `selectedMazeSevling`,
`selectedBrickSevling`, `selectedRacerSevling`):

- **Connect 4** — 7×6, simple heuristic AI (take a winning move, block
  an obvious loss, otherwise weighted toward the centre with some
  randomness so it's genuinely beatable). Opponent's counter is always
  a light-blue circle, with the Sevling's face zoomed in on top if one's
  selected.
- **Snake** — 12×12 grid, arrow buttons, 190ms tick, board renders up
  to 560px (bumped up after the CSS `max-width` alone turned out not
  to matter, the wrapping `.card`'s own padding was silently capping
  the real size, fixed by trimming that card's padding directly).
  Head is enlarged (~145% of a normal cell) with a border matching the
  body colour.
- **Maze Munch** — an original maze-chase game (not a reproduction of
  the licensed Pac-Man character designs, deliberately generic ghost
  shapes and a plain circular muncher). 3 ghosts, move every 3rd tick
  (half-then-some slower than the player), light randomness so they're
  not perfectly relentless.
- **Brick Breaker**: Breakout-style, 480×560 logical canvas scaled to
  the card. Drag a finger (or mouse) along the board to move the
  paddle, tap to launch; arrow keys and space also work on a laptop.
  Deliberately forgiving: 5 lives, ball starts at 240 logical px/s and
  only speeds up 6% per level (capped at +35%), wide 100px paddle, the
  ball always rests on the paddle after a lost life until tapped.
  Bounce angle depends on where the ball lands on the paddle (up to
  60°), with a little randomness and minimum horizontal/vertical speed
  so it can never get stuck bouncing straight up and down or
  side to side. Three wall layouts cycle (full wall, checkerboard,
  diamond), with 2-hit bricks from level 2. About 1 in 6 bricks is a
  holographic power-up brick; breaking it drops a falling icon that
  activates when caught: **wide paddle** (1.6×, 12s), **multi-ball**
  (every ball splits into three, capped at 9), **sticky paddle** (15s,
  catches the ball, tap/space to release) and **lasers** (10s, fire
  automatically from both ends of the paddle). Every power type appears
  at least once per wall. Level cleared → short banner → next wall.
- **Overhead Racer**: Micro Machines style top-down racer, 3 laps
  against 2 computer cars (blue and gold, player is pink) on one fixed
  winding track (`RACER_CONTROL_POINTS`, smoothed with Catmull-Rom),
  camera follows the player, minimap top-right. Ink-navy track with
  neon pink/blue edges and a gold centre dash on the usual lavender
  background. **Gold booster pads** give a short turbo (1.4×), **oil
  slicks** (holographic sheen) cause a 0.9s spin-out, after which the
  car carries on in its original direction. The car accelerates by
  itself, the player only steers: hold the big left/right buttons,
  hold either half of the track, or use the arrow keys. Driving off
  the track halves speed, a soft cushion just beyond the edge stops the
  car wandering off, and the car can never turn to face backwards.
  Two difficulty settings on the setup screen, saved per profile
  (`racerDifficulty`), **Speedy is the default** (Harry's choice, so
  winning takes real steering), Gentle is there for younger players:
  - **Gentle**: computer cars at 78% of the player's top
    speed, strong steering help (with no input the car steers itself
    round the track), heavy rubber-banding in the player's favour
    (computer cars ease right off when ahead). Simulated test: a player
    who never touches the controls wins every time; one who randomly
    mashes left/right usually comes 3rd but stays close.
  - **Speedy** (default): computer cars at 88%, weaker steering help,
    lighter rubber-banding, needs real steering to win. Simulated
    test: no steering or random mashing comes 3rd, accurate steering
    wins.

  **Pit-stop spellings**: before racing, a setup screen lists every
  saved list (English and Spanish together, tick any number), plus a
  choice for Spanish words: copy the Spanish (uses the `copy` practice
  mode) or translate from English (`translate` mode); English words
  always use `copy-en`. Selection saved per profile (`racerListIds`,
  `racerSpanishMode`). Every 10 seconds of actual racing, the race
  freezes and a word is shown as an overlay, using the same
  `PRACTICE_MODES` entry, audio, listen-then-reveal-then-fade sequence
  and feedback display as the practice screen, except the word only
  **flashes up for 1 second** (`RACER_FLASH_MS`) before fading, rather
  than practice mode's 4s (3s once mastered), and the mastered-list
  shortening doesn't apply. The speaker button still replays the
  audio. In translate mode the English prompt stays on screen, as in
  practice, since it's the clue rather than the spelling. Grading goes through
  `gradeSpelling()`, which is simply the existing
  `checkAnswer`/`normaliseBase`/`stripAccents` logic pulled out into a
  pure function that practice mode now also calls, so the rules are
  identical (missing accent still counts, with the orange note). A
  correct answer gives a 2.6s turbo (1.55×) and races straight on; a
  missing-accent or wrong answer shows the correct spelling and waits
  for "Keep racing". Wrong answers carry no penalty. Words cycle
  through a shuffled pool of all selected lists, reshuffled when used
  up. `unlockSpeech()` fires synchronously on the Start race tap so
  the pit-stop audio can play.

**Arcade time for the two new games**: both count against the same
shared `game.arcadeMs` balance as the other three, with the same rules
(Play buttons disabled at zero, a game in progress may finish, New
game/Race again blocked until more time is earned). One deliberate
difference: **the arcade clock pauses while an Overhead Racer
spelling question is on screen**, so time spent spelling never eats
into play time. The Racer setup screen (picking lists) doesn't tick
the clock either, it only starts once the race view is showing. Both
of these were Claude's calls, flagged to Harry rather than assumed.

**Sound effects** (Web Audio, all synthesised in code, no audio files):
- **Brick Breaker**: a short, bouncy upward pitch slide (square wave,
  roughly 2.6× rise over 0.09s) on every brick hit, slightly higher for
  higher rows, throttled so multi-ball/lasers don't pile up
  (`sfxBrickBlip()`).
- **Overhead Racer engine hum**: a continuous soft sawtooth plus a sine
  an octave below, through a low-pass filter. Pitch follows the car's
  speed (about 55Hz idle to 150Hz at top speed, higher on turbo),
  quiet on the starting grid, silenced during pit-stop questions so the
  spoken word is clear, stopped when the race ends, the player leaves,
  or the app goes into the background (`updateEngineHum()`,
  `stopEngineHum()`).
- **Overhead Racer "star" chime**: a cheerful high two-tone chime (C6
  then G6, triangle wave) on every correct pit-stop spelling, with the
  banner now reading "Spelling star! ⭐ Turbo!" (`sfxStarChime()`).
- One shared `AudioContext`, created or resumed on any tap within the
  Arcade (browsers only allow sound after a tap).

**Technical note**: Brick Breaker and Overhead Racer draw to a
`<canvas>` on a `requestAnimationFrame` loop (`startArcadeRaf()`,
cancelled by `stopAllArcadeIntervals()`), rather than rebuilding the
DOM each tick like Snake and Maze Munch. The loop looks its canvas up
by id every frame, so a `render()` of the same view (e.g. the countdown
expiring) just swaps in a fresh canvas. The Sevling face is an HTML
overlay (`sevlingIcon(s,false,true)`) positioned over the canvas each
frame, not drawn into it. The Racer's track is pre-drawn once to an
offscreen canvas capped at about 6 million pixels (iOS canvas memory
limits).

**iPad canvas fix (v2026-09-30.7)**: on the iPad, after a while both
games showed trails behind the ball/cars and the Racer's track
vanished. Cause: iPad Safari can throw away what's drawn on an
offscreen canvas when the app is backgrounded or short of memory, so
the pre-drawn board/track came back blank, and since that image was
also what "cleared" each frame, old frames were never painted over.
Not caused by the auto-update check (that never reloads mid-game).
Fixes: each frame now paints the whole screen solid first; the
pre-drawn images are rebuilt whenever the app comes back to the front,
when Safari reports a lost canvas (`contextlost`/`contextrestored`),
and if a spot-check every 1.5s finds them blank; the on-screen canvas
element is reused between redraws (`adoptPersistentCanvas()`) instead
of a new one each render, to cut memory use.

**Power-up brick icons on iPad**: Safari paints emoji with the current
fill style, so icons drawn while the fill was the brick's rainbow
gradient disappeared into the brick. Icons are now drawn on a small
white badge with a solid fill.

**Free item rewards** (added on top of the coin-spend care system):
playing games can earn a Sevling a free happiness boost without
spending coins, via `awardSevlingItem()` (+15 happiness, capped 100,
random flavour-text item name). Goes to whichever Sevling is selected
as that game's character; if playing "classic" with no Sevling
selected, no item is awarded. Triggers: Snake, eating the 10th apple;
Connect 4, winning the match; Maze Munch, surviving 10 real seconds;
Brick Breaker, breaking the 25th brick in one game (once per game,
the Snake-style equivalent, deliberately reachable without clearing a
whole wall); Overhead Racer, finishing the race in any position (once
per race, a "reach the end" style trigger, so effort
on the spellings is always rewarded even if a computer car wins).

## 9. Known limitations / next up

- **iOS home screen icon not appearing** for Harry, root cause not yet
  confirmed (see §2).
- **Brick Breaker and Overhead Racer have only been tested in a
  desktop headless browser** (the iPad canvas-wipe fix was tested by
  simulating the wipe, not on a real iPad), plus simulated races/games to tune
  difficulty, not yet on a real iPhone/iPad. Worth a first check:
  touch dragging/holding feels right, pit-stop audio plays after the
  Start race tap, and the on-screen keyboard doesn't push the spelling
  overlay awkwardly on a small phone.
- **Game sounds are muted by the iPhone/iPad ring/silent switch**
  (that's how iOS treats Web Audio), unlike the spoken words. No
  in-app sound on/off toggle yet.
- **Power-up and pit-stop icons are emoji drawn into the canvas**, so
  they depend on the device's emoji font (fine on iOS, Android and
  Windows, may show as plain glyphs on some Linux machines).
- **Overhead Racer has a single track**, no track choice yet. The
  default is now Speedy, which needs real steering to win, so Rosie
  may want switching to Gentle (can be won without steering at all).
  The setting is remembered per profile once a race has been started,
  so any profile that raced before Speedy became the default will
  still be on Gentle until changed once on the setup screen.
- **Racer pit stops always every 10 seconds**, not configurable, and
  there's no "skip this word" button (a wrong answer just carries on,
  so a child is never stuck).
- **Touch/tap-zone controls not yet built for Snake and Maze Munch.**
  (Brick Breaker and Overhead Racer already use the board itself as
  the touch control.) Harry asked for
  semi-transparent directional arrows overlaid directly on the Snake
  and Maze Munch boards (large forgiving tap zones, not just the small
  arrow glyph), on top of the existing button pad, easier for younger
  hands. Not started, a clean CSS approach was scoped (four
  `clip-path` triangular zones splitting the board by its diagonals)
  but no code written yet.
- 30 of 34 Sevlings still have no real art, unlocked ones show the
  "Hatching soon" egg until files are uploaded (see §7). Puggle,
  Lotyl, Sword and Pepper have real artwork and idle animation.
- Sevling happiness only goes up, no decay over time.
- No in-app way to remove a family member's account (Firebase console
  only).
- Gemini Interactions API call doesn't pin an `Api-Revision` header,
  the same class of breaking change that's hit this project twice
  before could recur.
- The gift-box reveal animation (confetti, 3D lid-pop) exists only as
  a standalone prototype file given to Harry directly, never wired
  into the actual results screen.
- Sevling idle animation (blink/happy cycling, driven by happiness) is
  built for Puggle, Lotyl and Sword (see §7). The other 31 would need
  matching `-blink`/`-happy` art before it applies to them too.

## 10. Visual design — "Foil"

Y2K holographic sticker aesthetic, unchanged since established:
lavender-white background (`--bg:#F7F5FF`), ink navy text/borders
(`--ink:#221A3D`), brand pink/blue/gold/lime accents, a holographic
gradient (`--holo`) on the masthead and primary buttons, white sticker
cards with a solid 2px ink border and a hard 3px offset shadow (no
blur). 'Baloo 2' for headings, 'Inter' for body. Every input/select
forces plain black-on-white regardless of theme. Full details and the
Sevling illustration brief are unchanged from before, not repeated
here, ask if a fresh copy of that section is needed.
