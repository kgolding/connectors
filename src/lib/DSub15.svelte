<script lang="ts" module>
	export type Variant = 'DA15' | 'DE15';
	export type Gender = 'male' | 'female';
	export type View = 'front' | 'back';
</script>

<script lang="ts">
	import { Tween } from 'svelte/motion';
	import { cubicOut } from 'svelte/easing';

	let {
		variant = 'DA15',
		gender = 'male',
		view = 'front',
		rotation = 0
	}: { variant?: Variant; gender?: Gender; view?: View; rotation?: number } = $props();

	type Pin = { n: number; x: number; y: number };

	// Geometry in millimetres, as seen looking at the mating face of a male
	// connector with the wide side of the D on top (pin 1 at top-left).
	const GEOMETRY = {
		DA15: {
			flangeW: 53.0,
			flangeH: 12.55,
			holeSpacing: 47.04,
			holeR: 1.55,
			shellW: 33.3,
			shellH: 8.0,
			pinR: 1.05,
			font: 1.15,
			pins: (() => {
				const p = 2.77;
				const dy = 2.84 / 2;
				const pins: Pin[] = [];
				for (let i = 0; i < 8; i++) pins.push({ n: i + 1, x: (i - 3.5) * p, y: -dy });
				for (let i = 0; i < 7; i++) pins.push({ n: i + 9, x: (i - 3) * p, y: dy });
				return pins;
			})()
		},
		DE15: {
			flangeW: 30.81,
			flangeH: 12.55,
			holeSpacing: 25.0,
			holeR: 1.55,
			shellW: 18.2,
			shellH: 8.7,
			pinR: 0.85,
			font: 0.95,
			pins: (() => {
				const p = 2.29;
				const dy = 1.98;
				const pins: Pin[] = [];
				for (let i = 0; i < 5; i++) pins.push({ n: i + 1, x: (i - 2) * p, y: -dy });
				for (let i = 0; i < 5; i++) pins.push({ n: i + 6, x: (i - 2.5) * p, y: 0 });
				for (let i = 0; i < 5; i++) pins.push({ n: i + 11, x: (i - 2) * p, y: dy });
				return pins;
			})()
		}
	} as const;

	const g = $derived(GEOMETRY[variant]);

	// A male connector seen from the front has pin 1 at top-left. A female
	// connector seen from the front, or either seen from the back, is the mirror
	// image; looking at the back of a female brings it round again.
	const mirrored = $derived((gender === 'female') !== (view === 'back'));

	const angle = new Tween(0, { duration: 350, easing: cubicOut });
	const flip = new Tween(1, { duration: 350, easing: cubicOut });
	$effect(() => {
		angle.target = rotation;
	});
	$effect(() => {
		flip.target = mirrored ? -1 : 1;
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

	const shellOuter = $derived(dShape(g.shellW, g.shellH, 1.6));
	const shellInner = $derived(dShape(g.shellW - 1.6, g.shellH - 1.6, 1.1));
	const flange = $derived(
		roundedPath(
			[
				[-g.flangeW / 2, -g.flangeH / 2],
				[g.flangeW / 2, -g.flangeH / 2],
				[g.flangeW / 2, g.flangeH / 2],
				[-g.flangeW / 2, g.flangeH / 2]
			],
			1
		)
	);

	const labels = $derived(g.pins.map((p) => ({ n: p.n, ...place(p.x, p.y) })));

	// Square viewBox big enough for any rotation.
	const half = $derived(g.flangeW / 2 + 2);
</script>

<svg
	viewBox="{-half} {-half} {half * 2} {half * 2}"
	role="img"
	aria-label="{variant} {gender} connector, {view} view, rotated {rotation}°"
>
	<g transform="rotate({angle.current}) scale({flip.current}, 1)">
		<path d={flange} class="flange" />
		{#each [-1, 1] as side}
			<circle cx={(side * g.holeSpacing) / 2} cy="0" r={g.holeR} class="hole" />
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
			/>
		{/each}
	</g>
	{#each labels as l (l.n)}
		<text x={l.x} y={l.y} font-size={g.font} class:first={l.n === 1}>{l.n}</text>
	{/each}
</svg>

<style>
	svg {
		width: 100%;
		height: 100%;
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
</style>
