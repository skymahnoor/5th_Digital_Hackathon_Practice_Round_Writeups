# Twin Vaults — Writeup (50 pts)

## Challenge Info
- **Category:** Game Theory / Scripting
- **Goal:** Beat "the Warden" in a coin game, 100 rounds in a row, to get the flag

## The Game
Two vaults hold coins. Players take turns. On your turn you can either:
- **Take:** remove `k` coins from one vault, or
- **Tunnel:** remove `i` coins from vault 1 and `j` coins from vault 2, where the difference `|i - j|` is at most 1.

Whoever removes the last coin wins the round. The Warden "never loses" — meaning he always plays perfectly.

## Step 1 — Recognize the Game
This is a version of a classic math puzzle called **Wythoff's Game**. In this type of game, every position is either a "winning" position or a "losing" position for whoever has to move next. If you always move your opponent into a losing position, they can never win. The Warden wins by always doing exactly this — so to beat him, we need to find those losing positions ourselves.

## Step 2 — Find the Losing Positions
A brute-force program on small numbers revealed losing pairs like:
```
(0,0), (1,3), (2,6), (4,10), (5,13), (7,17), (8,20)...
```
A pattern emerged: for a losing pair `(x, y)` where `x <= y`:
```
y - x = 2n
x = floor(n * sqrt(2))
```
This formula matched the brute-force results.

## Step 3 — Strategy
On every turn, find a losing pair reachable with one "take" or one "tunnel" move, and move there. This keeps the Warden stuck on losing positions every single turn.

## Step 4 — Problems Faced
- The numbers involved are huge (around 4 quadrillion), so normal JavaScript numbers lose precision. Used **BigInt** instead.
- The game has an idle timer — playing by hand is too slow and causes a timeout. The solution needed to be fully automatic.

## Step 5 — Solver Script
Ran in the browser console on the challenge page. It connects over WebSocket, reads the vault sizes on every round, calculates the nearest losing pair, and sends the correct move automatically.

```js
(() => {
  const isq = (n) => { if (n < 2n) return n; let x = n, y = (x + 1n) / 2n; while (y < x) { x = y; y = (x + n / x) / 2n; } return x; };
  const A = (n) => isq(2n * n * n);
  const B = (n) => A(n) + 2n * n;
  const cands = (v) => { const s = isq(v * v / 2n), t = v - s, r = new Set(); for (let k = -3n; k <= 3n; k++) { r.add(s + k); r.add(t + k); } return r; };
  const move = (a, b) => {
    const ns = new Set([...cands(a), ...cands(b)]);
    const d = (a > b ? a - b : b - a) / 2n;
    for (let k = -3n; k <= 3n; k++) ns.add(d + k);
    for (const n of ns) {
      if (n < 0n) continue;
      for (const [x, y] of [[A(n), B(n)], [B(n), A(n)]]) {
        const i = a - x, j = b - y;
        if (i < 0n || j < 0n || (i === 0n && j === 0n)) continue;
        if (i === 0n) return `{"kind":"take","vault":2,"count":${j}}`;
        if (j === 0n) return `{"kind":"take","vault":1,"count":${i}}`;
        const df = i > j ? i - j : j - i;
        if (df <= 1n) return `{"kind":"tunnel","i":${i},"j":${j}}`;
      }
    }
    return `{"kind":"take","vault":${a >= b ? 1 : 2},"count":1}`;
  };
  const w = new WebSocket(`ws://${location.host}/ws`);
  w.onopen = () => w.send('{"type":"start","mode":"gauntlet"}');
  w.onmessage = (ev) => {
    const t = ev.data, m = JSON.parse(t);
    if (m.type === "round" || m.type === "state") {
      const a = BigInt(t.match(/"a":\s*(\d+)/)[1]), b = BigInt(t.match(/"b":\s*(\d+)/)[1]);
      w.send(`{"type":"move","move":${move(a, b)}}`);
    } else if (m.type === "end") { console.log("END:", m); if (m.flag) console.log("FLAG:", m.flag); }
    else if (m.type === "error") console.log("ERR:", m);
  };
})();
```

## Flag
```
flag{beat_the_warden_along_the_root_two_line17941492399PGNl4cVb3e9903b29d907d1ec7019a1e83991a8198e3be7_133}
```

## Lesson Learned
When an opponent "never loses" in a game, it's usually because they're following a perfect mathematical strategy. Find the math behind it, then automate the same strategy with a script.
