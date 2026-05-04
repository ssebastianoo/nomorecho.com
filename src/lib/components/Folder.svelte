<script lang="ts">
	import { browser } from "$app/environment";
	import folderIcon from "$lib/assets/folder.png";

	let {
		text,
		onclick,
		containerEl = $bindable(),
		centerOffsetX,
		centerOffsetY
	}: {
		text: string;
		onclick: () => void;
		containerEl: HTMLDivElement;
		centerOffsetX?: number;
		centerOffsetY?: number;
	} = $props();

	let folderEl = $state<HTMLDivElement>();
	let folderX = $state(0);
	let folderY = $state(0);
	let dragOffsetX = $state(0);
	let dragOffsetY = $state(0);
	let isDragging = $state(false);
	let folderActive = $state(false);
	let hasInitializedPosition = $state(false);

	function clamp(value: number, min: number, max: number) {
		return Math.min(Math.max(value, min), max);
	}

	function getBounds() {
		if (!containerEl || !folderEl) return null;

		const containerRect = containerEl.getBoundingClientRect();
		const folderRect = folderEl.getBoundingClientRect();
		const maxX = containerRect.width - folderRect.width;
		const maxY = containerRect.height - folderRect.height;
		return { containerRect, folderRect, maxX, maxY };
	}

	function moveTo(clientX: number, clientY: number) {
		const bounds = getBounds();
		if (!bounds) return;

		folderX = clamp(clientX - bounds.containerRect.left - dragOffsetX, 0, bounds.maxX);
		folderY = clamp(clientY - bounds.containerRect.top - dragOffsetY, 0, bounds.maxY);
	}

	function initializePosition() {
		const bounds = getBounds();
		if (!bounds) return;

		const centeredX = (bounds.containerRect.width - bounds.folderRect.width) / 2;
		const centeredY = (bounds.containerRect.height - bounds.folderRect.height) / 2;
		folderX = clamp(centeredX + (centerOffsetX ?? 0), 0, bounds.maxX);
		folderY = clamp(centeredY + (centerOffsetY ?? 0), 0, bounds.maxY);

		hasInitializedPosition = true;
	}

	function clampToContainer() {
		const bounds = getBounds();
		if (!bounds) return;

		folderX = clamp(folderX, 0, bounds.maxX);
		folderY = clamp(folderY, 0, bounds.maxY);
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

	function handleWindowPointerDown(event: PointerEvent) {
		if (!folderEl) return;
		if (event.target instanceof Node && !folderEl.contains(event.target)) {
			folderActive = false;
		}
	}

	$effect(() => {
		if (!browser || !containerEl || !folderEl) return;

		if (!hasInitializedPosition) {
			requestAnimationFrame(initializePosition);
		}

		const handleResize = () => clampToContainer();
		window.addEventListener("resize", handleResize);
		window.addEventListener("pointerdown", handleWindowPointerDown, true);
		return () => {
			window.removeEventListener("resize", handleResize);
			window.removeEventListener("pointerdown", handleWindowPointerDown, true);
		};
	});
</script>

<div class="pointer-events-none absolute inset-0 z-10">
	<div
		bind:this={folderEl}
		data-no-selection
		class="group pointer-events-auto absolute flex cursor-grab flex-col items-center gap-2 select-none active:cursor-grabbing"
		style={`left: ${folderX}px; top: ${folderY}px; touch-action: none;`}
		role="button"
		tabindex="0"
		aria-label={`Draggable folder ${text}`}
		onpointerdown={handlePointerDown}
		onpointermove={handlePointerMove}
		onpointerup={handlePointerUp}
		onpointercancel={handlePointerUp}
		ondblclick={onclick}
	>
		<img
			class={`w-19 rounded p-1 ${folderActive ? 'bg-black/10' : ''}`}
			src={folderIcon}
			alt="pink macos folder"
			draggable="false"
		/>
		<p class={`rounded px-1 ${folderActive ? 'bg-pink-arc' : ''}`}>{text}</p>
	</div>
</div>
