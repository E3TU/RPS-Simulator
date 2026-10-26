
<script lang="ts">
  import { onMount } from 'svelte';

  type EntityType = 'rock' | 'paper' | 'scissors';

  type Entity = {
    x: number;
    y: number;
    vx: number;
    vy: number;
    type: EntityType;
  };

  type Counts = Record<EntityType, number>;

  let canvas: HTMLCanvasElement;
  let running = $state(false);
  let counts = $state<Counts>({
    rock: 0,
    paper: 0,
    scissors: 0
  });
  let speed = $state(1);

  const total = 150;
  const radius = 13;
  const step = 1 / 60;

  const types: EntityType[] = ['rock', 'paper', 'scissors'];

  const icons: Record<EntityType, string> = {
    rock: '🪨',
    paper: '📄',
    scissors: '✂️'
  };

  let entities: Entity[] = [];
  let ctx: CanvasRenderingContext2D | null = null;
  let width = 800;
  let height = 500;
  let frame: number | undefined;
  let lastTime = 0;
  let accumulator = 0;

  function randomEntity(type: EntityType): Entity {
    return {
      x: radius + Math.random() * Math.max(0, width - radius * 2),
      y: radius + Math.random() * Math.max(0, height - radius * 2),
      vx: (Math.random() - 0.5) * 100,
      vy: (Math.random() - 0.5) * 100,
      type
    };
  }

  function getCounts(): Counts {
    return {
      rock: entities.filter((e) => e.type === 'rock').length,
      paper: entities.filter((e) => e.type === 'paper').length,
      scissors: entities.filter((e) => e.type === 'scissors').length
    };
  }

  function reset(): void {
    entities = Array.from({ length: total }, (_, i) =>
      randomEntity(types[i % types.length])
    );

    counts = getCounts();
    accumulator = 0;
    lastTime = 0;
    draw();
  }

  // A beats B, B beats C, C beats A.
  function winner(
    a: EntityType,
    b: EntityType
  ): EntityType | null {
    if (a === b) return null;

    if (
      (a === 'rock' && b === 'scissors') ||
      (a === 'scissors' && b === 'paper') ||
      (a === 'paper' && b === 'rock')
    ) {
      return a;
    }

    return b;
  }

  function update(dt: number): void {
    for (const e of entities) {
      e.x += e.vx * dt * speed;
      e.y += e.vy * dt * speed;

      if (e.x < radius) {
        e.x = radius;
        e.vx = Math.abs(e.vx);
      } else if (e.x > width - radius) {
        e.x = width - radius;
        e.vx = -Math.abs(e.vx);
      }

      if (e.y < radius) {
        e.y = radius;
        e.vy = Math.abs(e.vy);
      } else if (e.y > height - radius) {
        e.y = height - radius;
        e.vy = -Math.abs(e.vy);
      }
    }

    // Resolve collisions.
    for (let i = 0; i < entities.length; i++) {
      for (let j = i + 1; j < entities.length; j++) {
        const a = entities[i];
        const b = entities[j];

        if (a.type === b.type) continue;

        const dx = a.x - b.x;
        const dy = a.y - b.y;
        const minDistance = radius * 2;

        if (dx * dx + dy * dy < minDistance * minDistance) {
          const winningType = winner(a.type, b.type);

          if (winningType) {
            a.type = winningType;
            b.type = winningType;
          }

          // Reverse movement to help separate colliding entities.
          a.vx *= -1;
          a.vy *= -1;
          b.vx *= -1;
          b.vy *= -1;
        }
      }
    }
  }

  function draw(): void {
    if (!ctx) return;

    ctx.clearRect(0, 0, width, height);
    ctx.font = '22px sans-serif';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';

    for (const e of entities) {
      ctx.fillText(icons[e.type], e.x, e.y);
    }
  }

  function loop(time: number): void {
    if (!running) return;

    const dt = Math.min(
      lastTime === 0 ? 0 : (time - lastTime) / 1000,
      0.05
    );

    lastTime = time;
    accumulator += dt;

    // Fixed-step simulation for consistent movement.
    while (accumulator >= step) {
      update(step);
      accumulator -= step;
    }

    counts = getCounts();
    draw();

    frame = requestAnimationFrame(loop);
  }

  function start(): void {
    if (running) return;

    running = true;
    lastTime = 0;
    frame = requestAnimationFrame(loop);
  }

  function pause(): void {
    running = false;

    if (frame !== undefined) {
      cancelAnimationFrame(frame);
      frame = undefined;
    }
  }

  onMount(() => {
    const context = canvas.getContext('2d');

    if (!context) return;

    ctx = context;

    function resize(): void {
      const rect = canvas.getBoundingClientRect();
      const dpr = window.devicePixelRatio || 1;

      width = rect.width;
      height = rect.height;

      canvas.width = Math.round(width * dpr);
      canvas.height = Math.round(height * dpr);

      ctx?.setTransform(dpr, 0, 0, dpr, 0, 0);

      // Keep entities inside the resized canvas.
      for (const e of entities) {
        e.x = Math.max(radius, Math.min(width - radius, e.x));
        e.y = Math.max(radius, Math.min(height - radius, e.y));
      }

      if (entities.length === 0) {
        reset();
      } else {
        draw();
      }
    }

    const observer = new ResizeObserver(resize);
    observer.observe(canvas);

    resize();

    return () => {
      pause();
      observer.disconnect();
      ctx = null;
    };
  });
</script>

<svelte:head>
  <title>RPS Ecosystem Simulator</title>
  <meta
    name="description"
    content="Watch rock, paper, and scissors compete in an animated ecosystem."
  />
</svelte:head>

<main>
  <h1>RPS Ecosystem</h1>
  <p class="subtitle">
    Watch rock, paper, and scissors battle for dominance.
  </p>

  <section class="stats" aria-label="Population counts">
    <div class="rock">
      <span>🪨 Rock</span>
      <strong>{counts.rock}</strong>
    </div>

    <div class="paper">
      <span>📄 Paper</span>
      <strong>{counts.paper}</strong>
    </div>

    <div class="scissors">
      <span>✂️ Scissors</span>
      <strong>{counts.scissors}</strong>
    </div>
  </section>

  <div class="arena">
    <canvas
      bind:this={canvas}
      aria-label="Animated rock-paper-scissors simulation"
    ></canvas>
  </div>

  <section class="controls" aria-label="Simulation controls">
    <button onclick={start} disabled={running}>
      ▶ Start
    </button>

    <button onclick={pause} disabled={!running}>
      ⏸ Pause
    </button>

    <button onclick={reset}>
      ↻ Reset
    </button>

    <label>
      <span>Speed: {speed.toFixed(1)}×</span>
      <input
        type="range"
        min="0.5"
        max="4"
        step="0.5"
        bind:value={speed}
      />
    </label>
  </section>
</main>

<style>
  :global(body) {
    margin: 0;
    background: #0b1020;
    color: #f1f5f9;
    font-family: system-ui, sans-serif;
  }

  main {
    max-width: 900px;
    margin: 40px auto;
    padding: 20px;
  }

  h1 {
    margin-bottom: 4px;
    font-size: 2.2rem;
  }

  .subtitle {
    color: #94a3b8;
    margin-bottom: 28px;
  }

  .stats {
    display: grid;
    grid-template-columns: repeat(3, minmax(0, 1fr));
    gap: 12px;
    margin-bottom: 18px;
  }

  .stats div {
    display: flex;
    flex-direction: column;
    gap: 12px;
    padding: 18px;
    border-radius: 14px;
    background: #151d30;
    border: 1px solid #293449;
  }

  .stats strong {
    font-size: 1.8rem;
    font-variant-numeric: tabular-nums;
  }

  .rock {
    color: #fbbf24;
  }

  .paper {
    color: #60a5fa;
  }

  .scissors {
    color: #fb7185;
  }

  .arena {
    overflow: hidden;
    border: 1px solid #334155;
    border-radius: 16px;
    background: #10182a;
  }

  canvas {
    display: block;
    width: 100%;
    height: 500px;
  }

  .controls {
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    gap: 12px;
    margin-top: 18px;
  }

  button {
    background: #293449;
    border: 1px solid #475569;
    color: white;
    padding: 11px 16px;
    border-radius: 9px;
    cursor: pointer;
  }

  button:hover:not(:disabled) {
    background: #37465f;
  }

  button:disabled {
    opacity: 0.45;
    cursor: default;
  }

  label {
    display: flex;
    align-items: center;
    gap: 10px;
    margin-left: auto;
  }

  input {
    width: 100px;
    accent-color: #60a5fa;
  }

  @media (max-width: 550px) {
    main {
      margin: 20px auto;
      padding: 14px;
    }

    h1 {
      font-size: 1.8rem;
    }

    .stats {
      gap: 8px;
    }

    .stats div {
      padding: 10px;
      gap: 8px;
      font-size: 0.9rem;
    }

    .stats strong {
      font-size: 1.4rem;
    }

    canvas {
      height: 400px;
    }

    label {
      margin-left: 0;
    }
  }
</style>
