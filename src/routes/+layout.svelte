<script lang="ts">
	import '@fontsource-variable/spline-sans-mono';
	import './app.css';
	import favicon from '$lib/assets/favicon.png';

	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
</svelte:head>

<main class="min-h-svh flex flex-col p-8">
	{@render children()}
</main>
