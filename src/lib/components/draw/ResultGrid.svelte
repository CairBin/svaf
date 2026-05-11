<script lang="ts">
	import Icon from '@iconify/svelte';
	import { onMount } from 'svelte';
	import PhotoSwipeLightbox from 'photoswipe/lightbox';
	import 'photoswipe/style.css';
	import { getImageProxyUrl, getImageUrl } from '$lib/draw/api/client';

	let {
		images = []
	}: {
		images?: { url: string; filename: string }[];
	} = $props();

	let galleryEl: HTMLElement;
	let lightbox: PhotoSwipeLightbox | null = null;

	function initLightbox() {
		lightbox?.destroy();
		if (!galleryEl) return;
		lightbox = new PhotoSwipeLightbox({
			gallery: galleryEl,
			children: 'a',
			pswpModule: () => import('photoswipe')
		});
		lightbox.addFilter('itemData', (itemData) => {
			const a = itemData.element as HTMLAnchorElement;
			const img = a?.querySelector('img') as HTMLImageElement | null;
			return {
				src: a.href,
				width: img?.naturalWidth || 1600,
				height: img?.naturalHeight || 1200,
				alt: img?.alt || ''
			};
		});
		lightbox.init();
	}

	onMount(() => {
		initLightbox();
		return () => lightbox?.destroy();
	});

	$effect(() => {
		if (images.length > 0) {
			// Svelte DOM 更新后重新初始化
			queueMicrotask(initLightbox);
		}
	});
</script>

{#if images.length > 0}
	<div class="space-y-2">
		<h3 class="text-sm font-medium flex items-center gap-1.5">
			<Icon icon="mdi:image-multiple-outline" class="size-4" />
			生成结果
			<span class="text-xs text-muted-foreground">({images.length})</span>
		</h3>
		<div bind:this={galleryEl} class="grid grid-cols-2 md:grid-cols-3 gap-2">
			{#each images as img}
				<a
					href={getImageUrl(img.filename)}
					class="aspect-square rounded-lg overflow-hidden border hover:ring-2 hover:ring-primary/50 transition-all block"
				>
					<img
						src={getImageProxyUrl(img.filename)}
						alt={img.filename}
						class="w-full h-full object-cover"
						loading="lazy"
					/>
				</a>
			{/each}
		</div>
	</div>
{/if}
