<script lang="ts">
	import './layout.css';
	import favicon from '$lib/assets/favicon.ico';

	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
	<link rel="preconnect" href="https://fonts.googleapis.com" />
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
	<link
		href="https://fonts.googleapis.com/css2?family=Instrument+Serif:ital@0;1&display=swap"
		rel="stylesheet"
	/>
	<link rel="canonical" href="https://scimo.net" />
</svelte:head>
{@render children()}

<style>
	:global(body) {
		font-family: 'Optima', serif;
		background: #0a0a0a;
		color: #fff;
		text-align: center;
		justify-content: center;
		font-size: 3rem;
	}
</style>
