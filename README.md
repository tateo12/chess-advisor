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
