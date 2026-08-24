# Scoresheet

An over-the-board chess advisor. Play your game on a real board, move the pieces
here as each move happens, and the engine tells you what to play next.

**Live: https://tateo12.github.io/chess-advisor/**

- Move pieces by tapping a piece then its destination, or by dragging.
- The game is saved in your browser and survives a reload. It stays until you press **New game**.
- The brass arrow is the engine's recommendation. Hover an alternative to see its arrow instead.
- Paste a PGN or a bare move list into **Replay a game** to watch it play out, then get the best move in the final position.
- **Depth** is the strength dial: ~12 is International Master, ~18 is Grandmaster. Higher takes longer.

## How it works

Fully static — no backend, no API key, no build step, and after the page loads, no
network calls at all.

- Rules, legal moves, and notation: [chess.js](https://github.com/jhlywa/chess.js), vendored in `vendor/`.
- Analysis: Stockfish 18 NNUE compiled to WebAssembly ([stockfish.js](https://github.com/nmrugg/stockfish.js)),
  vendored in `vendor/` and driven over UCI in a Web Worker.

Five things worth knowing if you fork this:

1. It uses the **single-threaded** `lite-single` build deliberately. The multi-threaded
   build needs `SharedArrayBuffer`, which needs `Cross-Origin-Opener-Policy` and
   `Cross-Origin-Embedder-Policy` headers — and GitHub Pages cannot send custom headers.
2. The `lite` net is also the only one that fits. The full-strength `.wasm` is 113 MB,
   past GitHub's 100 MB per-file limit; `lite-single` is 7.3 MB.
3. UCI scores are from the **mover's** perspective, not White's. `parseInfo` flips them
   once on the way in, so the rest of the file keeps speaking White-relative.
4. MultiPV rank 1 is already the mover's best line, so candidates are ordered by rank
   and never re-sorted by eval — which removes the perspective bug entirely.
5. The worker is booted once and reused. A 7 MB wasm load per move would be brutal, and
   reuse keeps Stockfish's transposition table warm between moves in the same game.

### Previously

This called [chess-api.com](https://chess-api.com/) over a WebSocket. Its free tier
queues public requests and drops the socket at ~60s without ever answering — at the time
of the swap the queue sat ~45 deep, so every move failed with "couldn't reach the engine."
The engine now runs in the tab: depth 12 with three lines returns in well under a second.

## Read a screenshot

Drop a chess.com screenshot on the import card (or click it, or press Ctrl/Cmd+V) and
the position is read off the image, shown on the board for you to check, and only then
loaded. Everything runs in the tab -- no model, no API key, no upload.

It handles **18 of chess.com's piece themes**. Each set's six silhouettes are baked in
as 32x32 bitmasks, 128 bytes each -- about 24 KB in total, matched against the real
artwork rather than guessed from Unicode glyphs. Which theme a screenshot uses is worked
out from the screenshot: every theme is scored against every piece found, and the set
that explains them best wins.

How it works, in order:

1. **Find the grid.** Column and row edge energy propose candidate `(start, step)` pairs;
   each is then rescored on how *checkered* it actually is, which is what rejects the
   surrounding app chrome. Steps are searched in half-pixels, because a board that fills
   a 390px-wide screen has a cell size of 48.75.
2. **Read twice.** The image is analysed at native size and again enlarged, and whichever
   grid scores higher on checkeredness wins. A small screenshot leaves too few pixels per
   square for the silhouettes to survive antialiasing.
3. **Per cell:** ink is anything lighter than the light squares or darker than the dark
   ones -- comparing against only *this* square loses a white piece on a light one. The
   ink is flood-filled into a solid silhouette, which is both what the templates look like
   and where the piece's real colour lives, clear of its own outlines.
4. **Discard coordinate labels.** chess.com paints file and rank digits inside the edge
   squares; keeping only the largest blob of ink stops them deforming the piece they
   share a square with.
5. **Decide colour relatively.** A set's "white" pieces can be *darker* than the board
   they stand on -- chess.com's shaded pieces sit near luma 181 on a pale blue board
   whose midpoint is 223 -- so no fixed threshold works. All the pieces in one
   screenshot come from one set, so their core brightness falls into two clusters and
   two-means reads the boundary off the image itself.
6. **Name it** by silhouette overlap, with height/width/fill as tie-breakers.

Three things an image cannot tell you, so the UI asks instead of guessing: whose **turn**
it is, which **side the board is drawn from** (chess.com puts your own colour at the
bottom), and **castling rights** -- placement is geometry, castling is history. Anything
the classifier was unsure about is listed so you can check it before confirming.

Known limits: an unlisted piece theme falls back to the closest-looking one, which is
usually still right but is worth checking on the preview; and a photo of a physical
wooden board is a different and much harder problem that this does not attempt.

## Credits and licensing

Stockfish is **GPLv3**. `vendor/stockfish-18-lite-single.{js,wasm}` are unmodified build
artifacts from [stockfish.js](https://github.com/nmrugg/stockfish.js). Stockfish is by
T. Romstad, M. Costalba, J. Kiiski, G. Linscott and contributors, with nets by Linmiao Xu.
chess.js is BSD-2-Clause.

This repo has no `LICENSE` file yet. Worth choosing one deliberately, since distributing a
GPLv3 engine alongside this page carries obligations — I'd check that rather than take my
word for it.

## Running locally

```sh
python3 -m http.server 8000   # then open http://localhost:8000
```

Opening `index.html` via `file://` won't work — the ES module import needs a real origin.
