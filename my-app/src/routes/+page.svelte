<script lang="ts">
	import { onMount } from 'svelte';

	let imageSrc = $state<string | null>(null);
	let imgElement = $state<HTMLImageElement | null>(null);
	let intrinsicSize = $state({ width: 0, height: 0 });
	let cursor = $state({ x: 0, y: 0 });
	let isFrozen = $state(false);
	let errorMessage = $state<string | null>(null);

	const allowedTypes = ['image/jpeg', 'image/png', 'image/gif', 'image/webp', 'image/bmp'];

	function handleFileUpload(event: Event) {
		const target = event.target as HTMLInputElement;
		const file = target.files?.[0];
		if (!file) return;

		if (!allowedTypes.includes(file.type)) {
			errorMessage = `Unsupported format: ${file.type}. Please upload JPG, PNG, GIF, WEBP, or BMP.`;
			imageSrc = null;
			return;
		}

		errorMessage = null;
		if (imageSrc) URL.revokeObjectURL(imageSrc);
		imageSrc = URL.createObjectURL(file);
		isFrozen = false;
	}

	function onImageLoad() {
		if (imgElement) {
			intrinsicSize = {
				width: imgElement.naturalWidth,
				height: imgElement.naturalHeight
			};
		}
	}

	function handleMouseMove(event: MouseEvent) {
		if (isFrozen || !imgElement || !intrinsicSize.width) return;

		const rect = imgElement.getBoundingClientRect();
		const rx = event.clientX - rect.left;
		const ry = event.clientY - rect.top;

		// Normalized to intrinsic scale
		const x_img = rx * (intrinsicSize.width / rect.width);
		const y_img_top = ry * (intrinsicSize.height / rect.height);

		// Bottom-left origin conversion
		// Y = Ih - y_img_top
		const X = Math.round(Math.max(0, Math.min(intrinsicSize.width, x_img)));
		const Y = Math.round(Math.max(0, Math.min(intrinsicSize.height, intrinsicSize.height - y_img_top)));

		cursor = { x: X, y: Y };
	}

	function handleImageClick() {
		if (imageSrc) {
			isFrozen = !isFrozen;
		}
	}

	function resetTracking() {
		isFrozen = false;
	}

	function clearImage() {
		if (imageSrc) URL.revokeObjectURL(imageSrc);
		imageSrc = null;
		intrinsicSize = { width: 0, height: 0 };
		cursor = { x: 0, y: 0 };
		isFrozen = false;
		errorMessage = null;
	}
</script>

<div class="flex min-h-screen flex-col bg-slate-950 text-slate-100 font-sans">
	<header class="border-b border-slate-800 bg-slate-900/50 p-4 backdrop-blur-sm">
		<div class="mx-auto flex max-w-6xl items-center justify-between">
			<h1 class="text-xl font-bold tracking-tight text-blue-400">Pixel Coordinates Viewer</h1>
			<div class="flex gap-4">
				{#if imageSrc}
					<button
						onclick={clearImage}
						class="rounded-md bg-red-500/10 px-3 py-1.5 text-sm font-medium text-red-400 border border-red-500/20 hover:bg-red-500/20 transition-colors"
					>
						Clear Image
					</button>
				{/if}
			</div>
		</div>
	</header>

	<main class="relative flex flex-1 flex-col items-center justify-center p-4">
		{#if !imageSrc}
			<label
				class="group flex cursor-pointer flex-col items-center justify-center rounded-2xl border-2 border-dashed border-slate-700 bg-slate-900/30 p-12 transition-all hover:border-blue-500/50 hover:bg-slate-900/50"
			>
				<div class="mb-4 rounded-full bg-slate-800 p-4 text-slate-400 group-hover:text-blue-400 transition-colors">
					<svg xmlns="http://www.w3.org/2000/svg" width="32" height="32" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/><polyline points="17 8 12 3 7 8"/><line x1="12" x2="12" y1="3" y2="15"/></svg>
				</div>
				<span class="text-lg font-medium">Upload an image to start</span>
				<p class="mt-2 text-sm text-slate-500">JPG, PNG, GIF, WEBP, BMP</p>
				<input type="file" accept="image/*" onchange={handleFileUpload} class="hidden" />
			</label>
			{#if errorMessage}
				<p class="mt-4 text-red-400 text-sm font-medium bg-red-400/10 px-4 py-2 rounded-lg border border-red-400/20">{errorMessage}</p>
			{/if}
		{:else}
			<div class="relative flex max-h-[80vh] max-w-full flex-col items-center gap-6">
				<div 
					class="relative overflow-hidden rounded-xl border border-slate-800 bg-slate-900 shadow-2xl ring-1 ring-white/5"
					role="button"
					tabindex="0"
					onmousemove={handleMouseMove}
					onclick={handleImageClick}
					onkeydown={(e) => e.key === 'Enter' && handleImageClick()}
				>
					<!-- svelte-ignore a11y_missing_attribute -->
					<img
						bind:this={imgElement}
						src={imageSrc}
						onload={onImageLoad}
						class="max-h-[70vh] max-w-full block select-none {isFrozen ? 'cursor-default' : 'cursor-crosshair'}"
					/>
					
					{#if isFrozen}
						<div class="pointer-events-none absolute inset-0 bg-blue-500/5 ring-4 ring-inset ring-blue-500/30"></div>
					{/if}
				</div>

				<!-- Coordinate Window -->
				<div class="flex items-center gap-4 rounded-full bg-slate-900/80 border border-slate-700/50 px-6 py-3 shadow-xl backdrop-blur-md">
					<div class="flex flex-col">
						<span class="text-[10px] uppercase tracking-widest text-slate-500 font-bold">Coordinates</span>
						<p class="text-lg font-mono font-medium text-blue-100 italic transition-all">
							<span class="text-blue-400">{cursor.x}</span> pixels right, <span class="text-blue-400">{cursor.y}</span> pixels up
						</p>
					</div>
					
					<div class="h-8 w-px bg-slate-700 mx-2"></div>
					
					<div class="flex items-center gap-2">
						<div class="flex h-6 items-center gap-1.5 rounded-full px-2.5 py-1 {isFrozen ? 'bg-amber-400/10 text-amber-400 ring-1 ring-inset ring-amber-400/20' : 'bg-green-400/10 text-green-400 ring-1 ring-inset ring-green-400/20'}">
							<div class="h-1.5 w-1.5 rounded-full {isFrozen ? 'bg-amber-400 animate-pulse' : 'bg-green-400'}"></div>
							<span class="text-xs font-bold uppercase tracking-tight">{isFrozen ? 'Frozen' : 'Live'}</span>
						</div>
						
						{#if isFrozen}
							<button 
								onclick={resetTracking}
								class="ml-2 rounded-lg bg-blue-500 px-3 py-1.5 text-xs font-bold text-white hover:bg-blue-400 transition-colors shadow-lg shadow-blue-500/20"
							>
								Unfreeze
							</button>
						{/if}
					</div>
				</div>
				
				<p class="text-xs text-slate-500 font-medium">Click on the image to {isFrozen ? 'resume' : 'freeze'} tracking</p>
			</div>
		{/if}
	</main>

	<footer class="p-4 text-center text-xs text-slate-600 border-t border-slate-900 bg-slate-950">
		<p>Intrinsic size: {intrinsicSize.width} × {intrinsicSize.height} px</p>
	</footer>
</div>

<style>
	:global(body) {
		margin: 0;
		background-color: #020617;
	}
</style>
