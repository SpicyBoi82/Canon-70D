# 70D 5x punch-in (PUNCH5X + REC5X) - notes

## The idea
A punch-in during a RAW clip: wide shot (Canon 1x: whole sensor, 3x3 binned) and tight
shot (Canon 5x: 1:1 pixels) in one clip, at the same output size, with Canon's own
correct LiveView in both.

## Why two modules
* Canon's 5x is the real thing (sensor readout, AF, screen) but runs at 29.97 fps and
  delivers a different raw frame: 1x 1984 x 1250, 5x 3616 x 1346.
* PUNCH5X makes Canon's 5x run at the movie frame rate and lets the magnify key work while
  recording (stock mlv_lite blocks it; ML's set_lv_zoom refuses while recording, so the
  module asks for PROP_LV_DISPSIZE directly, as the key does).
* Stock mlv_lite ends the clip when the raw size changes (raw_lv_settings_still_valid every
  vsync). REC5X keeps going.

## History (what was tried first, CROP60 v0.14-v0.22)
* "3x punch-in, held size": Canon's 5x readout with the raw window forced back to the 1x
  size (1984 x 1250). Recording: perfect (no bad frames, no brightness jump) and stock
  mlv_lite never noticed. Screen: Canon's 5x display path reads the frame with 5x geometry
  -> streaked pink band + frozen rest. No register group fixed it (v0.15/v0.16).
* Apparent "fixes" (focus press, fast arrow presses) were the punch-in letting go: Canon
  pauses LiveView for those, the module took that as a fault (v0.17-v0.21 logs, confirmed
  with a key logger in v0.17K). Letting go = Canon's own 5x = correct screen, but the raw
  size changed and mlv_lite ended the clip.
* Conclusion: keep Canon's own 5x and teach the recorder to follow the size change.

## PUNCH5X: registers (only Canon's own 5x values are touched)
| where | register | Canon 5x | 23.976 | 25 |
|---|---|---|---|---|
| ENGIO | Timer B C0F06014, C0F06024 | 0x633 (+-3) | 0x7C1 | 0x770 |
| ENGIO | VD late C0F07064 | 0x5F6 (+-8) | +0x18E | +0x13D |
| ADTG dst 2 | 0x8173, 0x8179 PowerSave OFF (Gray) | 1540 (+-4) | 1938 | 1857 |
| ADTG dst 4 | 0x8179 frame length (Gray) | 1588 (+-4) | 1986 | 1905 |

Timer B = round(32 MHz / (Timer A + 1) / fps) - 1, Timer A in 5x = 0x29F. Same values
as CROP60 "24p crop (5x)" (used on the camera since V25/V26). 1x and 10x values never
match, so they pass untouched. Canon writes Timer B every frame: switching off gives
Canon's timing back at once; the ADTG values are re-written by Canon on the next zoom
(PUNCH5X does that zoom for you when not recording). Hooks: ADTG 0x2684C, ENGIO
0xFF2BC6C4 (patch_hook_function, packets copied - never edited in place). No CMOS
changes: rows and columns are Canon's (the AF box).

## REC5X: the change to mlv_lite (source/rec5x/rec5x_vs_mlv_lite.diff)
Active only with the option on, on a 70D, movie 1080p, digital dolly off.
1. Full-size capture buffer sized for the larger of the current frame and 3616 x 1348
   at the clip's bit depth (its frame slots start right after it - a 5x frame in a
   1x-sized buffer would overwrite recorded frames).
2. Each frame slot remembers the source width, pitch and crop offsets it was captured with;
   the compression / copy task uses those (stock code read raw_info and skip_x/y at
   compression time - wrong for the first frame after a switch).
3. Vsync: raw size from C0F06800/04 (as raw.c computes it on the 70D: columns x 8,
   height - 1). On a size change or a zoom change: frames are left out until the size is
   seen again (fast: a known 70D size that differs from the old one, seen twice; anything
   else - the old size while Canon still switches, odd sizes - 3 times), then the crop is
   centred in the new frame, then one more frame is left out (the next buffer compressed
   must hold the new size). raw_info is not touched during the clip (raw_update_params
   can't run in vsync and asserts 14-bit).
4. Our buffer is only handed to the EDMAC if Canon's current frame fits it.
5. Clean stop with a message if the output size doesn't fit the new frame, the frame
   doesn't fit the buffer, or the size doesn't settle within 5 s.
6. ML/LOGS/MLV_ZOOM.LOG: one block per clip with switches.
Off (or not 70D / not 1080p / dolly on) = stock mlv_lite behaviour.

## Crop positions (1832 wide)
1x: from x 144 (left margin), centred vertically in the 1222 picture lines.
5x: from x 964 of 3616 (centred in 3472 picture columns), y centred in 1318 lines.
70D margins (raw.c): left 144, top 28, right 8 in 1x / 0 in 5x.

## Field results
* CROP60 v0.23 + mlv_lite zoom-switch build (= safe mode): clip M22-1352, 1832 x 766,
  4 switches, exactly 5 frames left out each, crops 964,304 (5x) / 144,256 (1x),
  no stops. "Works very well."
* v1.0 fast mode: host tested (3 frames per clean switch, 5-6 with lagging registers,
  odd sizes never accepted); not yet on the camera.

* v1.0 on the camera (PUNCH5X.LOG): every 5x entry at Timer B 0x7C1 within ~0.1 s, two
  punch-ins during a recording. Flaw: stopping a clip in 5x -> LiveView pause while the
  recorder writes -> timing off/on + a needless 1x -> 5x hop. Fixed in v1.0.1 (timing follows
  the movie settings, not LiveView).

## Punch-in frame (v1.1)
Dotted rectangle in 1x. Size from REC5X (exported rec5x_get_res, weak-linked), position =
Canon's 5x window for the AF box (CMOS[7]/[11] work in 120-pixel steps: start column
floor((x - 1736)/120)*120, start row floor((y - 659)/120)*120, clamped), recording centred
in the 3472 x 1318 picture. If the frame and the 5x view disagree on the camera, the row /
column rule is what to adjust (send a photo of the 1x frame and the 5x view).

## Aiming on the AF box (v1.2)
Field v1.1: the recording was ~1000 columns left of the 5x screen. Canon's 5x readout window
(3472 columns) moves only when the AF box would leave it (CROP60 run D2): a centred box keeps
columns 0-3472, rows 1080-2398. The 5x screen shows the box (column 2736); REC5X had centred
its crop in the window (column 1736). Now PUNCH5X reads Canon's CMOS[7]/[11] (read-only hook)
and exports punch5x_get_window(); REC5X centres the crop on the AF box inside the window,
clamped at its edges, and follows the box during a clip. Limit: box centred + 1832 wide ->
the window ends 180 columns short (2736 + 916 > 3472), the crop sits 180 left. Exact up to
1472 wide, or with the box >= 180 left of centre.
Possible full fix: force CMOS[7] to a window centred on the box AND move Canon's display crop
(C0F09050/54, the YUV area inside the raw frame) by the same amount - needs a camera test.

## Align AF box (v1.3)
Field v1.2 log (1832x1030): "5x window seen: c7 c00 c11 289 = columns 0.., rows 1080..; AF box
2736,1824" - the model is right; the remaining offset was the window clamp (180 columns).
v1.3 moves the box (move_lv_afframe in 1x, ~35 ms) to the middle of the area that will be
recorded just before the zoom, and back after the punch-out. Box at 2556: right edge 3103 <
3472, so Canon keeps the window at 0 and the box-centred crop is exactly the dotted frame.

## Compression error (v1.4)
Field v1.3: "Compression error -3" (no JPEG from the encoder) at frames 219-221 in 5x, then an
ASSERT in compress_task. Only since v1.2, when the crop moved to the right end of Canon's window:
the 5x raw is 3616 wide but has 3464 picture columns - 8 non-picture columns at the right (ML's
raw.c says right skip 0 in zoom). REC5X now keeps 16 columns clear in 5x; PUNCH5X uses the same
limit. A frame that still fails is written as a NULL block (left out) instead of an ASSERT.

## Box placement quirk (v1.5)
Field v1.4: aiming and clip matched, no compression errors. But move_lv_afframe(-196, 0) moved
the box to 2540,1640 - the 70D takes 184 rows off every vertical AF-frame request (ML's
move_lv_afframe only compensates an X quirk of other models); +184 back did nothing, so the box
crept up each cycle (1824 -> 1640 -> 1456) and Canon's 5x rows followed (CMOS[11] 0x289 -> 0x226).
v1.5 places the box in a closed loop and learns the offset.
Also seen: 5x screen narrower than the recording - expected (Canon's 5x shows width/5 ~ 1094
sensor columns; the recording is 1832).

## Box placement, second round (v1.6)
Field v1.5: targets reached, 2-3 moves each: the camera snaps the box to 184-row positions
(asked 1824 -> 1640, asked 2008 -> 1824 or 2008), not a fixed offset. v1.6 asks half a step
(92 rows) high, one correction at most, never waits for LiveView (v1.5 spent 2 s retrying
while the recorder was stopping and punched in after the clip had ended).
Recording stop in that test: probably the card not keeping up (ISO 1600, lossless frames
bigger) - waiting for MLV_ZOOM.LOG.

## Centred 5x readout (v1.7)
Chart test v1.6: recording = Canon's 5x view, both ~300 columns left of the centre. With
Canon's readout (0-3472 for a centred box) a centred 1832 crop is impossible (needs 3652).
v1.7: CMOS[7] start S centred on the aim V (V=2736 -> 960 = 0xC80, CROPCTR's value) and the
box moved to V - S. Canon positions its 5x screen from the box inside the window it believes
(start 0), so the screen shows sensor S + (V - S) = V; REC5X keeps using Canon's own [7]
(win_c7 is recorded before the rewrite), so its crop lands on V too. If the screen does not
follow the readout on the camera, the recording is still on V (REC5X's maths only needs the
readout shift) - only the screen would be off.

## Field v1.7
User (2026-10-22 17:24): "looks good" - centred punch-in at 1832 wide. The logs sent were from
two later short sessions (no punch-in in them), so the "centre:" lines weren't captured.

## Logging off (PUNCH5X v1.8 + REC5X v1.5, 2026-10-22 17:50)
User asked to disable logging. Only change: PUNCH5X writes no ML/PUNCH5X.LOG (no .OLD rename;
lines still go to the RAM ring), REC5X writes no ML/LOGS/MLV_ZOOM.LOG. Compile switches
PUNCH5X_LOG / REC5X_LOG (default 0; -D...=1 for a debug build). Imports 42 -> 36 / 179 -> 178,
none added. Built here (xPack arm-none-eabi-gcc 13.2.1-1.1, ML 1a0600a; rebuilt v1.7 .text ==
shipped v1.7). SHA-256 PUNCH5X 31096231...4d92, REC5X 537fc0bf...d73a. Package 70D_PUNCH5X_v1_8.zip.
Not camera-tested (logic unchanged). Future field diagnosis needs a debug build with logs on.

## Ideas
* Repeat the last frame instead of leaving frames out (clip keeps real-time length).
* Use the 5x frame's full 3472 width when recording only in 5x.
