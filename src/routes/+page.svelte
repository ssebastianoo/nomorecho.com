<script lang="ts">
	import { browser } from "$app/environment";
	import folderIcon from "$lib/assets/folder.png";

	let containerEl = $state<HTMLDivElement>();
	let folderEl = $state<HTMLDivElement>();
	let alertEl = $state<HTMLDivElement>();
	let folderX = $state(0);
	let folderY = $state(0);
	let dragOffsetX = $state(0);
	let dragOffsetY = $state(0);
	let isDragging = $state(false);
	let folderActive = $state(false);
	let showAlert = $state(false);
	let alertX = $state(0);
	let alertY = $state(0);
	let alertDragOffsetX = $state(0);
	let alertDragOffsetY = $state(0);
	let isAlertDragging = $state(false);
	let isSelecting = $state(false);
	let selectionStartX = $state(0);
	let selectionStartY = $state(0);
	let selectionX = $state(0);
	let selectionY = $state(0);
	let selectionWidth = $state(0);
	let selectionHeight = $state(0);

	function clamp(value: number, min: number, max: number) {
		return Math.min(Math.max(value, min), max);
	}

	function moveTo(clientX: number, clientY: number) {
		if (!containerEl || !folderEl) return;

		const containerRect = containerEl.getBoundingClientRect();
		const folderRect = folderEl.getBoundingClientRect();
		const maxX = containerRect.width - folderRect.width;
		const maxY = containerRect.height - folderRect.height;

		folderX = clamp(clientX - containerRect.left - dragOffsetX, 0, maxX);
		folderY = clamp(clientY - containerRect.top - dragOffsetY, 0, maxY);
	}

	function centerFolder() {
		if (!containerEl || !folderEl) return;

		const containerRect = containerEl.getBoundingClientRect();
		const folderRect = folderEl.getBoundingClientRect();
		folderX = (containerRect.width - folderRect.width) / 2;
		folderY = (containerRect.height - folderRect.height) / 2;
	}

	function moveAlertTo(clientX: number, clientY: number) {
		if (!containerEl || !alertEl) return;

		const containerRect = containerEl.getBoundingClientRect();
		const alertRect = alertEl.getBoundingClientRect();
		const maxX = containerRect.width - alertRect.width;
		const maxY = containerRect.height - alertRect.height;

		alertX = clamp(clientX - containerRect.left - alertDragOffsetX, 0, maxX);
		alertY = clamp(clientY - containerRect.top - alertDragOffsetY, 0, maxY);
	}

	function centerAlert() {
		if (!containerEl || !alertEl) return;

		const containerRect = containerEl.getBoundingClientRect();
		const alertRect = alertEl.getBoundingClientRect();
		alertX = (containerRect.width - alertRect.width) / 2;
		alertY = (containerRect.height - alertRect.height) / 2;
	}

	function handlePointerDown(event: PointerEvent) {
		if (!folderEl) return;

		const rect = folderEl.getBoundingClientRect();
		folderActive = true;
		isDragging = true;
		dragOffsetX = event.clientX - rect.left;
		dragOffsetY = event.clientY - rect.top;
		folderEl.setPointerCapture(event.pointerId);
	}

	function handlePointerMove(event: PointerEvent) {
		if (!isDragging) return;
		moveTo(event.clientX, event.clientY);
	}

	function handlePointerUp(event: PointerEvent) {
		if (!folderEl) return;
		isDragging = false;
		if (folderEl.hasPointerCapture(event.pointerId)) {
			folderEl.releasePointerCapture(event.pointerId);
		}
	}

	function handleFolderDoubleClick() {
		showAlert = true;
		requestAnimationFrame(centerAlert);
	}

	function handleAlertPointerDown(event: PointerEvent) {
		if (!alertEl) return;
		if (event.target instanceof Element && event.target.closest("button, a")) return;

		const rect = alertEl.getBoundingClientRect();
		isAlertDragging = true;
		alertDragOffsetX = event.clientX - rect.left;
		alertDragOffsetY = event.clientY - rect.top;
		alertEl.setPointerCapture(event.pointerId);
	}

	function handleAlertPointerMove(event: PointerEvent) {
		if (!isAlertDragging) return;
		moveAlertTo(event.clientX, event.clientY);
	}

	function handleAlertPointerUp(event: PointerEvent) {
		if (!alertEl) return;
		isAlertDragging = false;
		if (alertEl.hasPointerCapture(event.pointerId)) {
			alertEl.releasePointerCapture(event.pointerId);
		}
	}

	function getRelativePoint(clientX: number, clientY: number) {
		if (!containerEl) return { x: 0, y: 0 };

		const rect = containerEl.getBoundingClientRect();
		return {
			x: clamp(clientX - rect.left, 0, rect.width),
			y: clamp(clientY - rect.top, 0, rect.height)
		};
	}

	function isFolderInsideSelection() {
		if (!containerEl || !folderEl || selectionWidth === 0 || selectionHeight === 0) return false;

		const containerRect = containerEl.getBoundingClientRect();
		const folderRect = folderEl.getBoundingClientRect();
		const folderLeft = folderRect.left - containerRect.left;
		const folderTop = folderRect.top - containerRect.top;
		const folderRight = folderLeft + folderRect.width;
		const folderBottom = folderTop + folderRect.height;
		const selectionRight = selectionX + selectionWidth;
		const selectionBottom = selectionY + selectionHeight;

		return (
			folderLeft < selectionRight &&
			folderRight > selectionX &&
			folderTop < selectionBottom &&
			folderBottom > selectionY
		);
	}

	function handleContainerPointerDown(event: PointerEvent) {
		if (!containerEl || event.button !== 0) return;
		if (event.target instanceof Element && event.target.closest("[data-no-selection]")) return;

		folderActive = false;
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
		folderActive = isFolderInsideSelection();
	}

	function handleContainerPointerUp(event: PointerEvent) {
		if (!containerEl) return;
		isSelecting = false;
		if (containerEl.hasPointerCapture(event.pointerId)) {
			containerEl.releasePointerCapture(event.pointerId);
		}
	}

	$effect(() => {
		if (!browser || !containerEl || !folderEl) return;

		requestAnimationFrame(centerFolder);

		const handleResize = () => centerFolder();
		window.addEventListener("resize", handleResize);
		return () => window.removeEventListener("resize", handleResize);
	});
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
	{#if showAlert}
		<div
			bind:this={alertEl}
			data-no-selection
			class="absolute z-999 flex cursor-grab flex-col gap-5 rounded-3xl border border-black/10 bg-black/50 px-5 py-3 text-white ring ring-white/50 ring-inset backdrop-blur-[2px] active:cursor-grabbing"
			style={`left: ${alertX}px; top: ${alertY}px; touch-action: none;`}
			role="dialog"
			aria-modal="false"
			aria-label="Release notice"
			tabindex="0"
			onpointerdown={handleAlertPointerDown}
			onpointermove={handleAlertPointerMove}
			onpointerup={handleAlertPointerUp}
			onpointercancel={handleAlertPointerUp}
		>
			<p>This item won't exist before april 26th @ midnight</p>
			<div class="flex gap-2">
				<button
					class="bg-white/15 cursor-pointer py-1 px-2 rounded-full flex-1"
					onclick={() => (showAlert = false)}>Cancel</button
				>
				<a
					href="https://www.youtube.com/watch?v=Y9ilsgeylAk"
					target="_blank"
					class="bg-pink-arc text-center py-1 px-2 rounded-full flex-1">F**k u</a
				>
			</div>
		</div>
	{/if}

	<div class="-space-y-9">
		<h1 class="text-6xl tracking-wide leading-15">nomorecho</h1>
		<p class="text-6xl opacity-50 tracking-wide leading-15">nomorecho</p>
		<p class="text-6xl opacity-30 tracking-wide leading-15">nomorecho</p>
		<p class="text-6xl opacity-10 tracking-wide leading-15">nomorecho</p>
	</div>

	<div class="pointer-events-none absolute inset-0 z-10">
		<div
			bind:this={folderEl}
			data-no-selection
			class="group pointer-events-auto absolute flex cursor-grab flex-col items-center gap-2 select-none active:cursor-grabbing"
			style={`left: ${folderX}px; top: ${folderY}px; touch-action: none;`}
			role="button"
			tabindex="0"
			aria-label="Draggable folder"
			onpointerdown={handlePointerDown}
			onpointermove={handlePointerMove}
			onpointerup={handlePointerUp}
			onpointercancel={handlePointerUp}
			ondblclick={handleFolderDoubleClick}
		>
			<img
				class={`w-19 rounded p-1 ${folderActive ? 'bg-black/10' : ''}`}
				src={folderIcon}
				alt="pink macos folder"
				draggable="false"
			/>
			<p class={`rounded px-1 ${folderActive ? 'bg-pink-arc' : ''}`}>pink arc [#ce79b1]</p>
		</div>
	</div>

	{#if isSelecting && selectionWidth > 0 && selectionHeight > 0}
		<div class="pointer-events-none absolute inset-0 z-30">
			<div
				class="absolute rounded-xs border border-blue-400/90 bg-blue-400/20 shadow-[inset_0_0_0_1px_rgba(255,255,255,0.35)]"
				style={`left:${selectionX}px;top:${selectionY}px;width:${selectionWidth}px;height:${selectionHeight}px;`}
			></div>
		</div>
	{/if}

	<div class="text-pink-arc text-4xl text-right">
		<p class="leading-13">apr 26th<br />@ midnight</p>
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
