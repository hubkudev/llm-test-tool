<script lang="ts">
	import '../app.css';
	let { children } = $props();
</script>

<svelte:head>
	<title>API Response Test Tool</title>
	<meta name="description" content="Tools created by @resqiar to help measure consistency of an API call" />
</svelte:head>

{@render children()}
