<script lang="ts" module>
	export type Variant = 'DE9' | 'DE15' | 'DA15' | 'DA26' | 'DB25';
	export type Gender = 'male' | 'female';
	export type View = 'front' | 'back';

	export const PIN_COUNT: Record<Variant, number> = { DE9: 9, DE15: 15, DA15: 15, DA26: 26, DB25: 25 };
</script>

<script lang="ts">
	import { Tween } from 'svelte/motion';
	import { cubicOut } from 'svelte/easing';

	let {
		variant = 'DA15',
		gender = 'male',
		view = 'front',
		rotation = 0,
		maxWidth = 800,
		maxHeight = 600,
		used = [],
		animate = true
	}: {
		variant?: Variant;
		gender?: Gender;
		view?: View;
		rotation?: number;
		maxWidth?: number;
		maxHeight?: number;
		/** Pins to highlight; when non-empty, the other pins are dimmed. */
		used?: number[];
		/** When false, rotation and flip changes jump instead of animating. */
		animate?: boolean;
	} = $props();

	const usedSet = $derived(new Set(used));
	const dimUnused = $derived(usedSet.size > 0);

	type Pin = { n: number; x: number; y: number };

	const PITCH = 2.77;
	const ROW = 2.84;

	// Standard density: top row has one more pin than the bottom row, which is
	// offset by half a pitch.
	function standardPins(top: number): Pin[] {
		const pins: Pin[] = [];
		for (let i = 0; i < top; i++) pins.push({ n: i + 1, x: (i - (top - 1) / 2) * PITCH, y: -ROW / 2 });
		for (let i = 0; i < top - 1; i++) pins.push({ n: top + i + 1, x: (i - (top - 2) / 2) * PITCH, y: ROW / 2 });
		return pins;
	}

	// High density: three rows, 1.98mm apart, numbered row by row. Each row is
	// [pin count, offset in pitches]; the middle row is shifted towards pin 1.
	function hdPins(pitch: number, rows: [number, number][]): Pin[] {
		const pins: Pin[] = [];
		rows.forEach(([count, offset], r) => {
			for (let i = 0; i < count; i++) {
				pins.push({ n: pins.length + 1, x: (i - (count - 1) / 2 + offset) * pitch, y: (r - 1) * 1.98 });
			}
		});
		return pins;
	}

	const FLANGE_H = 12.55;
	const SHELL_H = 7.9;
	const HOLE_R = 1.55;

	// Geometry in millimetres (nominal shell sizes E, A and B), as seen looking
	// at the mating face of a male connector with the wide side of the D on top
	// (pin 1 at top-left).
	const GEOMETRY: Record<
		Variant,
		{ flangeW: number; holeSpacing: number; shellW: number; pinR: number; font: number; pins: Pin[] }
	> = {
		DE9: { flangeW: 30.81, holeSpacing: 24.99, shellW: 16.92, pinR: 1.05, font: 1.15, pins: standardPins(5) },
		DE15: { flangeW: 30.81, holeSpacing: 24.99, shellW: 16.92, pinR: 0.85, font: 0.95, pins: hdPins(2.29, [[5, 0], [5, -0.5], [5, 0]]) },
		DA15: { flangeW: 39.14, holeSpacing: 33.32, shellW: 25.25, pinR: 1.05, font: 1.15, pins: standardPins(8) },
		DA26: { flangeW: 39.14, holeSpacing: 33.32, shellW: 25.25, pinR: 0.85, font: 0.95, pins: hdPins(2.29, [[9, 0.25], [9, -0.25], [8, -0.25]]) },
		DB25: { flangeW: 53.04, holeSpacing: 47.04, shellW: 38.96, pinR: 1.05, font: 1.15, pins: standardPins(13) }
	};

	const g = $derived(GEOMETRY[variant]);

	// A male connector seen from the front has pin 1 at top-left. A female
	// connector seen from the front, or either seen from the back, is the mirror
	// image; looking at the back of a female brings it round again.
	const mirrored = $derived((gender === 'female') !== (view === 'back'));

	const angle = new Tween(0, { duration: 350, easing: cubicOut });
	const flip = new Tween(1, { duration: 350, easing: cubicOut });
	$effect(() => {
		angle.set(rotation, animate ? undefined : { duration: 0 });
	});
	$effect(() => {
		flip.set(mirrored ? -1 : 1, animate ? undefined : { duration: 0 });
	});

	// Map a point from connector space to screen space: mirror, then rotate.
	function place(x: number, y: number) {
		const a = (angle.current * Math.PI) / 180;
		const mx = x * flip.current;
		return {
			x: mx * Math.cos(a) - y * Math.sin(a),
			y: mx * Math.sin(a) + y * Math.cos(a)
		};
	}

	function roundedPath(pts: [number, number][], r: number) {
		let d = '';
		for (let i = 0; i < pts.length; i++) {
			const [px, py] = pts[(i - 1 + pts.length) % pts.length];
			const [cx, cy] = pts[i];
			const [nx, ny] = pts[(i + 1) % pts.length];
			const l1 = Math.hypot(px - cx, py - cy);
			const l2 = Math.hypot(nx - cx, ny - cy);
			const ax = cx + ((px - cx) / l1) * r;
			const ay = cy + ((py - cy) / l1) * r;
			const bx = cx + ((nx - cx) / l2) * r;
			const by = cy + ((ny - cy) / l2) * r;
			d += `${i === 0 ? 'M' : 'L'}${ax},${ay} Q${cx},${cy} ${bx},${by} `;
		}
		return d + 'Z';
	}

	// D-shaped shell: sides slope in at 10° towards the narrow bottom edge.
	function dShape(w: number, h: number, r: number) {
		const inset = h * Math.tan((10 * Math.PI) / 180);
		return roundedPath(
			[
				[-w / 2, -h / 2],
				[w / 2, -h / 2],
				[w / 2 - inset, h / 2],
				[-w / 2 + inset, h / 2]
			],
			r
		);
	}

	const shellOuter = $derived(dShape(g.shellW, SHELL_H, 1.6));
	const shellInner = $derived(dShape(g.shellW - 1.6, SHELL_H - 1.6, 1.1));
	const flange = $derived(
		roundedPath(
			[
				[-g.flangeW / 2, -FLANGE_H / 2],
				[g.flangeW / 2, -FLANGE_H / 2],
				[g.flangeW / 2, FLANGE_H / 2],
				[-g.flangeW / 2, FLANGE_H / 2]
			],
			1
		)
	);

	const labels = $derived(g.pins.map((p) => ({ n: p.n, ...place(p.x, p.y) })));

	// Square viewBox that holds the flange at any angle (its diagonal), so the
	// stage and drawing scale stay fixed while rotating.
	const MARGIN = 0.5;
	const box = $derived.by(() => {
		const side = Math.hypot(g.flangeW, FLANGE_H) + MARGIN * 2;
		return { w: side, h: side };
	});
	const scale = $derived(Math.max(0, Math.min(maxWidth / box.w, maxHeight / box.h)));
</script>

<svg
	viewBox="{-box.w / 2} {-box.h / 2} {box.w} {box.h}"
	width={box.w * scale}
	height={box.h * scale}
	role="img"
	aria-label="{variant} {gender} connector, {view} view, rotated {rotation}°"
>
	<g transform="rotate({angle.current}) scale({flip.current}, 1)">
		<path d={flange} class="flange" />
		{#each [-1, 1] as side}
			<circle cx={(side * g.holeSpacing) / 2} cy="0" r={HOLE_R} class="hole" />
		{/each}
		<path d={shellOuter} class="shell" class:back={view === 'back'} />
		<path d={shellInner} class="insulator" />
		{#each g.pins as p (p.n)}
			<circle
				cx={p.x}
				cy={p.y}
				r={g.pinR}
				class={view === 'back' ? 'cup' : gender === 'male' ? 'pin' : 'socket'}
				class:first={p.n === 1}
				class:used={usedSet.has(p.n)}
				class:dim={dimUnused && !usedSet.has(p.n)}
			/>
		{/each}
	</g>
	{#each labels as l (l.n)}
		<text
			x={l.x}
			y={l.y}
			font-size={g.font}
			class:used={usedSet.has(l.n)}
			class:dim={dimUnused && !usedSet.has(l.n)}>{l.n}</text
		>
	{/each}
</svg>

<style>
	svg {
		display: block;
	}
	.flange {
		fill: var(--metal);
		stroke: var(--metal-edge);
		stroke-width: 0.25;
	}
	.hole {
		fill: var(--panel);
		stroke: var(--metal-edge);
		stroke-width: 0.25;
	}
	.shell {
		fill: var(--metal-light);
		stroke: var(--metal-edge);
		stroke-width: 0.25;
	}
	.shell.back {
		fill: var(--metal);
	}
	.insulator {
		fill: var(--insulator);
		stroke: var(--metal-edge);
		stroke-width: 0.15;
	}
	.pin {
		fill: var(--gold);
		stroke: var(--gold-edge);
		stroke-width: 0.12;
	}
	.socket {
		fill: var(--insulator-dark);
		stroke: var(--gold);
		stroke-width: 0.3;
	}
	.cup {
		fill: var(--silver);
		stroke: var(--metal-edge);
		stroke-width: 0.12;
	}
	circle.used {
		fill: var(--used);
		stroke: var(--used-edge);
		stroke-width: 0.3;
	}
	circle.dim {
		opacity: 0.35;
	}
	circle.first {
		stroke: var(--accent);
		stroke-width: 0.35;
	}
	text {
		text-anchor: middle;
		dominant-baseline: central;
		font-family: ui-sans-serif, system-ui, sans-serif;
		font-weight: 700;
		fill: var(--label);
		paint-order: stroke;
		stroke: var(--label-halo);
		stroke-width: 0.25;
		user-select: none;
	}
	text.used {
		fill: var(--used-label);
		stroke: none;
	}
	text.dim {
		opacity: 0.45;
	}
</style>
