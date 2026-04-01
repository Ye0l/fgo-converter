<script>
	import './layout.css';
	import { base } from '$app/paths';
	let { children } = $props();
</script>

<svelte:head><link rel="icon" href="{base}/images/nunnos.png" /></svelte:head>
{@render children()}
