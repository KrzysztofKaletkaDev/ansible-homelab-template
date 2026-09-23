# 11. LAN-only live camera view via go2rtc behind Caddy

## Status

Accepted

## Context

This ADR covers only the live camera view — a browser page at
`kamery.{{ domain_name }}` that shows the IP cameras in a grid on laptops,
phones and the TV, served from the LAN — and how it is wired: go2rtc as the
restreamer, Caddy in front of it, and the one camera that needs WebRTC. It
does not cover recording, which the QNAP NAS (QVR) already does by pulling
directly from the cameras, and it does not change how any other service is
exposed.

The cameras speak RTSP, which browsers cannot play. Something has to turn
RTSP into a browser-playable transport. go2rtc does that with no transcoding
for H.264 cameras (it repackages into MSE/fMP4 over a WebSocket) and ships a
small player (`video-stream.js` / `video-rtc.js`). Everything below was
tested by hand first — the go2rtc container, the page on a laptop and a
phone, and the CPU load on the production VM — before being turned into
roles.

Two properties of go2rtc shape the design:

- `GET /api/streams` returns every stream's source URL, and for RTSP cameras
  that URL contains the camera username and password in clear text.
- `/api/config` can read and rewrite the running configuration, including
  `exec:` sources — i.e. run arbitrary commands inside the container.

One camera, the Eurolook, is awkward: it sends H.265 on both its main and
sub stream, PCMA audio, and a keyframe only every ~7–9 seconds. The codec
cannot be changed without a physical factory reset.

## Decision

- **Viewing is separate from recording.** go2rtc only shows the streams;
  QVR keeps recording straight from the cameras and does not go through
  go2rtc. The live view is a convenience layer, and a failure in it (a
  crashed container, a bad config, an upgrade gone wrong) must never cost
  recordings.
- **One hostname, MSE through Caddy.** The page, the player scripts and the
  go2rtc WebSocket all live under `kamery.{{ domain_name }}`, behind Caddy's
  existing wildcard certificate and a `basic_auth` login. Same-origin means
  go2rtc needs no `api.origin` setting, and none is set — `origin: "*"` would
  let any web page open in a LAN browser read `/api/streams`. go2rtc's API
  port (1984) and its built-in RTSP server (8554) are not published on the
  host; Caddy reaches the API over the `caddy-ingress` Docker network.
- **Caddy forwards an allowlist of paths, not a blocklist.** Only `/api/ws`,
  `/video-stream.js` and `/video-rtc.js` reach go2rtc; only `/`,
  `/index.html` and `/cameras.json` are served from disk; everything else
  under the hostname gets 403. The player needs nothing more, and a
  blocklist would have to anticipate every current and future API endpoint
  that leaks credentials or accepts config. The player scripts are proxied
  from go2rtc rather than versioned in this repo, so they always match the
  running go2rtc version. The strict portfolio CSP is not applied to this
  handler ([ADR-0010](0010-security-headers-scoped-to-static-site.md)): the
  player injects inline styles.
- **LAN only.** `kamery.*` has a Blocky `customDNS` entry and no
  `cloudflared` ingress entry — the public ingress list stays explicit and
  narrow ([ADR-0009](0009-www-to-apex-redirect-in-caddy.md)).
- **WebRTC exception for the Eurolook.** Its substream is transcoded to
  H.264/AAC by ffmpeg inside go2rtc and played over WebRTC. Measured on the
  production host (Intel N5095): about 9% of one core, in real time. The
  same transcoded stream over MSE stuttered; over WebRTC it plays smoothly.
  Every other camera stays on MSE with no transcoding.
- **Two roles, split around `caddy`.** `camera_wall` renders the static
  page into `{{ camera_wall_dir }}/site` and runs *before* `caddy`, because
  Caddy bind-mounts that directory and Docker silently creates a missing
  bind-mount source as an empty root-owned directory — the same ordering
  constraint as the portfolio site
  ([ADR-0003](0003-caddy-file-server-for-static-site.md)). `go2rtc` runs
  *after* `caddy`, because it joins `caddy-ingress` as an external network
  that `caddy` creates. Both roles read one `go2rtc_cameras` list; only
  `go2rtc.yaml` gets credentials, while the public `cameras.json` carries
  labels, stream names and display options only.
- **CPU cap of 1.0 on the go2rtc container.** The VM has 2 vCPU and also
  runs Blocky, the DNS resolver for the whole house. The measured transcode
  is small, but a stuck or runaway ffmpeg must not be able to starve DNS.

## Consequences

- A broken or stopped go2rtc blacks out the camera wall and nothing else;
  QVR recordings are unaffected.
- The Eurolook needs UDP 8555 from the client segment to the servers
  segment. That is an explicit hole in the segment boundary and belongs to
  the network repo (`ansible-network-template`), as a separate change with
  its own anti-lockout procedure. Until it lands, the five MSE cameras work
  and the Eurolook tile stays black — expected, not a bug. firewalld on the
  host opens 8555/udp for consistency, but Docker-published ports largely
  bypass firewalld; the router rule is the real boundary.
- The Eurolook shows its first frame after about 10 seconds (ffmpeg start
  plus waiting for a keyframe), and every viewing session pays for a live
  transcode. With the 1.0 CPU cap, many simultaneous Eurolook viewers would
  hit the cap before they hit DNS.
- The transcode source must be written with the camera's RTSP URL inline
  (`ffmpeg:rtsp://...#video=h264#audio=aac`). In go2rtc 1.9.14 the form that
  refers to another stream by name (`ffmpeg:<stream>#video=h264`) never
  starts ffmpeg for a video transcode.
- The built-in RTSP server (8554) must stay enabled even though nothing
  outside the container uses it: ffmpeg publishes the transcoded stream
  back into go2rtc through `rtsp://127.0.0.1:8554/...`.
- Camera credentials end up inside RTSP URLs, so the role percent-encodes
  them; `go2rtc.yaml` and the rendered `Caddyfile` (which now holds the
  `basic_auth` hash) are written with mode `0600`.
- The go2rtc config directory is mounted read-only, as a directory rather
  than a single file (a file bind mount keeps pointing at the old inode after
  the template module replaces the file). The allowlist already keeps the
  go2rtc web UI and `/api/config` out of reach through Caddy; the read-only
  mount is the second layer, so a config change made from inside the
  `caddy-ingress` network still cannot persist. The repo is the only source
  of truth for the camera config.
