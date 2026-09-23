<script lang="ts">
	import { onMount, tick } from 'svelte';
	import { replaceState } from '$app/navigation';
	import DSub, { PIN_COUNT, type Gender, type Variant, type View } from '$lib/DSub.svelte';

	let variant = $state<Variant>('DA15');
	let gender = $state<Gender>('male');
	let view = $state<View>('back');
	// Kept unbounded so the animation always turns the short way round.
	let rotation = $state(0);

	const displayAngle = $derived(((rotation % 360) + 360) % 360);

	// Comma, space or newline separated pin numbers; ranges like 2-5 allowed.
	let usedText = $state('');
	const parsed = $derived.by(() => {
		const max = PIN_COUNT[variant];
		const pins = new Set<number>();
		const invalid: string[] = [];
		for (const token of usedText.split(/[\s,;]+/).filter(Boolean)) {
			const m = token.match(/^(\d+)(?:-(\d+))?$/);
			const from = m ? Number(m[1]) : NaN;
			const to = m?.[2] ? Number(m[2]) : from;
			if (!m || from < 1 || to > max || from > to) {
				invalid.push(token);
				continue;
			}
			for (let n = from; n <= to; n++) pins.add(n);
		}
		return { pins: [...pins].sort((a, b) => a - b), invalid, max };
	});

	// Field values are mirrored into the URL query so a link can be shared.
	// Defaults are left out to keep links short.
	const VARIANTS: Record<string, Variant> = { de9: 'DE9', de15: 'DE15', da15: 'DA15', da26: 'DA26', db25: 'DB25' };
	const DEFAULT_TITLE = 'D-Sub connector pinout';
	let title = $state(DEFAULT_TITLE);
	let urlReady = $state(false);
	let animate = $state(false);

	onMount(async () => {
		const q = new URLSearchParams(location.search);
		const type = VARIANTS[q.get('type')?.toLowerCase() ?? ''];
		if (type) variant = type;
		const g = q.get('gender');
		if (g === 'male' || g === 'female') gender = g;
		const v = q.get('view');
		if (v === 'back' || v === 'front') view = v;
		const r = Number(q.get('rotate'));
		if ([90, 180, 270].includes(r)) rotation = r;
		usedText = q.get('pins') ?? '';
		title = q.get('title')?.trim() || DEFAULT_TITLE;
		urlReady = true;
		// Show a shared link's rotation/view straight away, then animate changes.
		await tick();
		animate = true;
	});

	$effect(() => {
		if (!urlReady) return;
		const q = new URLSearchParams();
		if (variant !== 'DA15') q.set('type', variant.toLowerCase());
		if (gender !== 'male') q.set('gender', gender);
		if (view !== 'back') q.set('view', view);
		if (displayAngle !== 0) q.set('rotate', String(displayAngle));
		if (usedText.trim()) q.set('pins', usedText.trim());
		if (title.trim() && title.trim() !== DEFAULT_TITLE) q.set('title', title.trim());
		const search = q.size ? `?${q}` : '';
		if (search !== location.search) replaceState(location.pathname + search, {});
	});

	function titleKeydown(e: KeyboardEvent) {
		if (e.key === 'Enter' || e.key === 'Escape') {
			e.preventDefault();
			(e.currentTarget as HTMLElement).blur();
		}
	}

	function titleBlur() {
		title = title.replace(/\s+/g, ' ').trim() || DEFAULT_TITLE;
	}

	let shareStatus = $state<'idle' | 'copied' | 'failed'>('idle');
	let shareTimer: ReturnType<typeof setTimeout>;
	async function share() {
		try {
			await navigator.clipboard.writeText(location.href);
			shareStatus = 'copied';
		} catch {
			shareStatus = 'failed';
			window.prompt('Copy this link:', location.href);
		}
		clearTimeout(shareTimer);
		shareTimer = setTimeout(() => (shareStatus = 'idle'), 2000);
	}

	const STAGE_PAD = 16;
	const BUTTON_ROW = 64;
	let canvasWidth = $state(0);
	let canvasHeight = $state(0);

	// Back to 0° by the shortest way round.
	function resetRotation() {
		rotation = Math.round(rotation / 360) * 360;
	}

	function onkeydown(e: KeyboardEvent) {
		if (e.altKey || e.ctrlKey || e.metaKey || e.shiftKey) return;
		// Arrow keys move between radio buttons, so leave form fields alone.
		const t = e.target as HTMLElement;
		if (t.closest('input, select, textarea, [contenteditable]')) return;
		if (e.key === 'ArrowLeft') rotation -= 90;
		else if (e.key === 'ArrowRight') rotation += 90;
		else if (e.key === 'ArrowUp') rotation += 180;
		else if (e.key === 'ArrowDown') rotation -= 180;
		else if (e.key === 'Home') resetRotation();
		else return;
		e.preventDefault();
	}
</script>

<svelte:window {onkeydown} />

<svelte:head>
	<title>{title.trim() && title.trim() !== DEFAULT_TITLE ? title.trim() : 'D-Sub Pinouts'}</title>
</svelte:head>

<main>
	<header>
		<h1
			contenteditable="plaintext-only"
			spellcheck="false"
			title="Click to edit the title"
			bind:textContent={title}
			onkeydown={titleKeydown}
			onblur={titleBlur}
		></h1>
		<button class="share" onclick={share} aria-live="polite">
			{shareStatus === 'copied' ? 'Link copied' : shareStatus === 'failed' ? 'Copy failed' : 'Share'}
		</button>
	</header>

	<div class="layout">
		<div class="controls">
			<fieldset>
				<legend>Type</legend>
				<label><input type="radio" name="variant" autocomplete="off" bind:group={variant} value="DE9" /> DE-9</label>
				<label><input type="radio" name="variant" autocomplete="off" bind:group={variant} value="DA15" /> DA-15</label>
				<label><input type="radio" name="variant" autocomplete="off" bind:group={variant} value="DE15" /> DE-15 HD (VGA)</label>
				<label><input type="radio" name="variant" autocomplete="off" bind:group={variant} value="DB25" /> DB-25</label>
				<label><input type="radio" name="variant" autocomplete="off" bind:group={variant} value="DA26" /> DA-26 HD</label>
			</fieldset>

			<fieldset>
				<legend>Gender</legend>
				<label><input type="radio" name="gender" autocomplete="off" bind:group={gender} value="male" /> Male (pins)</label>
				<label><input type="radio" name="gender" autocomplete="off" bind:group={gender} value="female" /> Female (sockets)</label>
			</fieldset>

			<fieldset>
				<legend>View</legend>
				<label><input type="radio" name="view" autocomplete="off" bind:group={view} value="back" /> Back (wiring side)</label>
				<label><input type="radio" name="view" autocomplete="off" bind:group={view} value="front" /> Front (mating face)</label>
			</fieldset>

			<fieldset class="used">
				<legend>Pins used</legend>
				<textarea
					bind:value={usedText}
					rows="3"
					placeholder="e.g. 1, 2, 5, 7-9"
					spellcheck="false"
					aria-describedby="used-status"
				></textarea>
				<small id="used-status" class:error={parsed.invalid.length > 0}>
					{#if parsed.invalid.length}
						Ignored: {parsed.invalid.join(', ')} (pins are 1–{parsed.max})
					{:else if parsed.pins.length}
						{parsed.pins.length} of {parsed.max} pins used
					{:else}
						Comma separated; ranges like 7-9 work
					{/if}
				</small>
			</fieldset>
		</div>

		<div class="drawing">
			<div class="stage">
				<button
					class="corner left"
					onclick={() => (rotation -= 90)}
					title="Rotate 90° anticlockwise (←)"
					aria-label="Rotate 90° anticlockwise">⟲</button
				>
				<button
					class="angle"
					onclick={resetRotation}
					disabled={displayAngle === 0}
					title="Reset rotation (Home)">{displayAngle}°</button
				>
				<button
					class="corner right"
					onclick={() => (rotation += 90)}
					title="Rotate 90° clockwise (→)"
					aria-label="Rotate 90° clockwise">⟳</button
				>
				<!-- Absolutely positioned so the drawing's size never feeds back into the stage's. -->
				<div
					class="canvas"
					style:inset="{BUTTON_ROW}px {STAGE_PAD}px {STAGE_PAD}px"
					bind:clientWidth={canvasWidth}
					bind:clientHeight={canvasHeight}
				>
					<DSub {variant} {gender} {view} {rotation} maxWidth={canvasWidth} maxHeight={canvasHeight} used={parsed.pins} {animate} />
				</div>
			</div>

			<p class="note">Pin 1 is outlined in red. Pin numbers stay upright whatever the rotation. Use ← and → to rotate 90°, ↑ and ↓ for 180°, Home to reset.</p>
		</div>
	</div>
</main>

<style>
	:global(:root) {
		--bg: #f6f5f2;
		--fg: #1d1d1f;
		--muted: #6b6b70;
		--panel: #ffffff;
		--border: #d9d7d2;
		--metal: #b9bcc2;
		--metal-light: #d3d6db;
		--metal-edge: #6f747c;
		--silver: #e4e6ea;
		--insulator: #2f3136;
		--insulator-dark: #16171a;
		--gold: #d9ae3f;
		--gold-edge: #8a6a17;
		--accent: #d23c3c;
		--label: #111;
		--label-halo: #fff;
		--used: #1f8a4c;
		--used-edge: #0f5a30;
		--used-label: #fff;
		--error: #c0392b;
	}
	@media (prefers-color-scheme: dark) {
		:global(:root) {
			--bg: #16171a;
			--fg: #ececef;
			--muted: #9a9aa2;
			--panel: #202226;
			--border: #34363c;
			--metal: #7d828b;
			--metal-light: #9a9fa8;
			--metal-edge: #4a4e55;
			--insulator: #0d0e10;
			--used: #2fb368;
			--used-edge: #7be0a6;
			--used-label: #06210f;
			--error: #ff7b6b;
		}
	}
	:global(body) {
		margin: 0;
		background: var(--bg);
		color: var(--fg);
		font-family: ui-sans-serif, system-ui, sans-serif;
	}
	main {
		max-width: 960px;
		min-height: 100dvh;
		box-sizing: border-box;
		margin: 0 auto;
		padding: 24px 16px 8px;
		display: flex;
		flex-direction: column;
	}
	header {
		display: flex;
		align-items: center;
		gap: 12px;
		margin: 0 0 16px;
	}
	h1 {
		flex: 1;
		min-width: 0;
		font-size: 1.5rem;
		margin: 0;
		padding: 2px 6px;
		margin-left: -6px;
		border-radius: 6px;
		outline: 2px dashed transparent;
		outline-offset: 2px;
		cursor: text;
		overflow-wrap: anywhere;
	}
	h1:hover {
		outline-color: var(--muted);
	}
	/* Match the Pins used textarea: the browser's own focus ring. */
	h1:focus {
		outline: auto;
		outline-offset: 0;
		background: var(--bg);
	}
	.share {
		flex: none;
		min-width: 7.5em;
		height: 36px;
		font-weight: 600;
	}
	.layout {
		flex: 1;
		display: grid;
		grid-template-rows: auto 1fr;
		gap: 16px;
	}
	.drawing {
		display: flex;
		flex-direction: column;
	}
	.controls {
		display: flex;
		flex-wrap: wrap;
		gap: 12px;
	}
	/* Very wide screens: controls become a column on the left of the stage. */
	@media (min-width: 1400px) {
		main {
			max-width: 1600px;
		}
		.layout {
			grid-template-columns: 220px minmax(0, 1fr);
			grid-template-rows: 1fr;
		}
		.controls {
			flex-direction: column;
			flex-wrap: nowrap;
			align-self: start;
		}
	}
	fieldset {
		background: var(--panel);
		border: 1px solid var(--border);
		border-radius: 8px;
		padding: 8px 12px 10px;
		display: flex;
		flex-direction: column;
		gap: 4px;
		min-width: 0;
	}
	legend {
		font-size: 0.8rem;
		color: var(--muted);
		padding: 0 4px;
	}
	label {
		display: flex;
		align-items: center;
		gap: 6px;
		cursor: pointer;
	}
	.used {
		flex: 1 1 220px;
	}
	textarea {
		font: inherit;
		font-family: ui-monospace, monospace;
		font-size: 0.9rem;
		padding: 6px 8px;
		border-radius: 6px;
		border: 1px solid var(--border);
		background: var(--bg);
		color: var(--fg);
		resize: vertical;
		min-height: 3.5em;
	}
	small {
		color: var(--muted);
		font-size: 0.8rem;
	}
	small.error {
		color: var(--error);
	}
	button {
		font: inherit;
		padding: 4px 10px;
		border-radius: 6px;
		border: 1px solid var(--border);
		background: var(--bg);
		color: var(--fg);
		cursor: pointer;
	}
	button:disabled {
		opacity: 0.5;
		cursor: default;
	}
	.stage {
		position: relative;
		background: var(--panel);
		border: 1px solid var(--border);
		border-radius: 12px;
		flex: 1;
		min-height: 320px;
	}
	.canvas {
		position: absolute;
		display: grid;
		place-items: center;
	}
	.stage button {
		position: absolute;
		top: 10px;
		height: 44px;
		line-height: 1;
	}
	.corner {
		width: 44px;
		padding: 0;
		font-size: 1.6rem;
		border-radius: 8px;
	}
	.corner.left {
		left: 10px;
	}
	.corner.right {
		right: 10px;
	}
	.angle {
		left: 50%;
		transform: translateX(-50%);
		font-size: 0.85rem;
		font-variant-numeric: tabular-nums;
	}
	.note {
		margin: 8px 0 0;
		color: var(--muted);
		font-size: 0.9rem;
		text-align: center;
	}
</style>
