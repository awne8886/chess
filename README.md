# ChessLens: screen-board scanner

Point your phone's rear camera at a 2D chessboard on another monitor, tablet or laptop. ChessLens maps the board to a grid, keeps track of the position, and draws Stockfish's best move as neon SVG arrows, both on a small digital board and in perspective over the camera view. Everything runs in the browser in a single file: `index.html`.

## Deploy on GitHub Pages (free)

1. Push this repo to GitHub.
2. Go to **Settings → Pages → Build and deployment**, choose **Deploy from a branch**, then pick the branch and `/ (root)`.
3. Open `https://<user>.github.io/<repo>/` on your phone. The camera needs HTTPS, and Pages provides it.

## Use

1. Tap **▶**. This starts the rear camera (`facingMode: { exact: "environment" }`), audio and the engine.
2. Frame the board in the upper half of the view (the left side in landscape).
3. Drag the 4 corner handles onto the board's outer corners. A magnifier appears while you drag, and "Grid match" shows how well the grid lines up. Tap **Lock grid**.
   - If the board shows the **start position** when you lock, the app learns that site's piece set and orientation. This gives the best accuracy.
4. When a move lands and the board stays still for 1 s, the position updates. Detected moves also switch the side to move.

Toolbar:
- **White / Black** sets the side to move by hand.
- **Flip** rotates the square mapping by 180°.
- **Grid** re-opens calibration.
- **▶ / ❚❚** starts or pauses. Pausing releases the camera.

Tap any square on the digital board to correct a piece. Each correction also teaches the scanner. **Rescan now** forces a full re-read of the board.

URL flags:
- `?demo=1` forces the simulated feed.
- `?cam=any` accepts any camera, such as a desktop webcam.
- `?debug=1` exposes internals as `window.ChessLens`.

If no rear camera is available (desktop, permission denied, no HTTPS), the app falls back to a **simulated demo feed**. The demo loops Morphy's Opera Game through the same CV pipeline.

## How it works

- **Loop:** `requestAnimationFrame`, phase-locked to one CV pass every 400 ms.
- **Warp:** a 4-corner homography, applied as a piecewise-affine warp. Each half-cell gets `ctx.transform()` with the local Jacobian, clipped to its cell. The result is a flat 320×320 grid.
- **Debounce:** a per-cell block-luminance signature, with global exposure drift removed. The board is read only after a spike followed by 1.0 s of stillness. As a safety net, a board that has drifted from the last read also counts as a spike.
- **Recognition:**
  - Occupancy is judged against each cell's own border colour, so it copes with highlights and themes.
  - Colour comes from an eroded "dark core" feature.
  - Piece type comes from 15×15 silhouette templates learned from the start position. Rendered glyphs are the fallback when nothing has been learned.
  - Moves are tracked by scoring every legal 1–2 ply continuation (chess.js) against the observation. This covers captures, castling, en passant and promotion. A full template re-sync runs only when no legal continuation fits.
- **Engine:** Stockfish.js 10.0.2. It is single-threaded WASM with classical eval and no NNUE (~370 KB), loaded from jsDelivr (unpkg as fallback) into a blob Web Worker. Each search is `go depth 14 movetime 250`. While the position is unchanged, a few more 250 ms passes deepen the result using the warm hash.

## Limits

- The calibration corners are fixed to the camera frame. Use a stand or hold steady; if the phone moves, tap **Grid**.
- Arrows, animations or pop-ups drawn over the board on the other screen can confuse detection.
- If you start mid-game without having calibrated on a start position, the first read uses generic templates. Fix it by tapping squares.

## Licenses

- Stockfish.js: GPLv3 (loaded at runtime from the CDN).
- chess.js: BSD-2-Clause (loaded at runtime from the CDN).
