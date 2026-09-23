<script lang="ts">
	import DSub15, { type Gender, type Variant, type View } from '$lib/DSub15.svelte';

	let variant = $state<Variant>('DA15');
	let gender = $state<Gender>('male');
	let view = $state<View>('front');
	// Kept unbounded so the animation always turns the short way round.
	let rotation = $state(0);

	const displayAngle = $derived(((rotation % 360) + 360) % 360);
</script>

<svelte:head>
	<title>15-pin D-Sub</title>
</svelte:head>

<main>
	<h1>15-pin D-Sub connector</h1>

	<div class="controls">
		<fieldset>
			<legend>Type</legend>
			<label><input type="radio" bind:group={variant} value="DA15" /> DA-15 (2 rows)</label>
			<label><input type="radio" bind:group={variant} value="DE15" /> DE-15 HD (3 rows)</label>
		</fieldset>

		<fieldset>
			<legend>Gender</legend>
			<label><input type="radio" bind:group={gender} value="male" /> Male (pins)</label>
			<label><input type="radio" bind:group={gender} value="female" /> Female (sockets)</label>
		</fieldset>

		<fieldset>
			<legend>View</legend>
			<label><input type="radio" bind:group={view} value="front" /> Front (mating face)</label>
			<label><input type="radio" bind:group={view} value="back" /> Back (wiring side)</label>
		</fieldset>

		<fieldset>
			<legend>Rotation · {displayAngle}°</legend>
			<div class="buttons">
				<button onclick={() => (rotation -= 90)} title="Rotate 90° anticlockwise">⟲ 90°</button>
				<button onclick={() => (rotation += 90)} title="Rotate 90° clockwise">⟳ 90°</button>
				<button onclick={() => (rotation = Math.round(rotation / 360) * 360)} disabled={displayAngle === 0}>
					Reset
				</button>
			</div>
		</fieldset>
	</div>

	<div class="stage">
		<DSub15 {variant} {gender} {view} {rotation} />
	</div>

	<p class="note">Pin 1 is outlined in red. Pin numbers stay upright whatever the rotation.</p>
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
		margin: 0 auto;
		padding: 24px 16px;
	}
	h1 {
		font-size: 1.5rem;
		margin: 0 0 16px;
	}
	.controls {
		display: flex;
		flex-wrap: wrap;
		gap: 12px;
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
	.buttons {
		display: flex;
		gap: 6px;
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
		margin-top: 16px;
		background: var(--panel);
		border: 1px solid var(--border);
		border-radius: 12px;
		aspect-ratio: 1;
		max-height: 70vh;
		margin-inline: auto;
		width: 100%;
		max-width: 70vh;
	}
	.note {
		color: var(--muted);
		font-size: 0.9rem;
		text-align: center;
	}
</style>
