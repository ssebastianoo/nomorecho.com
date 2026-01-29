<script lang="ts">
	import '@fontsource-variable/spline-sans-mono';
	import './app.css';

	let { children } = $props();
</script>

<main class="min-h-svh flex flex-col p-8">
	{@render children()}
</main>
