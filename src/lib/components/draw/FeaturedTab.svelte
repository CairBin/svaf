<script lang="ts">
	import Icon from '@iconify/svelte';
	import { Button } from '$lib/components/ui/button';
	import { Alert, AlertDescription } from '$lib/components/ui/alert';
	import { onMount, onDestroy } from 'svelte';
	import PhotoSwipeLightbox from 'photoswipe/lightbox';
	import 'photoswipe/style.css';
	import { fetchFeatured, getImageUrl, getImageProxyUrl } from '$lib/draw/api/client';
	import type { DrawOutputItem } from '$lib/draw/types';

	const tip = '精选图片由管理员挑选，展示社区优质作品。';

	let items = $state<DrawOutputItem[]>([]);
	let loading = $state(true);

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

	$effect(() => {
		loadFeatured();
	});

	onDestroy(() => lightbox?.destroy());

	async function loadFeatured() {
		loading = true;
		try {
			const res = await fetchFeatured();
			items = res.items;
		} catch {
			items = [];
		} finally {
			loading = false;
		}
		queueMicrotask(initLightbox);
	}
</script>

<div class="space-y-3">
	<div class="flex items-center justify-between">
		<h3 class="text-sm font-medium flex items-center gap-1.5">
			<Icon icon="mdi:star-outline" class="size-4" />
			精选
			<span class="text-xs text-muted-foreground">({items.length})</span>
		</h3>
		<Button variant="ghost" size="sm" onclick={loadFeatured} disabled={loading}>
			<Icon icon="mdi:refresh" class="size-4" />
		</Button>
	</div>

	{#if tip}
		<Alert>
			<AlertDescription class="text-xs">{tip}</AlertDescription>
		</Alert>
	{/if}

	{#if loading}
		<div class="text-xs text-muted-foreground py-8 text-center">加载中...</div>
	{:else if items.length === 0}
		<div class="text-xs text-muted-foreground py-8 text-center">暂无精选图片</div>
	{:else}
		<div bind:this={galleryEl} class="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 gap-2">
			{#each items as item}
				<a
					href={getImageUrl(item.path)}
					class="aspect-square rounded-lg overflow-hidden border hover:ring-2 hover:ring-primary/50 transition-all block"
				>
					<img
						src={getImageProxyUrl(item.path)}
						alt={item.path}
						class="w-full h-full object-cover"
						loading="lazy"
					/>
				</a>
			{/each}
		</div>
	{/if}
</div>
