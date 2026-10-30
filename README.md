<div align="center">
  <img src="docs/icon.png" alt="Pool" width="96" height="96" />
  <h1>Pool</h1>
  <p><em>3D WebGL pool with 194 flag balls, a look-ahead AI opponent, and a replay highlight reel</em></p>
  <p><a href="https://pool-bheng.vercel.app">Live</a> &middot; <a href="https://github.com/bunlongheng/pool">Repo</a> &middot; <a href="https://bunlongheng.com/projects?name=pool">Portfolio</a></p>
  <img src="docs/social-preview.png" alt="Pool - preview" width="820" />
</div>

---

# Pool

A landscape-first 3D WebGL pool game that also teaches categories: the balls can be country flags, colors, fruits, vegetables, or US state flags, and each one is named aloud when it drops.

![Pool table with glossy 3D flag balls racked on emerald felt](docs/screenshots/rack.png)

[![CI](https://github.com/bunlongheng/pool/actions/workflows/ci.yml/badge.svg)](https://github.com/bunlongheng/pool/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
![Next.js](https://img.shields.io/badge/Next.js-16-black?logo=next.js)
![React](https://img.shields.io/badge/React-19-149eca?logo=react)
![TypeScript](https://img.shields.io/badge/TypeScript-strict-3178c6?logo=typescript)
![three.js](https://img.shields.io/badge/three.js-0.185-000000?logo=three.js)
![Tests](https://img.shields.io/badge/tests-19%20passing-3fb950)

## Contents

- [Features](#features)
- [Architecture](#architecture)
- [How it works](#how-it-works)
- [Design decisions and trade-offs](#design-decisions-and-trade-offs)
- [Tech stack](#tech-stack)
- [Quick start](#quick-start)
- [Configuration](#configuration)
- [Project layout](#project-layout)
- [License](#license)

- **Ball Type categories** (a learning game): Countries (194 flags), Colors, Fruits, Vegetables, US States (50 flags), or Random - each ball is named on screen and spoken aloud when it drops.
- **AI mode**: a look-ahead computer player that clones the whole table, simulates every candidate shot to a full stop, refuses to scratch, and narrates each shot (target flag, pocket, straightness, reasoning) while it runs the rack in the fewest turns.
- **Replay**: a World-Cup-style highlight reel of the rack - per-move chapters, a scrubber, slow-mo playback, and a score synced to every frame.
- 15 object balls, each a real national flag on a glossy 3D WebGL sphere, plus a pearl-white cue ball.
- Drag-to-aim slingshot control: pull back from the cue ball, a colour-coded guide line (green -> amber -> red by power) shows the shot line and its first cushion bounce, release to break. Above 90% power, fire wraps the cue ball and the meter blinks red.
- Full 2D pool physics: elastic ball-on-ball collisions, cushion bounces, six pockets (recessed, realistic side pockets that reject lazy rail rolls), sub-stepped integration so nothing tunnels at speed and no ball is ever shoved over the frame.
- Pot an object flag and the pocket blinks green; scratch the cue and it blinks red, then the cue respots on the head string. Clear the rack to win with a fanfare.
- The HUD tracks **Score**, **Shots** (strokes taken), **Died** (scratches), and rack **Time**.
- **Kids mode**: a ghost-ball aim assist that shows the contact point and the struck ball's direction.
- **Settings** (persisted to localStorage): **Ball Size** (Normal / Big / Huge, resizes the real physics), **Cloth Color** (10 felt shades), **Table Frame** (Wood / Walnut / Leather / Black / Metal / Aluminum, with matching pocket rims and gloss), and **Cue Stick** (5 finishes).
- Synthesised sound effects (cue crack scaled by power + a fire boom, ball clicks by impact, rail thud, pocket drop, scratch, win) plus a looping background theme; one mute toggle silences both.
- Landscape-first table that fills the screen, with a rotate hint on portrait and iOS safe-area insets.

![AI mode narrating its shot - target flag, pocket, straightness and reasoning - as it runs the rack](docs/screenshots/ai.png)

## Architecture

A pure, deterministic physics core drives everything; rendering is split cleanly from simulation. The physics runs in abstract table units and writes a per-frame render buffer. A transparent react-three-fiber canvas reads that buffer to place glossy flag spheres, while a 2D canvas underneath paints the themed felt, framed rails, pockets, ball shadows and the aim guide. Table theming (surfaces, cloth colours, rail materials) lives in a pure data module. React state is touched only when a turn settles, never per frame.

```mermaid
flowchart LR
    P[pool.ts<br/>physics core] -->|render buffer| B[PoolBalls<br/>r3f WebGL balls]
    P -->|balls state| F[PoolTable<br/>2D felt + aim + loop]
    D[surfaces.ts<br/>themes + cloths + rails] -->|paint data| F
    I[pointer drag] -->|shoot| P
    S[sound.ts<br/>Web Audio + theme.mp3] -.->|impact events| F
    F --> C[canvas felt]
    B --> G[WebGL overlay]
```

| Layer | File | Role |
|-------|------|------|
| Physics + AI | `lib/pool.ts` | Pure, DOM-free: balls, collisions, cushions, pockets, aim geometry, rack, ball sizing, look-ahead AI shot planner |
| Themes | `app/data/surfaces.ts` | Pure paint data: 10 cloth colours, 6 rail/frame materials, 5 cue sticks + gloss |
| Sound | `lib/sound.ts` | Runtime-synthesised SFX + a looping theme (`public/theme.mp3`), graceful no-op fallback |
| Game loop | `app/components/PoolTable.tsx` | Single rAF loop, felt/aim canvas, drag-to-aim input, HUD, settings, AI driver, replay reel |
| 3D balls | `app/components/PoolBalls.tsx` | react-three-fiber overlay, glossy spheres from the render buffer |
| Lighting | `app/components/Studio.tsx` | r3f lights + environment for the clearcoat sheen |

## How it works

```mermaid
sequenceDiagram
    participant U as Player
    participant T as PoolTable
    participant Po as pool.ts
    participant B as PoolBalls (WebGL)
    U->>T: pointer down + drag from cue
    T->>Po: aimPath(dir, power) for the guide line
    U->>T: release
    T->>Po: shoot(cue, dir, power)
    loop every animation frame until all balls stop
        T->>Po: stepWorld(balls, dt)
        Po-->>T: collision + pocket events (sound)
        T->>B: write render buffer (positions, roll)
    end
    T->>U: update potted / shots / deaths
```

![A rack breaking open mid-play - flags scattered across the felt, a flag dropping into the top pocket, HUD tracking score, shots and time](docs/screenshots/play.png)

## Design decisions and trade-offs

| Decision | Chosen | Alternative | Why this trade-off | Cost we accept |
|----------|--------|-------------|--------------------|----------------|
| Physics | Pure, DOM-free core in table units | Physics baked into the render loop | Fully unit-testable and deterministic; render and sim stay decoupled | A render buffer has to be marshalled each frame |
| AI | Full look-ahead simulation of each shot | Cheap geometry heuristic | It knows exactly where every ball lands, never scratches, and plans run-outs | More CPU per turn than a heuristic |
| Side pockets | Recessed, reject lazy rail rolls | Generous circular capture | Rail-parallel skims roll past like real pool | Needs a directional mouth check, not just distance |
| Audio | Synthesised at runtime (Web Audio) | Bundled audio files | No files, no licensing, CSP stays `'self'` | SFX are tuned in code, not authored in a DAW |

## Tech stack

- Next.js 16 (App Router, static prerender) + React 19, TypeScript strict.
- three.js 0.185 with react-three-fiber and drei for the glossy MeshPhysicalMaterial balls.
- HTML5 Canvas 2D for the themed felt, framed rails, pockets and aim guide.
- Web Audio API for synthesised sound effects, plus a native `<audio>` element for the looping theme (`public/theme.mp3`, served same-origin).
- Tailwind CSS 4 for the HUD, settings tabs and overlays.
- node:test for the physics unit suite; ESLint 9. Deployed on Vercel.

## Quick start

```bash
git clone https://github.com/bunlongheng/pool.git
cd pool
npm install
npm run dev
```

Open http://localhost:3031 and turn the device to landscape. Pull back from the white cue ball, aim the line, and release to break. Tap the gear to open Settings. `npm test` runs the physics suite.

## Configuration

No environment variables required.

## Project layout

```
pool/
├── app/
│   ├── components/
│   │   ├── PoolTable.tsx    # game loop, felt canvas, drag-to-aim, HUD, settings, AI, replay
│   │   ├── PoolBalls.tsx    # r3f overlay: flag/state images, emoji + colour balls
│   │   └── Studio.tsx       # r3f lighting + environment
│   ├── data/
│   │   ├── themes.ts        # ball categories: countries, colors, fruits, veggies, US states
│   │   ├── countries.ts     # 194 countries (code, name, hue)
│   │   └── surfaces.ts      # 10 cloth colours, 6 rail materials, 5 cue sticks
│   ├── layout.tsx           # self-hosted fonts, metadata
│   ├── page.tsx
│   └── globals.css
├── lib/
│   ├── pool.ts              # pure physics core + look-ahead AI planner (unit-tested)
│   └── sound.ts             # Web Audio synth + theme playback
├── tests/pool.test.ts       # 19 node:test unit tests
├── public/
│   ├── flags/               # 194 country flag PNGs
│   ├── state-flags/         # 50 US state flag PNGs
│   └── theme.mp3            # looping background theme
└── next.config.ts           # CSP + security headers
```

## License

[MIT](LICENSE) (c) Bunlong Heng - covers the code. `public/theme.mp3` is not covered by the MIT license.

---

<p align="center">
  <sub>Built by <a href="https://bunlongheng.com">Bunlong Heng</a> &middot; <a href="https://bunlongheng.com/projects/pool">See it in my portfolio &rarr;</a></sub>
</p>

---

<div align="center">

<a href="https://bunlongheng.com"><img src="https://img.shields.io/badge/-bunlongheng.com-3A3A3C?style=for-the-badge&amp;labelColor=2A2A2C&amp;logo=data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABwAAAAcCAMAAABF0y%2BmAAADAFBMVEXx8vLq6v%2F19fX09PT8%2Ff309PT5%2BfnAwMBMaXHx8vL19fX5%2Bfny8vL09PT39%2Ff6%2Bfn6%2Bvr09PTx8fH4%2BPn%2F8vLy8%2FP09PTz8%2FPy8vL%2F%2F%2F%2Fz9PTz9PP19fby8vP19PXz8%2FT19%2Fb08%2FTx8vLy8vL09vby8%2FL29vf19fTv8PDz9vb7%2Bvn%2F%2Ff%2F9%2Ff3w8vH29fb%2F%2FP339%2Ff5%2Bfn59%2Ff19PT09PR7rao3VF3%2F%2Fv8AKDR6sqsBV1vv8fL4%2BPiux8b8%2Bvq90NDy8%2FR%2BlpppnZyZqrACLTxclZJSkI9en5ssZ2luqaUAeWgqbW4BX1gCfG0%2Fa3QaVG0MN0UKNUKUpKsGpYMAKjoRQ1UDf3AQRloJqIgEm32d0cUBg3ElbnQIln8eVWgrmIdP0KzF5d5s2acdfHwppJCP5MMXl4Tz8%2FLv8%2FP39%2FiewL5%2BqahYf4IrTFZkk5KlxsREiIYAHiqDnKFmnJowc3IdVllvpKK7zc3D2tnk5%2BmuycdwpKKuub2yv8Ly%2B%2FkAMT7z%2Bvjq7e09gH93q6kGdWpGgoJZmJSlw8M7c3WwyceTw76w1M8mYmpFfX8lW2AFOEpGiYhxqqRrqaRb2rdIsZ8AVE91jpU7d3t3saqBoKhako4AWVEcUlg3aWwDupAHb2YwmIkPP1ERSVA7WmZ8tq8HT14oZW%2BYuLsMl3wQT10ROksENUc8Z3MJSGVLh4dqpJ%2BXr7U51awUgHM4v6ad1MkPuJZAiowVd3IEhnZH0aoYrI05p5sDtYwkzZ8EPViYxsRel5USbHAIgmwIM0gQamzU4uF2malQi4oWQFQ1zaYrp4UXlYwfiHoyxqFE1q4qeX4yn5JIxKkdrX1b1rAbWGvd5Oh91sRijpomsJEVc3Ukc34dqIcwuJVKwZIXkoQZcXlNv5g%2BqZUZvpNl2rPG9N0WkIoNqouW4sVPwp%2BM0b6h0ssfq4d%2F1r8hpo%2Fh9e4AamUjnI9avaBixqgklYWSwL4xtItUxplYl5qZ58mq78wMgnYXfnlOmJaR1L%2Bj78ny9fS%2FrXQPAAAAFXRSTlP7Brvx%2FsJhAgD87r4U72C4uGH8vhRDodYHAAAACXBIWXMAAAsTAAALEwEAmpwYAAAC6UlEQVQokS2Sd3BUVRSHb0KS3QRCiZw5t%2By9j33zXjZkyUs2uekhEAi9dwRCL4JUEQRFAQUs9A4WOopKU0GpFoqCBZUivaqoFKkWLIS5GX7%2FfnPO78yZj%2Fj8UTViK6HggvNgSGuluEZRKbZqlN9H%2FFWiWUzIFQK5fhjOIRTDoqv4ia86S4q3KSByHdSqIhxofBKr5iNxLMm2bQqWVkorJ%2BAkO46nkQJhceSRmCTDvKJIJBLJzMxMLU6POJ4eBPF1KpNEbdt2Cms4LCcnZ%2FX6D3d%2BsKJJ7VSlNLoqgbjUtmkKKxlVmpf3bsFXW3d1b77gueJAQIPQBCilkMKemNi23Ts9T35%2FaMe2rsvmTypylGUFCSIYWJLXrm2nUwOPfNO5w%2B73Okxnjqe0JigQEL3JizoeONr%2F2u99vju4fd1rdZnyspQmnGvAUBprWNqz348%2FhXN%2F6P1%2By4Uvs%2FJkpRQRwpJBRVn7vf16Dfitz5XevQqWt3yalXtKaRIUQmglWfsu5y7%2BMrDs0TMftWo1ZVy64xioOVpcIdt8%2Fvqvf9%2F859KJsrLmrzA%2BOKC0IkFEtJza7JOfr%2FYd0P9G98tdOxe0Hp9qJrOIBgDuOeEtp%2F%2B4d%2FvWX%2F%2F9u%2B%2FtlaUTmOWZg7SQEEyuFe52%2BML9O3cZO%2F5xl06zn2VcadOJUqJTzt7a82XfP7sx9vmGNm1m1MstV4IHCVAAoTLqL1372ddn%2Fz%2B2v%2FWq12c1fSpsaeQWAZAANntxzuI3e3zbo8XGpm%2FMnNqiXrh%2BljaPd6UUabkjps19tdGnX2zKb9b4pTFDCzOMLmYtpW5h%2BpCRTeYtyV%2BT36zx808OL4xYGoFrkqDTbJpSXLdRgwbPZL%2BQnT129OOPFVEEsAMJpHKdeKMJgJSAiALdjJANQKnRJI4R6boAKEwLCiGoBBegQjBftQo1KQWOlplFKSV9qGaF1NpAE7QQESTlMSy6pp%2F4%2FFFVYxMNRBTaiM0tkImxNaL8vgfDR8gvoYRaxgAAAABJRU5ErkJggg%3D%3D" alt="bunlongheng.com"></a>
<a href="https://www.linkedin.com/in/bunlongheng/"><img src="https://img.shields.io/badge/LinkedIn-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white" alt="LinkedIn"></a>
<a href="https://www.instagram.com/ibunlong/"><img src="https://img.shields.io/badge/Instagram-C13584?style=for-the-badge&logo=instagram&logoColor=white" alt="Instagram"></a>
<a href="mailto:bheng.code@gmail.com"><img src="https://img.shields.io/badge/Email-2E7D32?style=for-the-badge&logo=gmail&logoColor=white" alt="Email"></a>

<br>

Built by **[Bunlong](https://bunlongheng.com)** &nbsp;&middot;&nbsp; [more apps](https://bunlongheng.com/projects)

</div>
