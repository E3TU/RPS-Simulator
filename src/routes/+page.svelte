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
	let frameIndex = 0;

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
		entities = Array.from({ length: total }, (_, i) => randomEntity(types[i % types.length]));

		counts = getCounts();
		accumulator = 0;
		lastTime = 0;
		draw();
	}

	function winner(a: EntityType, b: EntityType): EntityType | null {
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

		ctx.save();
		ctx.fillStyle = 'rgba(255, 255, 255, 0.045)';
		const gridStep = 28;
		for (let x = gridStep; x < width; x += gridStep) {
			for (let y = gridStep; y < height; y += gridStep) {
				ctx.fillRect(x, y, 1, 1);
			}
		}
		ctx.restore();

		ctx.fillStyle = '#ffffff';
		ctx.font = '24px "Apple Color Emoji", "Segoe UI Emoji", "Noto Color Emoji", sans-serif';
		ctx.textAlign = 'center';
		ctx.textBaseline = 'middle';

		for (const e of entities) {
			ctx.fillText(icons[e.type], e.x, e.y);
		}
	}

	function loop(time: number): void {
		if (!running) return;

		const dt = Math.min(lastTime === 0 ? 0 : (time - lastTime) / 1000, 0.05);

		lastTime = time;
		accumulator += dt;

		while (accumulator >= step) {
			update(step);
			accumulator -= step;
		}

		if (++frameIndex % 6 === 0) counts = getCounts();
		draw();

		frame = requestAnimationFrame(loop);
	}

	function start(): void {
		if (running) return;

		running = true;
		lastTime = 0;
		accumulator = 0;
		counts = getCounts();
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
	<meta property="og:type" content="website" />
	<meta property="og:title" content="RPS Ecosystem — field simulator" />
	<meta
		property="og:description"
		content="150 entities, one closed arena, three species that hunt each other in a loop."
	/>
	<meta name="twitter:card" content="summary" />
	<meta name="theme-color" content="#0d1117" />
</svelte:head>

<a class="skip-link" href="#arena">Skip to simulation</a>

<main>
	<header class="masthead">
		<div class="identity">
			<h1>RPS Ecosystem</h1>
		</div>
	</header>

	<p class="subtitle">
		Three species, one arena. Rock crushes scissors, scissors cuts paper, paper wraps rock — and
		every collision rewrites one of them. Every contact converts the loser. Populations drift toward whoever lands the last hit.
	</p>

	<section class="arena" id="arena" aria-label="Simulation arena">
		<canvas bind:this={canvas} aria-label="Animated rock-paper-scissors simulation"></canvas>
	</section>

	<section class="readout" aria-label="Population readout">
		<article class="species rock">
			<p class="label"><span aria-hidden="true">🪨</span> Rock</p>
			<p class="value">{counts.rock}</p>
			<div class="meter" aria-hidden="true">
				<span style:transform="scaleX({counts.rock / total})"></span>
			</div>
		</article>

		<article class="species paper">
			<p class="label"><span aria-hidden="true">📄</span> Paper</p>
			<p class="value">{counts.paper}</p>
			<div class="meter" aria-hidden="true">
				<span style:transform="scaleX({counts.paper / total})"></span>
			</div>
		</article>

		<article class="species scissors">
			<p class="label"><span aria-hidden="true">✂️</span> Scissors</p>
			<p class="value">{counts.scissors}</p>
			<div class="meter" aria-hidden="true">
				<span style:transform="scaleX({counts.scissors / total})"></span>
			</div>
		</article>
	</section>

	<section class="controls" aria-label="Simulation controls">
		<div class="buttons">
			<button class="primary" onclick={start} disabled={running}>
				<span aria-hidden="true">▶</span> Start
			</button>

			<button onclick={pause} disabled={!running}>
				<span aria-hidden="true">❚❚</span> Pause
			</button>

			<button class="ghost" onclick={reset}>
				<span aria-hidden="true">↺</span> Reset
			</button>
		</div>

		<label class="speed">
			<span class="speed-label">Speed</span>
			<input type="range" min="0.5" max="4" step="0.5" bind:value={speed} style:--v={speed} />
			<span class="speed-value">{speed.toFixed(1)}×</span>
		</label>
	</section>
</main>

<style>
	main {
		width: min(1240px, 100% - 2.5rem);
		margin: 0 auto;
		padding: 3.5rem 0 4.5rem;
	}

	.masthead {
		display: grid;
		grid-template-columns: minmax(0, 1fr) auto;
		align-items: end;
		gap: 1.5rem;
		padding-bottom: 1.5rem;
	}

	h1 {
		margin: 0;
		font-size: clamp(2.1rem, 5.2vw, 3.9rem);
		font-weight: 600;
		line-height: 0.98;
		letter-spacing: -0.035em;
		text-wrap: balance;
	}

	.subtitle {
		max-width: 58ch;
		margin: 1.1rem 0 2rem;
		color: var(--text-dim);
		font-size: 1rem;
		line-height: 1.6;
		text-wrap: pretty;
	}

	.arena {
		position: relative;
		border-radius: 20px;
		background: var(--bg-deep);
		box-shadow:
			0 0 0 1px rgba(255, 255, 255, 0.1),
			0 30px 60px -30px rgba(0, 0, 0, 0.85);
		overflow: hidden;
		isolation: isolate;
		animation: rise 520ms cubic-bezier(0.22, 1, 0.36, 1) both;
	}

	.arena::after {
		content: '';
		position: absolute;
		inset: 0;
		z-index: 2;
		border-radius: inherit;
		box-shadow: inset 0 0 90px rgba(4, 9, 12, 0.65);
		pointer-events: none;
	}

	canvas {
		display: block;
		width: 100%;
		height: clamp(320px, 52vh, 560px);
	}

	.readout {
		display: grid;
		grid-template-columns: repeat(3, minmax(0, 1fr));
		gap: 0.5rem;
		margin-top: 1.5rem;
	}

	.species {
		display: grid;
		grid-template-areas:
			'label'
			'value'
			'meter';
		grid-template-columns: 1fr;
		align-items: baseline;
		gap: 0.35rem 0.75rem;
		padding: 1.25rem 1.4rem 1.35rem;
		border-radius: 14px;
		background: var(--surface);
		animation: rise 520ms cubic-bezier(0.22, 1, 0.36, 1) both;
	}

	.species:nth-child(2) {
		animation-delay: 70ms;
	}

	.species:nth-child(3) {
		animation-delay: 140ms;
	}

	.label {
		grid-area: label;
		display: flex;
		align-items: center;
		gap: 0.45rem;
		margin: 0;
		font-size: 0.86rem;
		font-weight: 500;
		color: var(--text-dim);
	}

	.label span {
		font-size: 1.05rem;
		line-height: 1;
	}

	.value {
		grid-area: value;
		margin: 0.2rem 0 0;
		font-family: var(--font-mono);
		font-size: clamp(2rem, 3.4vw, 2.75rem);
		font-weight: 500;
		line-height: 1;
		letter-spacing: -0.02em;
		font-variant-numeric: tabular-nums;
	}

	.meter {
		grid-area: meter;
		height: 3px;
		margin-top: 0.9rem;
		border-radius: 3px;
		background: rgba(255, 255, 255, 0.06);
		overflow: hidden;
	}

	.meter span {
		display: block;
		height: 100%;
		border-radius: inherit;
		transform-origin: left center;
		transition: transform 220ms linear;
	}

	.rock .value {
		color: var(--rock);
	}

	.paper .value {
		color: var(--paper);
	}

	.scissors .value {
		color: var(--scissors);
	}

	.rock .meter span {
		background: var(--rock);
	}

	.paper .meter span {
		background: var(--paper);
	}

	.scissors .meter span {
		background: var(--scissors);
	}

	.controls {
		display: flex;
		flex-wrap: wrap;
		align-items: center;
		justify-content: space-between;
		gap: 1.5rem;
		margin-top: 1.75rem;
	}

	.buttons {
		display: flex;
		gap: 0.5rem;
	}

	button {
		display: inline-flex;
		align-items: center;
		gap: 0.6rem;
		padding: 0.82rem 1.4rem;
		border: 0;
		border-radius: 10px;
		background: var(--surface);
		color: var(--text-dim);
		font-family: var(--font-mono);
		font-size: 0.8rem;
		letter-spacing: 0.04em;
		cursor: pointer;
		transition:
			background 200ms ease,
			color 200ms ease,
			transform 120ms ease;
	}

	button span {
		font-size: 0.72em;
		opacity: 0.85;
	}

	button:hover:not(:disabled) {
		background: var(--surface-hover);
		color: var(--text);
	}

	button:active:not(:disabled) {
		transform: scale(0.98) translateY(1px);
	}

	button.primary {
		background: var(--accent-fill);
		color: #04120c;
		font-weight: 500;
	}

	button.primary:hover:not(:disabled) {
		background: var(--accent-hover);
		color: #04120c;
	}

	button.primary span {
		opacity: 1;
	}

	button.ghost {
		background: transparent;
		color: var(--text-faint);
	}

	button.ghost:hover:not(:disabled) {
		background: var(--surface);
		color: var(--text);
	}

	button:disabled {
		opacity: 0.38;
		cursor: default;
	}

	.speed {
		display: flex;
		align-items: center;
		gap: 1rem;
		padding: 0.55rem 0.9rem 0.55rem 1.1rem;
		border-radius: 12px;
		background: var(--surface);
		font-family: var(--font-mono);
		font-size: 0.72rem;
		letter-spacing: 0.08em;
		text-transform: uppercase;
		color: var(--text-faint);
	}

	.speed-value {
		min-width: 3.6ch;
		padding-left: 0.9rem;
		border-left: 1px solid var(--line);
		color: var(--text);
		font-variant-numeric: tabular-nums;
		text-transform: none;
		text-align: right;
	}

	input[type='range'] {
		--fill: calc((var(--v, 1) - 0.5) / 3.5 * 100%);
		-webkit-appearance: none;
		appearance: none;
		width: clamp(140px, 20vw, 220px);
		height: 34px;
		margin: 0;
		background: transparent;
		cursor: pointer;
	}

	input[type='range']::-webkit-slider-runnable-track {
		height: 6px;
		border-radius: 6px;
		background: linear-gradient(
			to right,
			var(--accent) 0 var(--fill),
			rgba(255, 255, 255, 0.09) var(--fill) 100%
		);
	}

	input[type='range']::-webkit-slider-thumb {
		-webkit-appearance: none;
		appearance: none;
		width: 18px;
		height: 18px;
		margin-top: -6px;
		border: 3px solid var(--surface);
		border-radius: 50%;
		background: var(--accent);
		box-shadow: 0 2px 8px rgba(0, 0, 0, 0.55);
		transition:
			transform 160ms ease,
			box-shadow 160ms ease;
	}

	input[type='range']:hover::-webkit-slider-thumb {
		transform: scale(1.12);
		box-shadow: 0 0 0 6px rgba(111, 211, 160, 0.14);
	}

	input[type='range']:active::-webkit-slider-thumb {
		transform: scale(1.18);
	}

	input[type='range']::-moz-range-track {
		height: 6px;
		border-radius: 6px;
		background: rgba(255, 255, 255, 0.09);
	}

	input[type='range']::-moz-range-progress {
		height: 6px;
		border-radius: 6px;
		background: var(--accent);
	}

	input[type='range']::-moz-range-thumb {
		width: 12px;
		height: 12px;
		border: 3px solid var(--surface);
		border-radius: 50%;
		background: var(--accent);
	}

	@keyframes rise {
		from {
			opacity: 0;
			transform: translateY(10px);
		}
		to {
			opacity: 1;
			transform: translateY(0);
		}
	}

	@media (max-width: 720px) {
		main {
			width: calc(100% - 1.75rem);
			padding: 2rem 0 3rem;
		}

		.masthead {
			grid-template-columns: 1fr;
			align-items: start;
			gap: 1rem;
		}

		.readout {
			grid-template-columns: 1fr;
			gap: 0.4rem;
		}

		.species {
			grid-template-areas: 'label value' 'meter meter';
			grid-template-columns: 1fr auto;
			align-items: center;
			padding: 0.95rem 1rem;
		}

		.value {
			margin: 0;
			font-size: 1.5rem;
		}

		.meter {
			margin-top: 0.65rem;
		}

		.controls {
			flex-direction: column;
			align-items: stretch;
		}

		.buttons {
			flex-wrap: wrap;
		}

		button {
			flex: 1 1 auto;
			justify-content: center;
		}

		.speed {
			justify-content: space-between;
		}

		input[type='range'] {
			flex: 1;
			width: auto;
		}
	}
</style>
