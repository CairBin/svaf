<script lang="ts">
	import Icon from '@iconify/svelte';
	import { siteConfig } from '$lib/config/site';
	import { Tabs, TabsContent, TabsList, TabsTrigger } from '$lib/components/ui/tabs';
	import { Badge } from '$lib/components/ui/badge';
	import { Alert, AlertDescription } from '$lib/components/ui/alert';
	import { Button } from '$lib/components/ui/button';
	import { forumAuth } from '$lib/forum/stores/auth';
	import { drawEnv, apiError, apiStatus, resolveApiRedirect } from '$lib/draw/stores/env';
	import { connectStatusWs } from '$lib/draw/api/ws';
	import { fetchMyImages, getImageUrl, getImageProxyUrl, forkOutputImage, recommendImage, deleteMyImage, fetchMyRecommendations, addToQueue, fetchMyQueue } from '$lib/draw/api/client';
	import { consumeFork } from '$lib/draw/stores/fork';
	import { onMount, onDestroy } from 'svelte';
	import type { WsStatusEvent, DrawWorkflow, DrawRecommendation } from '$lib/draw/types';

	import PageViews from '$lib/components/PageViews.svelte';

	import EnvironmentSwitcher from '$lib/components/draw/EnvironmentSwitcher.svelte';
	import PromptForm from '$lib/components/draw/PromptForm.svelte';
	
	import FeaturedTab from '$lib/components/draw/FeaturedTab.svelte';
	import Img2imgTab from '$lib/components/draw/Img2imgTab.svelte';
		import ImageLightbox from '$lib/components/draw/ImageLightbox.svelte';

	// State
	let currentBaseUrl = $state('');
	let onlineCount = $state(0);
	let globalBusy = $state(false);
	
	let authToken = $state<string | null>(null);
	let isLoggedIn = $derived(!!authToken);

	// Form state
	let workflowPath = $state('');
	let workflowName = $state('');
	let styleTags = $state('');
	let styleName = $state('');
	let directPrompt = $state('');
	let negativePrompt = $state('');
	let nlPrompt = $state('');
	let rewrite = $state(true);
	let width = $state(0);
	let height = $state(0);
	let inlineWorkflow = $state<object | null>(null);
	let safetyRating = $state('general');
	let forkSeed = $state<number | undefined>(undefined);
	let sameSeed = $state(false);

	// Restore form state from localStorage
	if (typeof localStorage !== 'undefined') {
		try {
			const saved = localStorage.getItem('draw-form');
			if (saved) {
				const p = JSON.parse(saved);
				if (p.workflowPath) workflowPath = p.workflowPath;
				if (p.workflowName) workflowName = p.workflowName;
				if (p.styleTags) styleTags = p.styleTags;
				if (p.styleName) styleName = p.styleName;
				if (p.directPrompt) directPrompt = p.directPrompt;
				if (p.negativePrompt) negativePrompt = p.negativePrompt;
				if (p.nlPrompt) nlPrompt = p.nlPrompt;
				if (p.rewrite !== undefined) rewrite = p.rewrite;
				if (p.width) width = p.width;
				if (p.height) height = p.height;
				if (p.safetyRating) safetyRating = p.safetyRating;
			}
		} catch {}
	}

	// Progress state

	// Queue dialog state
	let pendingPayload: WsRunPayload | null = null;
	let queueError = $state('');

	// My images state
	let myImages = $state<{ path: string; mtime: number }[]>([]);
	let myImagesTotal = $state(0);
	let myImagesLoading = $state(false);
	let myImagesLoaded = $state(false);
	let myQueueItems = $state<Array<{ id: number; status: string; created_at: number; started_at?: number; finished_at?: number; error?: string; position?: number | null }>>([]);
	let myQueueLoading = $state(false);
	// Masonry layout
	let columnCount = $state(4);
	let imgColumns = $state<string[][]>([[], [], [], []]);
	let columnHeights: number[] = [0, 0, 0, 0];
	let sentinelEl: HTMLDivElement | undefined;
	let io: IntersectionObserver | null = null;
	let hasMore = $state(true);
	let loadingMore = $state(false);

	// My images lightbox
	let myLbOpen = $state(false);
	let myLbIndex = $state(0);
	let myLbImages = $derived(myImages.map((it) => ({ src: getImageUrl(it.path), cached: getImageProxyUrl(it.path), creator_id: '' })));

	// Recommendations
	let myRecommendations = $state<DrawRecommendation[]>([]);
	let myRecsLoaded = $state(false);
	let selectMode = $state(false);
	let selectedPaths = $state(new Set<string>());

	// WebSocket refs
	let statusConn: ReturnType<typeof connectStatusWs> | null = null;
	

	// API error state
	let apiErrorMessage = $state("");
	let apiStatusValue = $state("checking");

	$effect(() => {
		const unsub = apiError.subscribe((v) => {
			apiErrorMessage = v || '';
		});
		return unsub;
	});

	$effect(() => {
		const unsub = apiStatus.subscribe((v) => (apiStatusValue = v));
		return unsub;
	});

	// 页面加载时探测 API 状态
	$effect(() => {
		resolveApiRedirect();
	});

	// Tab state
	let activeTab = $state('generate');
	let genSubTab = $state('txt2img');

	// Persist form state to localStorage
	$effect(() => {
		if (typeof localStorage === 'undefined') return;
		const state = { workflowPath, workflowName, styleTags, styleName, directPrompt, negativePrompt, nlPrompt, rewrite, width, height, safetyRating };
		localStorage.setItem('draw-form', JSON.stringify(state));
	});

	$effect(() => {
		if (activeTab === 'mine' && isLoggedIn) {
			if (!myImagesLoaded) loadMyImages();
			loadMyQueue();
			if (!myRecsLoaded) loadMyRecommendations();
		}
	});

	$effect(() => {
		const u1 = drawEnv.baseUrl.subscribe((v) => (currentBaseUrl = v));
		authToken = forumAuth.getToken();

		// 检查是否有待消费的 fork 数据（从其他页面跳转过来）
		const fork = consumeFork();
		if (fork) {
			inlineWorkflow = fork.workflow;
			if (fork.builtin_prompt) directPrompt = fork.builtin_prompt;
			if (fork.builtin_negative_prompt) negativePrompt = fork.builtin_negative_prompt;
			if (fork.default_width) width = fork.default_width;
			if (fork.default_height) height = fork.default_height;
			forkSeed = fork.seed;
			workflowPath = 'fork';
			workflowName = '(fork)';
			if (fork.style_tags) {
				styleTags = fork.style_tags;
				styleName = fork.style_tags;
			} else {
				styleTags = '';
				styleName = '';
			}
			activeTab = 'generate';
		}

		// Connect status WebSocket
		statusConn = connectStatusWs(currentBaseUrl, handleStatusMessage, undefined, () => { globalBusy = false; });

		return () => {
			u1();
			statusConn?.close();
		};
	});

	// Reconnect status WS when base URL changes
	$effect(() => {
		const url = currentBaseUrl;
		if (!url) return;
		statusConn?.close();
		statusConn = connectStatusWs(url, handleStatusMessage, undefined, () => { globalBusy = false; });
	});

	function handleStatusMessage(msg: WsStatusEvent) {
		switch (msg.type) {
			case 'status':
				onlineCount = msg.online;
				globalBusy = msg.busy;
				break;
			case 'online':
				onlineCount = msg.count;
				break;
		}
	}

	function handleWorkflowSelect(wf: DrawWorkflow) {
		workflowPath = wf.path;
		workflowName = wf.path.split('/').pop()?.replace('.json', '') || '';
		inlineWorkflow = null;
		forkSeed = undefined;
		sameSeed = false;
	}

	function handleStyleSelect(tags: string, name: string) {
		styleTags = tags;
		styleName = name;
	}

	function handlePromptLoad(positive: string, negative: string) {
		directPrompt = positive;
		negativePrompt = negative;
	}

	async function handleFork(path: string) {
		try {
			const res = await forkOutputImage(path);
			inlineWorkflow = res.workflow;
			if (res.builtin_prompt) directPrompt = res.builtin_prompt;
			if (res.builtin_negative_prompt) negativePrompt = res.builtin_negative_prompt;
			if (res.default_width) width = res.default_width;
			if (res.default_height) height = res.default_height;
			forkSeed = res.seed;
			workflowPath = 'fork';
			workflowName = '(fork)';
			if (res.style_tags) {
				styleTags = res.style_tags;
				styleName = res.style_tags;
			} else {
				styleTags = '';
				styleName = '';
			}
			activeTab = 'generate';
		} catch (e) {
			alert(e instanceof Error ? e.message : 'Fork 失败');
		}
	}

	function startGeneration() {
		if (queuing) return;
		if (!authToken) {
			alert('请先在论坛登录');
			return;
		}
		if (!workflowPath && !inlineWorkflow) {
			alert('请选择工作流');
			return;
		}

		const ratingTag = `rating:${safetyRating}`;
		const finalDirectPrompt = directPrompt
			? `${directPrompt}, ${ratingTag}`
			: ratingTag;

		queuing = true;
		queueSuccess = '';
		try {
			await addToQueue({
				direct_prompt: finalDirectPrompt,
				nl_prompt: nlPrompt || undefined,
				rewrite,
				width: width || undefined,
				height: height || undefined,
				style_tags: styleTags || undefined,
				negative_prompt: negativePrompt || undefined,
				seed: sameSeed ? forkSeed : undefined,
				workflow_path: workflowPath,
				inline_workflow: inlineWorkflow || undefined,
			});
			queueSuccess = '已加入队列，生成完成后可前往"我的"页面查看';
		} catch (e) {
			alert(e instanceof Error ? e.message : '加入队列失败');
		} finally {
			queuing = false;
		}
	}

