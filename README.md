# ChessLens: live board lens

Point your phone's rear camera at a 2D chessboard on a monitor, tablet or laptop. ChessLens finds the 8×8 grid by itself, takes a snapshot of the position the moment it sees the board, reads the pieces, and draws Stockfish's best move as a neon arrow on the board in the camera view and on a mini-board. It keeps watching in the background, so a move or a completely new position is picked up while you keep pointing. It is built for a shaky hand: there is no calibration and no "hold still" wait. Everything runs in the browser in one file: `index.html`.

## Deploy on GitHub Pages (free)

1. Push this repo to GitHub.
2. Go to **Settings → Pages → Build and deployment**, choose **Deploy from a branch**, then pick the branch and `/ (root)`.
3. Open `https://<user>.github.io/<repo>/` on your phone. The camera needs HTTPS, and Pages provides it.

## Use

1. Tap **▶**. This starts the rear camera (`facingMode: { exact: "environment" }`) and the engine.
2. Point the camera at the board. Hand-held is fine. The **8×8 LOCKED** chip turns neon green the moment the grid is found.
3. The board flashes white: that's the snapshot. It is read at once and the best move appears on the board in the camera view.
4. Keep pointing. Every later frame is checked in the background. A move is followed through the game, and a different position (another game, a jump in the move list) is read as a new snapshot. The side to move follows the game (tracked moves, the site's last-move highlight).

Toolbar:
- **White / Black** sets the side to move. The FEN is rebuilt and sent to the engine in the same tap.
- **Flip** maps the squares the other way round (for boards shown from Black's side). The arrow is remapped through the new orientation straight away.
- **Board** shows or hides the mini-board.
- **▶ / ❚❚** starts or pauses. Pausing releases the camera.

Tap any square on the mini-board to fix a piece. **Start position**, **Rescan** and **Forget set** are in the same sheet. Fixes, start positions and tracked games teach the scanner the site's piece set; it is saved on the device.

URL flags:
- `?demo=1` forces the simulated feed. `&shake=0…3` sets how shaky its hand is (default 1, a normal hand; 2 is very shaky, with heavy motion blur).
- `?cam=any` accepts any camera, such as a desktop webcam.
- `?debug=1` exposes internals as `window.ChessLens`. With it, `&cfg=KEY:value,…` overrides numeric settings from `CFG` for field tuning (e.g. `&cfg=VOTE_SHARE:0.7`).

If no rear camera is available (desktop, permission denied, no HTTPS), the app falls back to a **simulated demo feed**: a chess site on a "monitor" filmed by a shaky hand-held "camera" (tremor, jerks and the motion blur they cause), playing Morphy's Opera Game through the same pipeline.

## How it works

Every camera frame goes through the pipeline once. Nothing waits for the camera to hold still: a hand-held phone never does.

- **Frame pump (main thread):** the `requestAnimationFrame` loop takes each new camera frame (`requestVideoFrameCallback` where available) as an `ImageBitmap` scaled to 720 px on the long side, and hands it to the CV worker, with at most two frames in flight. Browsers without `OffscreenCanvas` in workers get the RGBA pixels instead.
- **Frame-synchronous view:** a shaking hand moves the board several pixels per frame, and a frame's analysis lands 40–70 ms after it was taken. An overlay drawn on the newest frame would always trail the board. So each frame handed to the CV worker is also copied as it will be shown, and it goes on screen when its result arrives, together with that frame's own board corners. The arrow sits on the exact pixels it was computed from. The view runs at the analysis rate, one CV pass behind the camera (about the lag of a normal camera preview).
- **Grid lock (CV worker), about 2 ms:**
  - A half-resolution grey image goes through a ChESS X-corner detector (Bennett & Lasenby), which fires only where four squares meet.
  - Corners are grown into a lattice from the strongest seeds with a least-squares homography. Sheared and half- or double-density lattices are corrected by basis reduction and parity splits.
  - Every candidate 7×7 window of inner corners is checked against the image: adjacent squares must alternate light/dark (≥ 82% of the 112 pairs) and the top-left square must be light, which is true for either orientation. A board that slipped by whole squares climbs back, because the true board always verifies better than a shifted copy of itself.
  - The previous frame only keeps the corner labelling continuous. Detection runs fresh on every frame, around the predicted board first and on the whole frame if that fails.
  - On a fresh lock, the orientation that matches the position already being read is kept (a shaky hand drops the lock now and then). On a new board, "piece gravity" (silhouettes sit low in their squares) fixes it, including a phone held sideways.
  - Motion blur can wipe out the X-corners while the squares' mid-tones survive. When the lock drops, the predicted board is slid over the frame (±0.6 of a square, coarse to fine) to where the light/dark pattern lines up best. Such frames only move the overlay; they are too blurred to read pieces from.
- **Flatten:** the homography warps the board into a 320×320 image (40 px per square). That matches the pixels the board actually has in a 720 px frame; a larger target would only upsample.
- **Squares:**
  - Each square's colour is the median of its border ring. The piece silhouette is the pixels that differ from it, with outline gaps closed and holes filled.
  - Occupancy is the silhouette's central area.
  - Colour is the median brightness of the silhouette's body (lower half, outline peeled off, square-coloured pixels skipped). It is compared with an Otsu split across the board's pieces.
  - Type comes from shape features: area, height, top width, neck, head, left/right lean (knights), base and mass distribution. They are matched against three reference style profiles, or against the site's own piece set once learned. The board-level solver then enforces exactly one king per side, at most 8 pawns, no pawns on the back ranks, identical silhouettes voting as one group, and count priors.
  - The start position is recognised from occupancy alone and teaches the exact piece set.
  - **Sharpness:** the edges between empty squares are ideal steps. Their steepest 1 px step, relative to the whole step, is about 1 when sharp and drops with motion blur or defocus. The worse of the two directions counts, because motion blur smears one direction only. A frame's weight is its sharpness relative to the sharpest recent frames, squared.
- **Snapshot, then background watch:**
  - *Snapshot:* the first sharp locked frame after the board is found (or the sharpest one within 160 ms) is read and sent to the engine straight away. The board flashes white in the view.
  - *Per-square vote:* after that, every locked frame votes on each square, weighted by its sharpness. Occupancy and colour votes fade in about 110 ms. A square only changes when 60% of the recent weight agrees, so blurred or half-locked frames can't flip it, and a dropped lock doesn't lose what was seen. Piece-type evidence builds up for as long as a square's occupant stays the same (about 3 s), and a type only switches when another has 1.5× the evidence. Exactly one king per side is enforced.
  - *Commit whole:* a change is committed once no other square is still mid-change, so a move or a new position lands in one piece rather than square by square.
  - *Matching:* one legal move from the known position (or one by the other side, or two plies) is matched on occupancy and colour. Piece types, castling rights, en passant and the side to move then come from the game (chess.js).
  - *Refinements:* for 0.9 s after a snapshot, differences are treated as corrections of it, not moves.
  - *New positions:* five or more squares changing with no legal explanation is a new position (another game, a jump in the move list). It is read like a fresh snapshot, with piece types taken from the latest frames only.
  - *Small unexplained changes:* a change of one to four squares that no legal move explains must persist for 0.45 s before it counts (a misread otherwise). A layout where pieces only vanished (a piece in hand, a move animation) keeps the last position for up to 1.5 s.
- **Engine:** two Stockfish.js 10 workers (single-threaded WASM, classical eval, about 370 KB). A new FEN always starts at once on an idle worker while the stale search is stopped (`stop` alone takes 150–220 ms to land mid-search). The search is `go depth 22 movetime 2500`, and the arrow streams from the `info` PV as soon as depth 6 is reached. That arrives as fast as `go depth 6` would (about 5–20 ms), and the search keeps improving it.
- **Overlay:** the board corners of the frame on screen are mapped through a homography, and arrows and square highlights are drawn in true perspective. They come from that very frame, so there is no smoothing or prediction to lag behind the hand. Only sub-pixel noise on a still camera is held back, so the grid doesn't shimmer.

Measured in headless Chromium on a desktop CPU with the demo feed (a phone is roughly 2–3× slower). "Very shaky" is `shake=2`: tremor, jerks and 30 ms of motion blur.
- CV takes about 15–20 ms per 720 px frame and runs at the camera or display rate.
- First snapshot: about 0.45–0.65 s after tapping ▶, camera start-up included.
- A move in a tracked game: about 0.1 s median (0.4 s worst) from the frame that shows it to the new position, at every shake level.
- A new position (a jump in the game): about 0.33 s median to the exact position. None missed, including when very shaky.
- Overlay error, as a fraction of a square: 0.02 steady, 0.03 normal hand, 0.07 median (0.13 at p90) very shaky.

## Limits

- Piece *types* on a set it hasn't learned yet are a best guess. The built-in profiles cover common Staunton-style sets well, but unusual sets can mix up king/queen/bishop. Showing a start position once, tracking a game from a known position, or fixing a square teaches the set.
- Keep the camera in focus. Heavy defocus (about 2 px of blur at 720 px) degrades piece types and then colours. Grid lock and occupancy hold up much longer. Motion blur from a shaking hand is handled: blurred frames count for little and sharp ones decide.
- On a piece set it hasn't learned, the snapshot's piece types can be off at first. They are refined over the next second as sharper frames come in.
- The whole board must be in the frame.
- Arrows, pop-ups or animations drawn over the board on the other screen can confuse the reading for as long as they are visible.

## Licenses

- Stockfish.js: GPLv3 (loaded at runtime from the CDN).
- chess.js: BSD-2-Clause (loaded at runtime from the CDN).
