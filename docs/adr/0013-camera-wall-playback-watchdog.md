# 13. Page-side playback watchdog for the camera wall

## Status

Accepted

## Context

This ADR covers only how the camera-wall page (`roles/camera_wall/files/index.html`,
[ADR-0011](0011-live-camera-view-via-go2rtc.md)) recovers a tile whose
playback has degraded while its connection is still alive. It does not change
go2rtc, Caddy, the streams a tile plays, the layouts, or what a click does.

On 2026-11-23 the wall kiosk (`ansible-kiosk-template`) was showing the
`uklad=nvr6` layout with the large tile on the main stream of a VIGI camera
(H.264, 2560×1440), over Wi-Fi at -73 dBm on 5 GHz. After a short hiccup at
about 12:16, the large tile and the tile of the second VIGI camera stuttered
from about 12:20 to 12:27. go2rtc's outbound traffic over that time
(cAdvisor, `container_network_transmit_bytes_total`) was flat: the server
kept sending. The picture only recovered after someone clicked the large
tile into fullscreen and back, which replaces the tile's `<video-stream>`
element with a new one (`setStream`). Until this change, the first
fullscreen toggle after page load rebuilt *every* tile, not only the clicked
one (`tile._current` was not set when a tile was created), which is likely
why the second VIGI tile recovered too.

The player is go2rtc's own `www/video-rtc.js` (1.9.14, served from go2rtc,
not versioned here). In MSE mode, after every `updateend` (lines ~467–479) it:

- trims the source buffer to the last ~5 seconds;
- jumps `currentTime` forward when playback is more than 5 s behind the end
  of the buffer;
- sets `playbackRate = gap` (seconds between `currentTime` and the buffer
  end, at least 0.1), so it speeds up to catch up.

It reconnects only when the `<video>` element raises an error (the handler
closes the WebSocket) or when the WebSocket closes (`onclose`, at most once
per 15 s). Nothing in the player looks at whether playback is actually
healthy. A player that keeps its connection but stops advancing, lags
between 3 and 5 seconds behind (never far enough for the jump), or drops a
large share of decoded frames stays in that state indefinitely.

## Decision

- **A watchdog in the page, not a patched player.** The player scripts are
  proxied from the running go2rtc on purpose, so they always match its
  version (ADR-0011). The page already knows how to rebuild a tile; the
  watchdog does the same thing the manual click did.
- **Every 2 s, every tile** — including the tile in fullscreen — reads the
  tile's `<video>` (`tile.querySelector('video')`) and restarts the tile
  when one of these holds:
  1. **stall** — `currentTime` has not increased for **≥ 6 s** (any mode);
  2. **lag** — any tile not playing WebRTC: `buffered.end(last) -
     currentTime` **> 3 s** without a break for **≥ 10 s**;
  3. **dropped** — from `getVideoPlaybackQuality()`, the increase in dropped
     frames divided by the increase in total frames over a **10 s** window
     is **> 0.2**, with the total increase above zero.
- **Restart** is `setStream(tile, tile._current, cam.mode)` — the stream the
  tile is currently meant to play, so a fullscreen tile stays on its main
  stream. It records the creation time and logs `console.warn` with the
  camera id, the reason (`stall`, `lag`, `dropped`) and the measured value.
- **At most one restart per tile per 30 s.**
- **15 s grace period** after every (re)creation of a `<video-stream>`, in
  which nothing is measured and the baselines are reset.
- **Hidden page: no checks.** The player disconnects by itself when
  `document.hidden`; when the page becomes visible again it reconnects, so
  the watchdog treats that as a new creation and starts a new grace period.
- **Also fixed:** `tile._current` is set when the tile is created, so a
  fullscreen toggle rebuilds only the tiles whose stream actually changes.

Why these thresholds:

- **2 s tick** — several samples inside every window, negligible cost.
- **15 s grace** — the Eurolook (WebRTC, transcoded) shows its first frame
  after about 10 s (ADR-0011); any stream needs its first keyframe before
  `currentTime` moves at all.
- **6 s stall** — longer than the ~5 s the player keeps buffered, so a 6 s
  freeze means the buffer ran dry and nothing new played, not a short
  network wobble that the buffer absorbs.
- **3 s lag for 10 s** — with `playbackRate = gap` a healthy player 3 s
  behind plays at 3× and catches up within seconds. Staying above 3 s for
  10 s means it cannot keep up, and the player's own jump does not fire
  below 5 s. The check is skipped for WebRTC, where there is no buffer to
  compare against. A tile counts as WebRTC only when `video.srcObject` is a
  `MediaStream`. An empty `srcObject` does not mean MSE: in `onmse`
  (lines ~424–443) the player puts a blob URL in `src` when only
  `MediaSource` exists, but assigns the `MediaSource` itself to `srcObject`
  when `ManagedMediaSource` is available. The `lag` check covers both.
- **20 % dropped over 10 s** — a healthy decoder drops close to nothing; a
  fifth of frames lost is visible stutter. The 10 s window keeps a single
  catch-up burst from counting.
- **30 s between restarts** — longer than grace plus the stall threshold
  (21 s), so a tile that cannot be fixed is retried twice a minute instead of
  on every check.

## Consequences

- A degraded tile recovers by itself within about 6–12 s of the condition
  starting (later if the tile itself was created less than 15 s earlier),
  without anyone touching the screen. Each restart is visible: the
  tile is black until the new connection's first keyframe.
- **What it does not fix:** a link that permanently cannot carry the stream
  (for example the 2560×1440 main stream over a weak Wi-Fi link). The
  watchdog then restarts the tile every 30 s — every restart briefly helps
  and degrades again. The fix for that is the link (cable, better signal) or
  the layout URL (`glowna_strumien=sub`), not the watchdog. It also cannot
  help when the problem is on the server side (camera offline, go2rtc
  down); there it only adds a reconnect every 30 s next to the player's own.
- Restarts are visible only in the browser console of the screen that made
  them. The kiosk does not ship its console anywhere, so a tile restarting
  every 30 s shows up as a periodically black tile, not as a metric.
- The thresholds are constants at the top of the page script. Lowering them
  trades faster recovery for false restarts on healthy streams; any change
  should be checked against a healthy wall over several minutes (zero
  restarts expected).
- The watchdog depends on `video-rtc.js` keeping one `<video>` inside
  `<video-stream>` and on its 1.9.14 behaviour (5 s buffer, `playbackRate =
  gap`). After a go2rtc upgrade, re-check those lines before trusting the
  thresholds.
