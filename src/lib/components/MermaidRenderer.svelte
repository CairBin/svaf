<script lang="ts">
	import { onMount, tick } from 'svelte';
	import { renderMermaidIn, rerenderAllMermaid } from '$lib/utils/mermaid';
	import { isDark } from '$lib/stores/theme';

	let { selector = '.prose' }: { selector?: string } = $props();

	onMount(async () => {
		await tick();
		const container = document.querySelector(selector) as HTMLElement | null;
		renderMermaidIn(container);
	});

	$effect(() => {
		const dark = $isDark;
		console.log(`[mermaid] $effect fired, isDark=${dark}`);
		rerenderAllMermaid();
	});
</script>
