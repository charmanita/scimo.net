<script lang="ts">
	import { onMount, onDestroy } from 'svelte';

	let canvas: HTMLCanvasElement;
	let ctx: CanvasRenderingContext2D;
	let rafId: number;
	let won = false;
	let deaths = 0;

	const CANVAS_W = 800;
	const CANVAS_H = 300;
	const GRAVITY = 0.22;
	const MOVE_SPEED = 1.8;
	const JUMP_VEL = -6;
	const GROUND_Y = 240;

	// --- Level layout -------------------------------------------------
	// Ground segments (with gaps to jump over) + floating platforms
	const START_X = 40;

	const platforms = [
		// ground segments
		{ x: 0, y: GROUND_Y, w: 300, h: 60 },
		{ x: 380, y: GROUND_Y, w: 220, h: 60 },
		{ x: 700, y: GROUND_Y, w: 260, h: 60 },
		{ x: 1060, y: GROUND_Y, w: 340, h: 60 },
		{ x: 1500, y: GROUND_Y, w: 500, h: 60 },
		// floating platforms
		{ x: 200, y: 170, w: 90, h: 16 },
		{ x: 460, y: 190, w: 80, h: 16 },
		{ x: 640, y: 150, w: 70, h: 16 },
		{ x: 900, y: 180, w: 90, h: 16 },
		{ x: 1120, y: 150, w: 80, h: 16 },
		{ x: 1300, y: 190, w: 100, h: 16 },
		{ x: 1620, y: 150, w: 120, h: 16 }
	];

	const LEVEL_WIDTH = 2000;

	type Enemy = {
		x: number;
		y: number;
		w: number;
		h: number;
		vx: number;
		minX: number;
		maxX: number;
		alive: boolean;
	};

	function makeEnemies(): Enemy[] {
		return [
			{ x: 120, y: GROUND_Y - 20, w: 20, h: 20, vx: 0.7, minX: 60, maxX: 260, alive: true },
			{ x: 420, y: GROUND_Y - 20, w: 20, h: 20, vx: 0.8, minX: 390, maxX: 570, alive: true },
			{ x: 760, y: GROUND_Y - 20, w: 20, h: 20, vx: 0.75, minX: 710, maxX: 930, alive: true },
			{ x: 1100, y: GROUND_Y - 20, w: 20, h: 20, vx: 0.9, minX: 1070, maxX: 1370, alive: true },
			{ x: 1560, y: GROUND_Y - 20, w: 20, h: 20, vx: 0.85, minX: 1510, maxX: 1950, alive: true }
		];
	}

	let enemies: Enemy[] = makeEnemies();

	const star = { x: 1900, y: 120, r: 18 };

	const player = {
		x: START_X,
		y: GROUND_Y - 24,
		w: 18,
		h: 24,
		vx: 0,
		vy: 0,
		onGround: false,
		facing: 1
	};

	const keys: Record<string, boolean> = {};

	function resetPlayer() {
		player.x = START_X;
		player.y = GROUND_Y - 24;
		player.vx = 0;
		player.vy = 0;
		enemies = makeEnemies();
	}

	function handleKeydown(e: KeyboardEvent) {
		keys[e.key] = true;
		if (['ArrowLeft', 'ArrowRight', 'ArrowUp', ' '].includes(e.key)) e.preventDefault();
	}
	function handleKeyup(e: KeyboardEvent) {
		keys[e.key] = false;
	}

	function aabb(
		ax: number,
		ay: number,
		aw: number,
		ah: number,
		bx: number,
		by: number,
		bw: number,
		bh: number
	) {
		return ax < bx + bw && ax + aw > bx && ay < by + bh && ay + ah > by;
	}

	function update() {
		if (won) return;

		if (keys['ArrowLeft'] || keys['a'] || keys['A']) {
			player.vx = -MOVE_SPEED;
			player.facing = -1;
		} else if (keys['ArrowRight'] || keys['d'] || keys['D']) {
			player.vx = MOVE_SPEED;
			player.facing = 1;
		} else {
			player.vx = 0;
		}

		if ((keys['ArrowUp'] || keys[' '] || keys['w'] || keys['W']) && player.onGround) {
			player.vy = JUMP_VEL;
			player.onGround = false;
		}

		player.vy += GRAVITY;
		player.x += player.vx;
		player.y += player.vy;

		if (player.x < 0) player.x = 0;
		if (player.x + player.w > LEVEL_WIDTH) player.x = LEVEL_WIDTH - player.w;

		player.onGround = false;
		for (const p of platforms) {
			const wasAbove = player.y + player.h - player.vy <= p.y + 1;
			if (
				wasAbove &&
				player.vy >= 0 &&
				player.x + player.w > p.x &&
				player.x < p.x + p.w &&
				player.y + player.h >= p.y &&
				player.y + player.h <= p.y + p.h + 12
			) {
				player.y = p.y - player.h;
				player.vy = 0;
				player.onGround = true;
			}
		}

		if (player.y > CANVAS_H + 100) {
			deaths += 1;
			resetPlayer();
		}

		for (const en of enemies) {
			if (!en.alive) continue;
			en.x += en.vx;
			if (en.x < en.minX || en.x + en.w > en.maxX) en.vx *= -1;

			if (aabb(player.x, player.y, player.w, player.h, en.x, en.y, en.w, en.h)) {
				const stomping = player.vy > 0 && player.y + player.h - player.vy <= en.y + 6;
				if (stomping) {
					en.alive = false;
					player.vy = JUMP_VEL * 0.6;
				} else {
					deaths += 1;
					resetPlayer();
				}
			}
		}

		const dx = player.x + player.w / 2 - star.x;
		const dy = player.y + player.h / 2 - star.y;
		if (Math.sqrt(dx * dx + dy * dy) < star.r + 14) {
			won = true;
		}
	}

	function drawStar(cx: number, cy: number, outerR: number, glow: number) {
		ctx.save();
		ctx.shadowColor = '#ffffff';
		ctx.shadowBlur = glow;
		ctx.fillStyle = '#ffffff';
		ctx.beginPath();
		const innerR = outerR * 0.28;
		for (let i = 0; i < 4; i++) {
			const angle = (Math.PI / 2) * i;
			const nextAngle = angle + Math.PI / 2;
			const midAngle = (angle + nextAngle) / 2;
			const ox = cx + Math.cos(angle) * outerR;
			const oy = cy + Math.sin(angle) * outerR;
			const ix = cx + Math.cos(midAngle) * innerR;
			const iy = cy + Math.sin(midAngle) * innerR;
			if (i === 0) ctx.moveTo(ox, oy);
			else ctx.lineTo(ox, oy);
			ctx.quadraticCurveTo(
				ix,
				iy,
				cx + Math.cos(nextAngle) * outerR,
				cy + Math.sin(nextAngle) * outerR
			);
		}
		ctx.closePath();
		ctx.fill();
		ctx.restore();
	}

	function draw(time: number) {
		ctx.fillStyle = '#000000';
		ctx.fillRect(0, 0, CANVAS_W, CANVAS_H);

		const camX = Math.max(0, Math.min(player.x - CANVAS_W / 2, LEVEL_WIDTH - CANVAS_W));

		ctx.save();
		ctx.translate(-camX, 0);

		ctx.strokeStyle = '#ffffff';
		ctx.fillStyle = '#000000';
		ctx.lineWidth = 2;
		for (const p of platforms) {
			ctx.fillRect(p.x, p.y, p.w, p.h);
			ctx.strokeRect(p.x, p.y, p.w, p.h);
		}

		for (const en of enemies) {
			if (!en.alive) continue;
			ctx.fillStyle = '#ffffff';
			ctx.fillRect(en.x, en.y, en.w, en.h);
			ctx.fillStyle = '#000000';
			ctx.fillRect(en.x + 4, en.y + 6, 3, 3);
			ctx.fillRect(en.x + en.w - 7, en.y + 6, 3, 3);
		}

		const glow = 16 + Math.sin(time * 0.004) * 8;
		drawStar(star.x, star.y, star.r, glow);

		drawPlayer();

		ctx.restore();
	}

	function drawPlayer() {
		const { x, y, w, h, facing } = player;

		// body (jacket torso + legs)
		ctx.fillStyle = '#ffffff';
		ctx.fillRect(x, y + 8, w, h - 8);

		// curly hair (bumpy silhouette on top of head)
		ctx.beginPath();
		ctx.arc(x + 3, y + 2, 3, 0, Math.PI * 2);
		ctx.arc(x + 7, y, 3.5, 0, Math.PI * 2);
		ctx.arc(x + 11, y + 1, 3, 0, Math.PI * 2);
		ctx.arc(x + 14, y + 3, 2.5, 0, Math.PI * 2);
		ctx.fill();

		// face block
		ctx.fillRect(x + 2, y + 3, w - 4, 7);

		// jacket collar flare at the neck
		ctx.fillRect(x - 1, y + 8, 3, 4);
		ctx.fillRect(x + w - 2, y + 8, 3, 4);

		// -- dark (outline-only) details --
		ctx.strokeStyle = '#000000';
		ctx.fillStyle = '#000000';
		ctx.lineWidth = 1;

		// eyes
		const eyeX = facing > 0 ? x + w - 6 : x + 3;
		ctx.fillRect(eyeX, y + 6, 2, 2);

		// mustache / goatee
		ctx.fillRect(x + w / 2 - 3, y + 9, 6, 1.5);
		ctx.fillRect(x + w / 2 - 1.5, y + 10.5, 3, 2);

		// jacket zipper (center line down the torso)
		ctx.fillRect(x + w / 2 - 0.5, y + 12, 1, h - 14);

		// racing stripe on the leading sleeve
		const sleeveX = facing > 0 ? x + w - 3 : x + 1;
		ctx.fillRect(sleeveX, y + 14, 2, 4);
	}

	function loop(time: number) {
		update();
		draw(time);
		rafId = requestAnimationFrame(loop);
	}

	function restart() {
		won = false;
		deaths = 0;
		resetPlayer();
	}

	onMount(() => {
		ctx = canvas.getContext('2d')!;
		ctx.imageSmoothingEnabled = false;
		window.addEventListener('keydown', handleKeydown);
		window.addEventListener('keyup', handleKeyup);
		rafId = requestAnimationFrame(loop);
	});

	onDestroy(() => {
		if (typeof window !== 'undefined') {
			window.removeEventListener('keydown', handleKeydown);
			window.removeEventListener('keyup', handleKeyup);
		}
		if (rafId) cancelAnimationFrame(rafId);
	});
</script>

<svelte:head>
	<title>MERCY - SCIMO</title>
	<meta property="og:title" content="scimo.net/mercy" />
	<meta property="og:description" content="MERCY - SCIMO" />
	<meta property="og:url" content="https://scimo.net/mercy" />
	<meta property="og:image" content="https://scimo.net/images/og-image.png" />
	<meta name="theme-color" content="#0a0a0a" />
	<meta property="og:site_name" content="MERCY - SCIMO" />
</svelte:head>

<div class="wrap">
	<div class="title">
		<h1>MERCY</h1>
		<h2>A little "BS game" - Scimo</h2>
	</div>

	<div class="game-frame">
		<canvas bind:this={canvas} width={CANVAS_W} height={CANVAS_H}></canvas>

		{#if won}
			<div class="overlay">
				<div class="win-star">★</div>
				<p class="win-text">You reached the star.</p>
				<button on:click={restart}>Play again</button>
			</div>
		{/if}
	</div>

	<p class="hint">Arrow keys / WASD to move, Space to jump. Jump on enemies to defeat them.</p>
	<p class="deaths">Deaths: {deaths}</p>
	<a href="/" class="scimo-net-redirect">→ SCIMO.NET</a>
</div>

<style>
	.wrap {
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 0.75rem;
	}

	.title {
		color: #fff;
		text-shadow: 0 2px 8px rgba(0, 0, 0, 0.6);
		margin-top: 3rem;
	}

	.game-frame {
		position: relative;
		width: 800px;
		max-width: 95vw;
		aspect-ratio: 800 / 300;
		border: 2px solid #fff;
	}

	canvas {
		width: 100%;
		height: 100%;
		display: block;
		image-rendering: pixelated;
		background: #000;
	}

	.overlay {
		position: absolute;
		inset: 0;
		background: rgba(0, 0, 0, 0.85);
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		gap: 0.75rem;
		color: #fff;
	}

	.win-star {
		font-size: 3rem;
		color: #fff;
		text-shadow:
			0 0 20px #fff,
			0 0 40px #fff;
	}

	.win-text {
		font-size: 1.1rem;
		letter-spacing: 0.05em;
	}

	.overlay button {
		background: #fff;
		color: #000;
		border: none;
		padding: 0.5rem 1.25rem;
		font-weight: bold;
		cursor: pointer;
		border-radius: 4px;
	}

	.hint,
	.deaths,
	.scimo-net-redirect {
		color: #aaa;
		font-size: 0.85rem;
		text-align: center;
	}
</style>
