# ChessLens: live board lens

Wave your phone's rear camera over a 2D chessboard on a monitor, tablet or laptop. ChessLens finds the 8×8 grid by itself on every camera frame, reads the pieces, and draws Stockfish's best move as a neon arrow on the board in the live camera view and on a mini-board. There is no calibration and no "hold still" wait. Everything runs in the browser in one file: `index.html`.

## Deploy on GitHub Pages (free)

1. Push this repo to GitHub.
2. Go to **Settings → Pages → Build and deployment**, choose **Deploy from a branch**, then pick the branch and `/ (root)`.
3. Open `https://<user>.github.io/<repo>/` on your phone. The camera needs HTTPS, and Pages provides it.

## Use

1. Tap **▶**. This starts the rear camera (`facingMode: { exact: "environment" }`) and the engine.
2. Point the camera at the board. The **8×8 LOCKED** chip turns neon green the moment the grid is found, and the board outline glows in the camera view.
3. The position is read and the best move appears on the board, usually within a few frames. The side to move follows the game (tracked moves, the site's last-move highlight).

Toolbar:
- **White / Black** sets the side to move. The FEN is rebuilt and sent to the engine in the same tap.
- **Flip** maps the squares the other way round (for boards shown from Black's side). The arrow is remapped through the new orientation straight away.
- **Board** shows or hides the mini-board.
- **▶ / ❚❚** starts or pauses. Pausing releases the camera.

Tap any square on the mini-board to fix a piece. **Start position**, **Rescan** and **Forget set** are in the same sheet. Fixes, start positions and tracked games teach the scanner the site's piece set; it is saved on the device.

URL flags:
- `?demo=1` forces the simulated feed.
- `?cam=any` accepts any camera, such as a desktop webcam.
- `?debug=1` exposes internals as `window.ChessLens`.

If no rear camera is available (desktop, permission denied, no HTTPS), the app falls back to a **simulated demo feed**: a chess site on a "monitor" filmed by a swaying hand-held "camera", playing Morphy's Opera Game through the same pipeline.

## How it works

Every camera frame goes through the pipeline once. There are no timers, debounces or cooldowns.

- **Frame pump (main thread):** the `requestAnimationFrame` loop takes each new camera frame (`requestVideoFrameCallback` where available) as an `ImageBitmap` scaled to 720 px on the long side, and hands it to the CV worker, with at most two frames in flight. Browsers without `OffscreenCanvas` in workers get the RGBA pixels instead. The camera view is painted from the same frames with `drawImage`, so the overlay and the image come from identical pixels.
- **Grid lock (CV worker), about 2 ms:**
  - A half-resolution grey image goes through a ChESS X-corner detector (Bennett & Lasenby), which fires only where four squares meet.
  - Corners are grown into a lattice from the strongest seeds with a least-squares homography. Sheared and half- or double-density lattices are corrected by basis reduction and parity splits.
  - Every candidate 7×7 window of inner corners is checked against the image: adjacent squares must alternate light/dark (≥ 82% of the 112 pairs) and the top-left square must be light, which is true for either orientation. A board that slipped by whole squares climbs back, because the true board always verifies better than a shifted copy of itself.
  - The previous frame only keeps the corner labelling continuous. Detection runs fresh on every frame, around the predicted board first and on the whole frame if that fails.
  - On a fresh lock, "piece gravity" (silhouettes sit low in their squares) fixes the orientation, including a phone held sideways.
- **Flatten:** the homography warps the board into a 320×320 image (40 px per square). That matches the pixels the board actually has in a 720 px frame; a larger target would only upsample.
- **Squares:**
  - Each square's colour is the median of its border ring. The piece silhouette is the pixels that differ from it, with outline gaps closed and holes filled.
  - Occupancy is the silhouette's central area.
  - Colour is the median brightness of the silhouette's body (lower half, outline peeled off, square-coloured pixels skipped). It is compared with an Otsu split across the board's pieces.
  - Type comes from shape features: area, height, top width, neck, head, left/right lean (knights), base and mass distribution. They are matched against three reference style profiles, or against the site's own piece set once learned. The board-level solver then enforces exactly one king per side, at most 8 pawns, no pawns on the back ranks, identical silhouettes voting as one group, and count priors.
  - The start position is recognised from occupancy alone and teaches the exact piece set.
- **Two-frame validator:** the 64-square layout string must be identical in two consecutive frames (≈ 33 ms at 60 fps) before anything is committed. Then:
  - One legal move from the known position (or one by the other side, or two plies) is matched on occupancy and colour. Piece types, castling rights, en passant and the side to move then come from the game (chess.js).
  - Otherwise the board is read as seen, with the turn taken from the site's last-move highlight when it shows one.
  - A layout where pieces only vanished (a piece in hand, a move animation) keeps the last position, because no legal move looks like that. It is read if it persists for about 1.5 s.
- **Engine:** two Stockfish.js 10 workers (single-threaded WASM, classical eval, about 370 KB). A new FEN always starts at once on an idle worker while the stale search is stopped (`stop` alone takes 150–220 ms to land mid-search). The search is `go depth 22 movetime 2500`, and the arrow streams from the `info` PV as soon as depth 6 is reached. That arrives as fast as `go depth 6` would (about 5–20 ms), and the search keeps improving it.
- **Overlay:** the four tracked corners are smoothed with a One-Euro filter and predicted forward by the pipeline latency (up to 70 ms), then mapped through a homography every display frame. Arrows and square highlights are drawn in true perspective.

Measured in headless Chromium on a desktop CPU (a phone is roughly 2–3× slower):
- CV takes about 10–12 ms per 720 px frame and runs at the camera or display rate.
- The first arrow comes about 35–90 ms after the camera frame that showed the new position.

## Limits

- Piece *types* on a set it hasn't learned yet are a best guess. The built-in profiles cover common Staunton-style sets well, but unusual sets can mix up king/queen/bishop. Showing a start position once, tracking a game from a known position, or fixing a square teaches the set.
- Keep the camera in focus. Heavy defocus (about 2 px of blur at 720 px) degrades piece types and then colours. Grid lock and occupancy hold up much longer.
- The whole board must be in the frame.
- Arrows, pop-ups or animations drawn over the board on the other screen can confuse the reading for as long as they are visible.

## Licenses

- Stockfish.js: GPLv3 (loaded at runtime from the CDN).
- chess.js: BSD-2-Clause (loaded at runtime from the CDN).
