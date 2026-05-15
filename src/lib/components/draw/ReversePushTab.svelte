<script lang="ts">
	import Icon from '@iconify/svelte';
	import { Button } from '$lib/components/ui/button';
	import { Alert, AlertDescription } from '$lib/components/ui/alert';
	import { Badge } from '$lib/components/ui/badge';
	import { forumAuth } from '$lib/forum/stores/auth';
	import { drawEnv, apiError } from '$lib/draw/stores/env';
	import { connectRunWs } from '$lib/draw/api/ws';
	import { get } from 'svelte/store';
	import ProgressPanel from './ProgressPanel.svelte';
	import type { WsRunMessage } from '$lib/draw/types';

	const STORAGE_KEY = 'draw-reversepush';

	let {
		globalBusy = false,
		otherNode = $bindable(''),
		otherStage = $bindable(''),
		otherValue = $bindable(0),
		otherMax = $bindable(0),
	}: {
		globalBusy?: boolean;
		otherNode?: string;
		otherStage?: string;
		otherValue?: number;
		otherMax?: number;
	} = $props();

	let currentBaseUrl = $state('');
	let authToken = $state<string | null>(null);
	let isLoggedIn = $derived(!!authToken);
	let apiErrorMessage = $state('');

	let images = $state<{ file: File; dataUrl: string }[]>([]);
	let prompt = $state('');
	let denoise = $state(0.7);
	let uploading = $state(false);
	let error = $state('');

	// WebSocket progress state
	let progressMessages = $state<WsRunMessage[]>([]);
	let showProgress = $state(false);
	let isGenerating = $state(false);
	let resultImages = $state<{ url: string; filename: string }[]>([]);
	let genCost = $state(0);
	let runWs: WebSocket | null = null;

	$effect(() => {
		const unsub = apiError.subscribe((v) => {
			apiErrorMessage = v || '';
		});
		return unsub;
	});

	$effect(() => {
		const u1 = drawEnv.baseUrl.subscribe((v) => (currentBaseUrl = v));
		authToken = forumAuth.getToken();
		return u1;
	});

	// Restore state from localStorage
	$effect(() => {
		if (typeof localStorage === 'undefined') return;
		try {
			const saved = localStorage.getItem(STORAGE_KEY);
			if (!saved) return;
			const parsed = JSON.parse(saved);
			if (parsed.prompt) prompt = parsed.prompt;
			if (parsed.denoise != null) denoise = parsed.denoise;
			if (parsed.image) {
				fetch(parsed.image)
					.then(r => r.blob())
					.then(blob => {
						const ext = parsed.image.split(';')[0].split('/')[1] || 'png';
						const file = new File([blob], `reversepush.${ext}`, { type: blob.type });
						images = [{ file, dataUrl: parsed.image }];
					})
					.catch(() => {});
			}
		} catch {}
	});

	function saveState() {
		if (typeof localStorage === 'undefined') return;
		const data: any = { prompt, denoise };
		if (images.length > 0 && images[0].dataUrl.startsWith('data:')) {
			data.image = images[0].dataUrl;
		}
		localStorage.setItem(STORAGE_KEY, JSON.stringify(data));
	}

	function handleFileInput(e: Event) {
		const input = e.target as HTMLInputElement;
		if (!input.files?.length) return;
		const f = input.files[0];
		const url = URL.createObjectURL(f);
		images = [{ file: f, dataUrl: url }];
		const reader = new FileReader();
		reader.onload = () => {
			images = [{ ...images[0], dataUrl: reader.result as string }];
			URL.revokeObjectURL(url);
			saveState();
		};
		reader.readAsDataURL(f);
		input.value = '';
	}

	function removeImage() {
		if (images[0]) URL.revokeObjectURL(images[0].dataUrl);
		images = [];
		saveState();
	}

	function handlePromptInput(e: Event) {
		const el = e.target as HTMLTextAreaElement;
		prompt = el.value;
		saveState();
	}

	function handleDenoiseInput(e: Event) {
		const el = e.target as HTMLInputElement;
		denoise = parseFloat(el.value);
		saveState();
	}

	function handleRunMessage(msg: WsRunMessage) {
		progressMessages = [...progressMessages, msg];

		if (msg.type === 'image') {
			if (msg.image_type !== 'output') return;
			resultImages = [...resultImages, { url: msg.url, filename: msg.filename }];
		}
		if (msg.type === 'error') {
			isGenerating = false;
		}
		if (msg.type === 'cost') {
			genCost = msg.cost;
		}
		if (msg.type === 'done') {
			isGenerating = false;
		}
	}

	async function startGeneration() {
		if (isGenerating || uploading || images.length === 0) return;

		const token = forumAuth.getToken();
		if (!token) {
			error = '请先在论坛登录';
			return;
		}

		error = '';
		uploading = true;

		try {
			// 1. Upload image to ComfyUI
			const form = new FormData();
			form.append('image1', images[0].file);

			const baseUrl = get(drawEnv.baseUrl);
			const uploadResp = await fetch(`${baseUrl}/api/img2img/upload`, {
				method: 'POST',
				headers: { 'Authorization': `Bearer ${token}` },
				body: form,
			});

			if (!uploadResp.ok) {
				const body = await uploadResp.json().catch(() => ({ message: uploadResp.statusText }));
				throw new Error(body.message || body.detail || '上传失败');
			}

			const uploadData = await uploadResp.json();
			uploading = false;

			// 2. Connect WebSocket for generation
			isGenerating = true;
			progressMessages = [];
			resultImages = [];
			genCost = 0;
			showProgress = true;

			runWs = connectRunWs(
				baseUrl,
				{
					token,
					direct_prompt: prompt.trim(),
					image1_name: uploadData.image1_name,
					denoise,
					reverse_push: true,
				},
				handleRunMessage,
				undefined,
				() => { isGenerating = false; },
				() => { isGenerating = false; },
			);
		} catch (e) {
			uploading = false;
			error = e instanceof Error ? e.message : '生成失败';
		}
	}
</script>

<div class="space-y-4">
	{#if !isLoggedIn}
		<Alert>
			<Icon icon="mdi:account-alert-outline" class="size-4" />
			<AlertDescription class="text-xs">
				请先<a href="/forum/auth/login?redirect=/draw/" class="underline font-medium">登录论坛</a>后使用反推重绘功能。
			</AlertDescription>
		</Alert>
	{/if}

	{#if apiErrorMessage}
		<Alert>
			<Icon icon="mdi:cloud-alert" class="size-4 shrink-0" />
			<AlertDescription class="text-xs">{apiErrorMessage}</AlertDescription>
		</Alert>
	{/if}

	<!-- Image Upload -->
	<div class="space-y-2">
		<div class="flex items-center justify-between">
			<h3 class="text-sm font-medium flex items-center gap-1.5">
				<Icon icon="mdi:image-plus" class="size-4" />
				上传图片
				<Badge variant="secondary" class="text-xs">{images.length}/1</Badge>
			</h3>
		</div>

		<div class="flex gap-3 flex-wrap">
			{#each images as item, i}
				<div class="relative group">
					<img
						src={item.dataUrl}
						alt="preview"
						class="size-28 object-cover rounded-lg border border-border"
					/>
					<button
						class="absolute -top-2 -right-2 size-5 rounded-full bg-destructive text-destructive-foreground flex items-center justify-center opacity-0 group-hover:opacity-100 transition-opacity"
						onclick={removeImage}
						title="移除"
					>
						<Icon icon="mdi:close" class="size-3" />
					</button>
				</div>
			{/each}

			{#if images.length < 1}
				<label class="size-28 rounded-lg border-2 border-dashed border-muted-foreground/30 flex flex-col items-center justify-center gap-1 cursor-pointer hover:border-primary/50 transition-colors text-muted-foreground">
					<Icon icon="mdi:plus" class="size-6" />
					<span class="text-xs">选择图片</span>
					<input
						type="file"
						accept="image/png,image/jpeg,image/webp"
						class="hidden"
						onchange={handleFileInput}
					/>
				</label>
			{/if}
		</div>
	</div>

	<!-- Redraw Prompt -->
	<div class="space-y-2">
		<h3 class="text-sm font-medium flex items-center gap-1.5">
			<Icon icon="mdi:text-box-outline" class="size-4" />
			重绘提示词
		</h3>
		<textarea
			value={prompt}
			oninput={handlePromptInput}
			placeholder="输入你想要的修改描述，留空则仅使用 AI 自动生成的描述"
			class="w-full min-h-[80px] rounded-lg border border-input bg-background px-3 py-2 text-sm ring-offset-background placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring resize-y"
			disabled={isGenerating || uploading}
		></textarea>
	</div>

	<!-- Denoise Slider -->
	<div class="space-y-2">
		<div class="flex items-center justify-between">
			<h3 class="text-sm font-medium flex items-center gap-1.5">
				<Icon icon="mdi:water-percent" class="size-4" />
				重绘幅度 (Denoise)
			</h3>
			<span class="text-sm font-mono text-muted-foreground">{denoise.toFixed(2)}</span>
		</div>
		<input
			type="range"
			min="0"
			max="1"
			step="0.05"
			value={denoise}
			oninput={handleDenoiseInput}
			disabled={isGenerating || uploading}
			class="w-full accent-primary"
		/>
		<div class="flex justify-between text-xs text-muted-foreground">
			<span>0 (原图参考多)</span>
			<span>1 (全靠提示词)</span>
		</div>
	</div>

	<!-- Run Button -->
	<Button
		class="w-full gap-2"
		onclick={startGeneration}
		disabled={!isLoggedIn || isGenerating || uploading || globalBusy || images.length === 0}
	>
		{#if uploading}
			<Icon icon="mdi:loading" class="size-4 animate-spin" />
			上传中...
		{:else if isGenerating}
			<Icon icon="mdi:loading" class="size-4 animate-spin" />
			生成中...
		{:else if globalBusy}
			<Icon icon="mdi:loading" class="size-4 animate-spin" />
			他人生图中...
			{#if otherStage === 'llm'}
				<span class="text-[10px] opacity-70">(LLM处理)</span>
			{:else if otherMax > 0}
				<span class="text-[10px] opacity-70">({Math.round(otherValue / otherMax * 100)}%)</span>
			{/if}
		{:else}
			<Icon icon="mdi:play" class="size-4" />
			开始生成
		{/if}
	</Button>

	<!-- Error -->
	{#if error}
		<Alert variant="destructive">
			<Icon icon="mdi:alert-circle" class="size-4" />
			<AlertDescription class="text-xs">{error}</AlertDescription>
		</Alert>
	{/if}

	<!-- Progress & Results -->
	<ProgressPanel
		messages={progressMessages}
		visible={showProgress}
		busy={isGenerating}
		resultImages={resultImages}
		cost={genCost}
	/>
</div>
