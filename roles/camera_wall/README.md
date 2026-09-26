# camera_wall

Renders the static camera grid page (`index.html` + `cameras.json`) that Caddy
serves at `kamery.<domain>` (ADR-0011). `cameras.json` is public: per camera it
carries the stable `id` from `go2rtc_cameras`, the label, the go2rtc stream
names and display options — never hosts, paths or credentials.

## Page parameters

All optional. Without any of them the page is the automatic grid it has always
been: the column count is chosen so the 16:9 tiles come out as large as
possible, with the header and the "Pełny ekran" button.

| Parameter | Values | Effect |
|---|---|---|
| `uklad` | *(absent)* / `nvr6` | `nvr6` switches to the NVR-style layout below. Any other value keeps the automatic grid and logs a console warning. |
| `glowna` | a camera `id` | The camera in the large tile (`nvr6` only). Absent: the first camera in the list. Unknown id: the first camera and a console warning. |
| `glowna_strumien` | `sub` *(default)* / `main` | Which stream the large tile plays in the layout (`nvr6` only). `main` falls back to the substream for a camera without a main stream, as fullscreen does. Unknown value: `sub` and a console warning. |

Values are plain words (camera ids are `[a-z0-9_]`); in a URL query `+` means
a space, so do not use it.

Example: `https://kamery.<domain>/?uklad=nvr6&glowna=cam2&glowna_strumien=sub`

### `uklad=nvr6`

- A fixed 3×3 CSS grid that fills the whole viewport exactly. The large tile
  spans two columns and two rows in the top-left corner; the other cameras,
  in list order, fill the right column and then the bottom row.
- More than five other cameras: the extra ones are left out, with a console
  warning naming them. Fewer: the empty cells stay black.
- No header, margins, rounded corners or borders; a 2 px black gap. The small
  camera labels stay. Per-camera `fit` applies as in the grid.
- The automatic column fitting is off — the layout does not change with the
  window size.
- Clicking works exactly as in the grid: the tile goes fullscreen with the
  main stream; a second click or Esc returns to the layout, back on the
  tile's layout stream. The same holds for the large tile. Tiles never swap
  places.

The page only offers the layouts; which one a screen uses is decided by
whoever builds the URL that screen opens.
