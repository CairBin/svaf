<script lang="ts">
	import Icon from '@iconify/svelte';
	import { Button } from '$lib/components/ui/button';
	import { onMount, onDestroy } from 'svelte';
	import PhotoSwipeLightbox from 'photoswipe/lightbox';
	import 'photoswipe/style.css';
	import { fetchOutputList, getImageUrl, getImageProxyUrl } from '$lib/draw/api/client';
	import type { DrawOutputItem } from '$lib/draw/types';

	let items = $state<DrawOutputItem[]>([]);
	let total = $state(0);
	let loading = $state(false);
	let offset = $state(0);
	const limit = 30;
	let hasMore = $derived(offset < total);

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
		loadGallery();
	});

	onMount(() => {
		queueMicrotask(initLightbox);
	});

	onDestroy(() => lightbox?.destroy());

	async function loadGallery() {
		loading = true;
		try {
			const res = await fetchOutputList(limit, 0);
			items = res.items;
			total = res.total;
			offset = res.items.length;
		} catch {
			items = [];
		} finally {
			loading = false;
		}
	}

	async function loadMore() {
		if (loading || !hasMore) return;
		loading = true;
		try {
			const res = await fetchOutputList(limit, offset);
			items = [...items, ...res.items];
			offset += res.items.length;
			queueMicrotask(initLightbox);
		} catch {
			// ignore
		} finally {
			loading = false;
		}
	}
</script>

<div class="space-y-3">
	<div class="flex items-center justify-between">
		<h3 class="text-sm font-medium flex items-center gap-1.5">
			<Icon icon="mdi:image-multiple-outline" class="size-4" />
			画廊
			<span class="text-xs text-muted-foreground">({items.length}/{total})</span>
		</h3>
		<Button variant="ghost" size="sm" onclick={loadGallery} disabled={loading}>
			<Icon icon="mdi:refresh" class="size-4" />
		</Button>
	</div>

	{#if items.length === 0 && !loading}
		<div class="text-xs text-muted-foreground py-8 text-center">暂无图片</div>
	{:else}
		<div bind:this={galleryEl} class="grid grid-cols-3 sm:grid-cols-4 md:grid-cols-5 gap-1.5">
			{#each items as item}
				<a
					href={getImageUrl(item.path)}
					class="aspect-square rounded-md overflow-hidden border hover:ring-2 hover:ring-primary/50 transition-all block"
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

		{#if hasMore}
			<div class="text-center">
				<Button variant="outline" size="sm" onclick={loadMore} disabled={loading}>
					{#if loading}
						<Icon icon="mdi:loading" class="size-4 animate-spin mr-1" />
					{/if}
					加载更多
				</Button>
			</div>
		{/if}
	{/if}
</div>
