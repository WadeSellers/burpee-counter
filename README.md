# Burpee Counter

A fullscreen rep counter built for filming workout videos. Black screen, huge white text, designed for maximum legibility when captured on camera in vertical video (Instagram Reels / Facebook, channel: @wades_burpees). It counts a 100-burpee set by tap, spacebar, or voice, times the set, celebrates milestones, and ends with a follow/like/share end card for the video edit.

**Live app:** https://wadesellers.github.io/burpee-counter/

## What this is, technically

One page of vanilla HTML, CSS, and JavaScript (`index.html`), plus a small PWA shell: `manifest.webmanifest`, `sw.js` (service worker), and three PNG icons.

- No build step, no framework, no package.json, no external dependencies.
- No network calls except the browser's built-in speech recognition (Web Speech API).
- Deployed as-is via GitHub Pages from the repo root.
- Installable: on iPhone/Android use Share > Add to Home Screen. It launches fullscreen like a native app, and the service worker keeps it working offline (voice excepted).
- Settings persist per device via localStorage (keys prefixed `bc_`): animation style, voice on/off, day-one date, custom end card phrases.

## Controls and shortcuts

| Input | Action |
|---|---|
| `Space` or tap/click anywhere | Count one rep. Rep 1 auto-starts the clock. |
| `S` | Stop / resume the timer |
| `R` | Full reset (count and clock to zero) |
| `V` or the Voice button (top left) | Toggle voice recognition. Red dot = listening. |
| `E` | End-card mode: jump straight to the follow/like/share loop, for recording a clean outro clip |
| `O` or the Options button (bottom left) | Open the options menu: animation style, day counter, shout-out pool, end card phrases |
| Guide button (bottom right) | In-app reference for all of the above |

### Voice commands

With voice on, say the rep number ("fourteen") and the counter jumps to it. Say "stop" or "pause" to freeze the clock, "start" to start or resume it.

Voice needs Chrome (or another Chromium browser) and an internet connection; audio is processed by Google's speech service. Speak numbers clearly rather than whispering.

## Feature inventory

**Display.** Title, giant rep count, and a stopwatch (MM:SS.hundredths), all auto-sized by a measure-and-scale `fitText()` routine to fill ~97% of any screen. Text re-fits on every count change and window resize so digits never clip when crossing 9 to 10 or 99 to 100.

**Voice counting.** Uses `webkitSpeechRecognition`. The key design decision: spoken numbers SET the count to the number heard rather than incrementing it, so a missed detection self-heals on the next number spoken. A guard only accepts numbers greater than the current count and within +10, which filters out mishears. The parser handles digit strings, number words ("fourteen"), compound tens ("twenty one"), common homophones (won/to/for/ate), and "hundred". Continuous recognition auto-restarts when Chrome periodically fires `onend`.

**Timer.** Starts on rep 1 or on the "start" voice command. A manually stopped clock is never restarted by counting (spoken or tapped); only `S` or "start" resumes it. Auto-freezes at rep 100 to lock the final time.

**Screen wake lock.** While the clock is running or the end card loop is playing, the page holds a screen wake lock so a laptop or iPad left untouched during a set never dims or sleeps on camera. Released on reset, re-acquired automatically if the tab is backgrounded and returned to. Requires Chrome or Safari 16.4+; silently does nothing elsewhere.

**Milestone celebrations.** Milestones are detected by checking every integer crossed between the old and new count, because voice can jump the count past one (e.g. 48 to 52):

- Every 10: non-blocking gold flash on the number plus a radial spark burst. Taps keep working.
- 25 and 75: roughly 1.6s full-screen invert (white background, black number, "Quarter down" / "Three quarters") with confetti. The overlay uses `pointer-events: none`, so taps during it still count reps.
- 50: roughly 2.5s bigger invert takeover, "Halfway" — followed by a random follower shout-out card (~2.2s) when the shout-out pool has handles.
- 100: sticky finale. Black screen, gold "100", "Done", the locked final time, heavy confetti. Stays until any tap or keypress.

**Post-100 CTA loop.** Four seconds after the finale, an endless attention loop starts (the video end card): FOLLOW (letter wave), LIKE (red beating heart), SHARE THIS (bounce), 100 EVERY DAY (slam), SEE YOU TOMORROW (wiggle). Each phrase flips the background between black and white, fires confetti, and auto-fits to the screen width. The final time stays pinned. Runs until dismissed.

**End-card mode (`E`).** Jumps straight into that CTA loop with zero reps, so a clean outro clip can be screen-recorded once and reused in every edit.

**Count animations.** Each rep lands with one of nine animations: zoom pop, sky drop, spin, jelly, neon flicker, flip board, punch, odometer, slot-machine scramble. Jelly is the default; Random mode rotates through all of them, never the same one twice in a row. The choice is remembered per device.

**Day badge.** A gold "DAY N" badge above the title, computed from a day-one date set in Options (default 2026-07-02). Alternatively, type today's day number directly: the start date is recalibrated so today equals that day, and the two fields stay in sync. The badge rolls over at local midnight, feeds the rep-100 finale label ("Day N done"), and is available to end card phrases via the `{day}` token. Clearing the date hides the badge and restores the original layout proportions.

**Shout-out pool.** A list of follower handles in Options, one per line. When the pool is non-empty, the counter picks a handle at random (uppercased, `@` added if missing) in two places: a dedicated shout-out card that chains right after the rep-50 "Halfway" takeover, and a "SHOUT OUT @HANDLE" phrase inserted into the default end card loop. Custom end card phrases can place the picked handle anywhere via the `{follower}` token; lines using `{follower}` are skipped when the pool is empty. One handle is picked per end-card run, and a fresh one for each rep-50 moment.

**Custom end card phrases.** The Options menu has a textarea, one phrase per line, that replaces the default FOLLOW/LIKE/SHARE loop (useful for day numbers and follower shout-outs, e.g. "THANKS @NANCY462"). `{day}` is substituted with the current day number, `{follower}` with the randomly picked shout-out handle, any line containing LIKE gets the beating heart treatment, and other lines rotate through the loop's animation set. Empty textarea = default loop (which auto-gains a SHOUT OUT line when the pool has handles). Persisted per device.

**Options menu (`O`).** Animation style, day-one date, and end card phrases. Every animation chip contains a live mini number that performs its animation on a loop while the menu is open, driven by the same engine as the big number (with chip-scaled keyframe overrides for the moves that travel in screen units). The demo loop starts when the menu opens and stops when it closes. While the menu's text fields are focused, global shortcuts are suspended so typing a phrase can't fire reps or resets.

## Architecture notes

- **Animation engine.** Generalized across elements: each animated element has a state object `{node, gen, cls, useFit}`. Cancellation works by bumping the generation counter; pending `setTimeout` steps from older runs check the generation and die silently. This is why fast tapping can never lose counts or strand the display mid-animation. The big number (`mainAnim`) and all ten option-chip previews share this one engine.
- **`setCount()` is the single entry point** for count changes. Taps and voice both route through it. It starts the clock if needed, animates the new value, then runs milestone detection against the previous value.
- **Overlay stack.** While the Guide or Options overlay is open, any key closes it (prevents accidental reps or resets while reading). The sticky 100 celebration dismisses on any tap or key. Timed celebrations pass pointer events through so counting continues.
- **Reset (`R`) tears everything down:** celebrations, CTA loop, in-flight count animations, clock, and count.

## Known constraints

- Voice recognition requires Chrome/Chromium plus an internet connection, and works poorly with whispering or heavy breathing.
- The rep count itself intentionally resets on reload; only settings persist.
- Mic permission on a `file://` URL is flaky in some browsers; the https URL from GitHub Pages avoids that.
- The service worker is network-first: online visitors always get the newest version, and the cache is only a fallback for offline use.
