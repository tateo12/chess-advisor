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

Fully static — no backend, no API key, no build step.

- Rules, legal moves, and notation: [chess.js](https://github.com/jhlywa/chess.js), vendored in `vendor/`.
- Analysis: [chess-api.com](https://chess-api.com/) (Stockfish 18 NNUE) over WebSocket.
  WebSocket rather than POST because POST collapses `variants` down to a single move.

Two things about that API worth knowing if you fork this:

1. `eval` is **always** from White's perspective. Black's best move is the *lowest* eval, not the highest.
2. It answers HTTP 200 even on failure — you have to branch on `data.type === 'error'`.

## Running locally

```sh
python3 -m http.server 8000   # then open http://localhost:8000
```

Opening `index.html` via `file://` won't work — the ES module import needs a real origin.
