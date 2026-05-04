<script lang="ts">
	import Folder from "$lib/components/Folder.svelte";

	let containerEl = $state<HTMLDivElement>();
	let isSelecting = $state(false);
	let selectionStartX = $state(0);
	let selectionStartY = $state(0);
	let selectionX = $state(0);
	let selectionY = $state(0);
	let selectionWidth = $state(0);
	let selectionHeight = $state(0);

	const folders = [
		{
			id: "spotify",
			text: "spotify",
			url: "https://open.spotify.com/album/4IynMpI5KfGbSPBa7vv1wu",
			offsetX: -20,
			offsetY: -120
		},
		{
			id: "apple-music",
			text: "apple music",
			url: "https://music.apple.com/it/album/pink-arc-ce79b1-ep/1893284584",
			offsetX: 100,
			offsetY: -100
		},
		{
			id: "amazon-music",
			text: "amazon music",
			url: "https://music.apple.com/it/album/pink-arc-ce79b1-ep/1893284584",
			offsetX: -120,
			offsetY: -40
		},
		{
			id: "youtube",
			text: "youtube",
			url: "https://music.youtube.com/playlist?list=OLAK5uy_nTan8iGTWVi9maiwyLoq01Iezxrr_Qh_Q",
			offsetX: -120,
			offsetY: 90
		},
		{
			id: "tidal",
			text: "tidal",
			url: "https://tidal.com/album/515376749",
			offsetX: 120,
			offsetY: 180
		},
		{
			id: "deezer",
			text: "deezer",
			url: "https://www.deezer.com/en/album/960022261",
			offsetX: 0,
			offsetY: 140
		}
	] as const;

	function clamp(value: number, min: number, max: number) {
		return Math.min(Math.max(value, min), max);
	}

	function getRelativePoint(clientX: number, clientY: number) {
		if (!containerEl) return { x: 0, y: 0 };

		const rect = containerEl.getBoundingClientRect();
		return {
			x: clamp(clientX - rect.left, 0, rect.width),
			y: clamp(clientY - rect.top, 0, rect.height)
		};
	}

	function handleContainerPointerDown(event: PointerEvent) {
		if (!containerEl || event.button !== 0) return;
		if (event.target instanceof Element && event.target.closest("[data-no-selection]")) return;

		const point = getRelativePoint(event.clientX, event.clientY);
		isSelecting = true;
		selectionStartX = point.x;
		selectionStartY = point.y;
		selectionX = point.x;
		selectionY = point.y;
		selectionWidth = 0;
		selectionHeight = 0;
		containerEl.setPointerCapture(event.pointerId);
	}

	function handleContainerPointerMove(event: PointerEvent) {
		if (!isSelecting) return;

		const point = getRelativePoint(event.clientX, event.clientY);
		selectionX = Math.min(selectionStartX, point.x);
		selectionY = Math.min(selectionStartY, point.y);
		selectionWidth = Math.abs(point.x - selectionStartX);
		selectionHeight = Math.abs(point.y - selectionStartY);
	}

	function handleContainerPointerUp(event: PointerEvent) {
		if (!containerEl) return;
		isSelecting = false;
		if (containerEl.hasPointerCapture(event.pointerId)) {
			containerEl.releasePointerCapture(event.pointerId);
		}
	}

</script>

<div
	bind:this={containerEl}
	class="relative min-h-svh select-none p-2 flex flex-col justify-between"
	role="application"
	aria-label="Desktop area"
	onpointerdown={handleContainerPointerDown}
	onpointermove={handleContainerPointerMove}
	onpointerup={handleContainerPointerUp}
	onpointercancel={handleContainerPointerUp}
>
	<div class="-space-y-9">
		<h1 class="text-6xl tracking-wide leading-15">nomorecho</h1>
		<p class="text-6xl opacity-50 tracking-wide leading-15">nomorecho</p>
		<p class="text-6xl opacity-30 tracking-wide leading-15">nomorecho</p>
		<p class="text-6xl opacity-10 tracking-wide leading-15">nomorecho</p>
	</div>

	{#each folders as folder (folder.id)}
		<Folder
			bind:containerEl
			text={folder.text}
			centerOffsetX={folder.offsetX}
			centerOffsetY={folder.offsetY}
			onclick={() => window.open(folder.url, '_blank', 'noopener,noreferrer')}
		/>
	{/each}

	{#if isSelecting && selectionWidth > 0 && selectionHeight > 0}
		<div class="pointer-events-none absolute inset-0 z-30">
			<div
				class="absolute rounded-xs border border-blue-400/90 bg-blue-400/20 shadow-[inset_0_0_0_1px_rgba(255,255,255,0.35)]"
				style={`left:${selectionX}px;top:${selectionY}px;width:${selectionWidth}px;height:${selectionHeight}px;`}
			></div>
		</div>
	{/if}

	<div class="text-pink-arc text-4xl text-right">
		<p class="leading-13">pink arc [#ce79b1]<br />out now</p>
	</div>

	<footer class="overflow-hidden whitespace-nowrap">
		<div class="footer-marquee text-4xl">
			<span>WE WANTED TO USE SQUARE BRACKETS BUT DISTROKID DIDN'T LET US</span>
			<span aria-hidden="true">WE WANTED TO USE SQUARE BRACKETS BUT DISTROKID DIDN'T LET US</span>
			<span aria-hidden="true">WE WANTED TO USE SQUARE BRACKETS BUT DISTROKID DIDN'T LET US</span>
			<span aria-hidden="true">WE WANTED TO USE SQUARE BRACKETS BUT DISTROKID DIDN'T LET US</span>
			<span aria-hidden="true">WE WANTED TO USE SQUARE BRACKETS BUT DISTROKID DIDN'T LET US</span>
			<span aria-hidden="true">WE WANTED TO USE SQUARE BRACKETS BUT DISTROKID DIDN'T LET US</span>
		</div>
	</footer>
</div>

<style>
	.footer-marquee {
		display: inline-flex;
		gap: 2rem;
		min-width: max-content;
		animation: marquee 40s linear infinite;
	}

	@keyframes marquee {
		from {
			transform: translateX(0);
		}
		to {
			transform: translateX(calc(-50% - 1rem));
		}
	}
</style>
