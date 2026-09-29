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
4. The screen switches to the **engine view**: a big digital board with Stockfish's arrows and the eval, plus a small camera preview zoomed onto the board. Tap **✕** on the preview (or **Hide cam**) to hide the camera and keep only the engine view. Scanning continues while it's hidden.
5. A move is picked up about 0.4 s after the piece lands. The side to move follows the game automatically, including from the site's own last-move highlight.

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

- **Loop:** `requestAnimationFrame`, phase-locked to one CV pass every 120 ms (~8 Hz, a few ms each).
- **Warp:** a 4-corner homography, applied as a piecewise-affine warp. Each half-cell gets `ctx.transform()` with the local Jacobian, clipped to its cell. The result is a flat 320×320 grid.
- **Debounce:** a per-cell block-luminance signature, with global exposure drift removed. The board is read after a change followed by 0.2 s of stillness. As a safety net, a board that has drifted from the last read also counts as a change.
- **Recognition:**
  - Each piece is a filled silhouette: pixels that differ from the cell's own border colour, with outline gaps closed and holes filled. An over-exposed white piece that blends into a light square still counts through its outline.
  - Colour is the median brightness inside the silhouette compared with the midpoint of the two square colours in the same frame. White pieces sit above it and black pieces below, whatever the exposure or theme.
  - Piece type comes from 19×19 silhouette templates. They are learned from the start position and refined while a trusted game is tracked, then saved per device. Identical-looking pieces are grouped and vote together; generic glyph shapes are the fallback.
  - Moves are tracked by scoring every legal 1–2 ply continuation (chess.js) against the observation. This covers captures, castling, en passant and promotion.
  - When the site marks the last move, a move only counts once the highlight shows it. A piece still being dragged is never mistaken for a move, and the highlight also fixes the side to move.
  - A full re-read (for a jump, a takeback or a new game) must give a legal position (≤16 pieces and ≤8 pawns per side, one king each) twice in a row before it replaces the current one.
- **Exposure:** on Chrome for Android, exposure compensation is stepped down automatically when the board's light squares clip to white.
- **Engine:** Stockfish.js 10.0.2. It is single-threaded WASM with classical eval and no NNUE (~370 KB), loaded from jsDelivr (unpkg as fallback) into a blob Web Worker. Each search is `go depth 14 movetime 250`. While the position is unchanged, a few more 250 ms passes deepen the result using the warm hash.

## Limits

- The calibration corners are fixed to the camera frame. Use a stand or hold steady; if the phone moves, tap **Grid**.
- Arrows, animations or pop-ups drawn over the board on the other screen can confuse detection.
- If you start mid-game on a site whose piece set it hasn't learned, piece types are a best guess, especially under heavy overexposure. Tap squares to fix them; fixes and later tracked moves teach the scanner. Calibrating once on a start position avoids this.

## Licenses

- Stockfish.js: GPLv3 (loaded at runtime from the CDN).
- chess.js: BSD-2-Clause (loaded at runtime from the CDN).
