<script>
	import { fade } from 'svelte/transition';
	import badgeSvg from '../../../static/team-badge.svg?raw';
	import jomaSvg from '../../../static/joma.svg?raw';
	import { Check, Download, ImageUp, LoaderCircle, X } from '@lucide/svelte';
	import { Button, buttonVariants } from '#lib/components/ui/button/index.js';
	import { Popover, PopoverContent, PopoverTrigger } from '#lib/components/ui/popover/index.js';
	import { Slider } from '#lib/components/ui/slider/index.js';
	import { ToggleGroup, ToggleGroupItem } from '#lib/components/ui/toggle-group/index.js';

	/** @typedef {'collar' | 'cuffs' | 'panels'} ZoneId */
	/** @typedef {{ label: string; color: string | null; texture: string | null; scale: number }} Zone */

	const BASES = { white: '#f2f2ef', maroon: '#6b1d2c' };
	const SWATCHES = ['#6b1d2c', '#f2f2ef', '#111111', '#1d2a44', '#c9a14a', '#e8dcc4', '#1f4d3a', '#8fb8de', '#b8412e'];

	const TORSO = 'M165,30 Q200,58 235,30 L305,50 L292,160 L290,425 Q200,433 110,425 L108,160 L95,50 Z';
	const SLEEVE = 'M95,50 L30,150 L78,185 L108,160 Z';
	const CUFF = 'M30,150 L78,185 L86.4,178 L36,140.8 Z';
	const PANEL = 'M106,155 L126,162 L128,440 L100,440 Z';
	const NECK_FRONT = 'M165,30 Q200,58 235,30';
	const NECK_BACK = 'M165,30 Q200,18 235,30';
	const MIRROR = 'translate(400,0) scale(-1,1)';
	// inlined so the PNG export (SVG drawn via <img>) can render it
	const BADGE = `data:image/svg+xml,${encodeURIComponent(badgeSvg)}`;
	// inline the Joma mark's path so its colour can follow the base
	const JOMA = {
		d: jomaSvg.match(/ d="([^"]+)"/)?.[1],
		transform: jomaSvg.match(/<g[^>]* transform="([^"]+)"/)?.[1]
	};

	let base = $state('white');
	/** @type {Record<ZoneId, Zone>} */
	let zones = $state({
		collar: { label: 'Collar', color: null, texture: null, scale: 60 },
		cuffs: { label: 'Cuffs', color: null, texture: null, scale: 60 },
		panels: { label: 'Side panels', color: null, texture: null, scale: 120 }
	});
	let sponsor = $state(/** @type {string | null} */ (null));
	let sponsorSize = $state(140);
	let hover = $state(/** @type {ZoneId | 'sponsor' | null} */ (null));
	let dragging = $state(false);
	let exportState = $state(/** @type {'idle' | 'busy' | 'done'} */ ('idle'));
	/** @type {SVGSVGElement} */
	let svg;

	/** @param {ZoneId} id */
	const fill = (id) => (zones[id].texture ? `url(#tex-${id})` : colorOf(zones[id]));
	// untouched zones (color null) follow the base, so you start from the plain jersey
	/** @param {Zone} z */
	const colorOf = (z) => z.color ?? BASES[base];
	/** @param {Zone} z */
	const swatchBg = (z) => (z.texture ? `center / cover url(${z.texture})` : colorOf(z));
	/** @param {string} hex */
	const isLight = (hex) => {
		const n = parseInt(hex.slice(1), 16);
		return ((n >> 16) * 299 + ((n >> 8) & 255) * 587 + (n & 255) * 114) / 1000 > 150;
	};

	/** @param {File} file @returns {Promise<string>} */
	const readFile = (file) =>
		new Promise((resolve) => {
			const reader = new FileReader();
			reader.onload = () => resolve(/** @type {string} */ (reader.result));
			reader.readAsDataURL(file);
		});

	/** @param {Event} e @param {(url: string) => void} set */
	async function pick(e, set) {
		const input = /** @type {HTMLInputElement} */ (e.currentTarget);
		const file = input.files?.[0];
		input.value = ''; // allow re-picking the same file
		if (file) set(await readFile(file));
	}

	/** @param {DragEvent} e */
	async function drop(e) {
		e.preventDefault();
		dragging = false;
		const file = [...(e.dataTransfer?.files ?? [])].find((f) => f.type.startsWith('image/'));
		if (file) sponsor = await readFile(file);
	}

	async function exportPng() {
		exportState = 'busy';
		try {
			await document.fonts.load("36px 'Bebas Neue'");
			const img = new Image();
			const url = URL.createObjectURL(new Blob([new XMLSerializer().serializeToString(svg)], { type: 'image/svg+xml' }));
			img.src = url;
			await img.decode();
			URL.revokeObjectURL(url);

			// 4:5 portrait, Instagram's feed size
			const canvas = document.createElement('canvas');
			canvas.width = 1080;
			canvas.height = 1350;
			const ctx = /** @type {CanvasRenderingContext2D} */ (canvas.getContext('2d'));
			const glow = ctx.createRadialGradient(540, 560, 0, 540, 560, 820);
			glow.addColorStop(0, '#343434');
			glow.addColorStop(0.5, '#141414');
			glow.addColorStop(1, '#050505');
			ctx.fillStyle = glow;
			ctx.fillRect(0, 0, 1080, 1350);
			ctx.drawImage(img, 90, 70, 900, 1012.5);
			ctx.font = "34px 'Bebas Neue'";
			ctx.letterSpacing = '7px';
			ctx.textAlign = 'center';
			ctx.fillStyle = 'rgb(255 255 255 / 0.6)';
			ctx.fillText('SPORTING CLUB AIKEN', 540, 1260);

			/** @type {Blob} */
			const blob = await new Promise((resolve) => canvas.toBlob(resolve, 'image/png'));
			const file = new File([blob], 'sc-aiken-kit.png', { type: 'image/png' });
			// phones get the share sheet (straight to Instagram); desktops download
			if (navigator.canShare?.({ files: [file] }) && matchMedia('(pointer: coarse)').matches) {
				await navigator.share({ files: [file] }).catch(() => {});
			} else {
				const a = document.createElement('a');
				a.href = URL.createObjectURL(file);
				a.download = file.name;
				a.click();
				URL.revokeObjectURL(a.href);
			}
			exportState = 'done';
			setTimeout(() => (exportState = 'idle'), 1600);
		} catch {
			exportState = 'idle';
		}
	}
</script>

<svelte:head>
	<title>Kit Lab · Sporting Club Aiken</title>
</svelte:head>

<header class="label flex items-baseline gap-3 px-6 pt-6">
	<a href="/" class="transition-colors hover:text-white">Sporting Club Aiken</a>
	<span class="text-white/25">/</span>
	<span>Kit Lab</span>
</header>

<main class="mx-auto grid max-w-6xl items-center gap-6 px-4 py-6 sm:px-6 lg:grid-cols-[minmax(0,1fr)_20rem] lg:gap-10">
	<section
		aria-label="Kit preview"
		class="stage relative overflow-hidden rounded-3xl border border-white/[0.07] transition-[border-color,box-shadow] duration-300"
		class:dragging
		ondragover={(e) => {
			e.preventDefault();
			dragging = true;
		}}
		ondragleave={() => (dragging = false)}
		ondrop={drop}
	>
		<svg bind:this={svg} viewBox="0 0 400 450" class="relative mx-auto block max-h-[78dvh] w-full p-[6%]">
			<defs>
				<clipPath id="torso"><path d={TORSO} /></clipPath>
				<linearGradient id="shade">
					<stop offset="0" stop-opacity="0.2" />
					<stop offset="0.3" stop-opacity="0" />
					<stop offset="0.7" stop-opacity="0" />
					<stop offset="1" stop-opacity="0.2" />
				</linearGradient>
				<filter id="lift" x="-20%" y="-20%" width="140%" height="140%">
					<feDropShadow dy="10" stdDeviation="12" flood-opacity="0.55" />
				</filter>
				{#each Object.entries(zones) as [id, z] (id)}
					{#if z.texture}
						<pattern id="tex-{id}" patternUnits="userSpaceOnUse" width={z.scale} height={z.scale}>
							<image href={z.texture} width={z.scale} height={z.scale} preserveAspectRatio="xMidYMid slice" />
						</pattern>
					{/if}
				{/each}
			</defs>

			<g filter="url(#lift)">
				<!-- inside of the neck, seen through the opening -->
				<path class="paint" d="{NECK_FRONT} Q200,18 165,30Z" fill={BASES[base]} />
				<path d="{NECK_FRONT} Q200,18 165,30Z" opacity="0.35" />
				<path class="paint" d={NECK_BACK} fill="none" stroke={fill('collar')} stroke-width="7" />

				{#each ['', MIRROR] as transform (transform)}
					<g {transform}>
						<path class="paint" d={SLEEVE} fill={BASES[base]} />
						<path class="paint" d={CUFF} fill={fill('cuffs')} />
						<path d={SLEEVE} fill="url(#shade)" />
					</g>
				{/each}

				<path class="paint" d={TORSO} fill={BASES[base]} />
				<g clip-path="url(#torso)">
					<path class="paint" d={PANEL} fill={fill('panels')} />
					<path class="paint" d={PANEL} fill={fill('panels')} transform={MIRROR} />
				</g>
				<path class="paint" d={NECK_FRONT} fill="none" stroke={fill('collar')} stroke-width="9" />

				{#if sponsor}
					<image
						transition:fade={{ duration: 250 }}
						href={sponsor}
						x={200 - sponsorSize / 2}
						y={200 - sponsorSize * 0.3}
						width={sponsorSize}
						height={sponsorSize * 0.6}
						preserveAspectRatio="xMidYMid meet"
					/>
				{/if}

				<!-- crest on the wearer's left chest -->
				<image href={BADGE} x="229" y="80" width="34" height="51" preserveAspectRatio="xMidYMid meet" />

				<!-- Joma mark on the wearer's right chest, mirroring the crest (artwork spans ~92–908 × 274–726) -->
				<g transform="translate(141 98) scale(0.032) translate(-92 -274)">
					<path class="paint" d={JOMA.d} transform={JOMA.transform} fill={base === 'maroon' ? '#fff' : '#111'} />
				</g>

				<path d={TORSO} fill="url(#shade)" />
			</g>

			<!-- rim light so dark kits don't melt into the background -->
			<g fill="none" stroke="#fff" stroke-opacity="0.12" stroke-width="0.75">
				<path d={TORSO} />
				<path d={SLEEVE} />
				<path d={SLEEVE} transform={MIRROR} />
			</g>

			<!-- highlight the zone whose control is hovered -->
			<g class="pointer-events-none" fill="#fff" stroke="#fff">
				{#if hover === 'collar'}
					<path class="hl" transition:fade={{ duration: 150 }} d={NECK_FRONT} fill="none" stroke-width="10" />
				{:else if hover === 'cuffs'}
					<g class="hl" transition:fade={{ duration: 150 }} stroke="none">
						<path d={CUFF} />
						<path d={CUFF} transform={MIRROR} />
					</g>
				{:else if hover === 'panels'}
					<g class="hl" transition:fade={{ duration: 150 }} clip-path="url(#torso)" stroke="none">
						<path d={PANEL} />
						<path d={PANEL} transform={MIRROR} />
					</g>
				{:else if hover === 'sponsor'}
					<rect
						transition:fade={{ duration: 150 }}
						x={200 - sponsorSize / 2}
						y={200 - sponsorSize * 0.3}
						width={sponsorSize}
						height={sponsorSize * 0.6}
						rx="4"
						fill="none"
						stroke-opacity="0.5"
						stroke-dasharray="4 4"
					/>
				{/if}
			</g>
		</svg>

		{#if dragging}
			<div
				transition:fade={{ duration: 150 }}
				class="pointer-events-none absolute inset-0 label grid place-items-center bg-black/40 backdrop-blur-[2px]"
			>
				Drop sponsor logo
			</div>
		{/if}
	</section>

	<aside class="rounded-3xl border border-white/[0.07] bg-white/[0.02] p-5 backdrop-blur-xl">
		<div class="flex items-center justify-between pb-4">
			<span class="label">Base</span>
			<ToggleGroup type="single" variant="outline" size="sm" value={base} onValueChange={(v) => v && (base = v)}>
				{#each Object.entries(BASES) as [name, color] (name)}
					<ToggleGroupItem value={name} class="gap-1.5 px-2.5 capitalize">
						<span class="size-2.5 rounded-full ring-1 ring-white/20" style:background={color}></span>
						{name}
					</ToggleGroupItem>
				{/each}
			</ToggleGroup>
		</div>

		{#each Object.entries(zones) as [id, z] (id)}
			<div
				role="group"
				aria-label={z.label}
				class="flex items-center justify-between border-t border-white/[0.06] py-3.5"
				onpointerenter={() => (hover = /** @type {ZoneId} */ (id))}
				onpointerleave={() => (hover = null)}
			>
				<span class="label">{z.label}</span>
				<Popover>
					<PopoverTrigger
						aria-label="Edit {z.label.toLowerCase()}"
						class="size-8 rounded-full ring-1 ring-white/20 ring-offset-2 ring-offset-black transition-[transform,box-shadow] duration-200 hover:scale-105 hover:ring-white/40 focus-visible:ring-2 focus-visible:ring-white/60 focus-visible:outline-none active:scale-95 data-[state=open]:ring-white/60"
						style="background: {swatchBg(z)}"
						onfocus={() => (hover = /** @type {ZoneId} */ (id))}
						onblur={() => (hover = null)}
					/>
					<PopoverContent align="end" class="w-60 gap-3 rounded-2xl border-white/10 bg-neutral-950/90 p-3 backdrop-blur-xl">
						<div class="grid grid-cols-5 gap-2">
							{#each SWATCHES as c (c)}
								{@const selected = !z.texture && colorOf(z) === c}
								<button
									type="button"
									aria-label={c}
									aria-pressed={selected}
									class="relative grid size-8 place-items-center rounded-full ring-1 ring-white/15 transition-transform duration-150 hover:scale-110 active:scale-95"
									style:background={c}
									onclick={() => {
										z.color = c;
										z.texture = null;
									}}
								>
									{#if selected}
										<span transition:fade={{ duration: 120 }}><Check class="size-3.5" color={isLight(c) ? '#000' : '#fff'} strokeWidth={3} /></span>
									{/if}
								</button>
							{/each}
							<label
								title="Custom colour"
								class="custom relative size-8 cursor-pointer rounded-full ring-1 ring-white/15 transition-transform duration-150 hover:scale-110 active:scale-95"
								class:selected={!z.texture && !SWATCHES.includes(colorOf(z))}
							>
								<input
									type="color"
									class="absolute inset-0 size-full cursor-pointer opacity-0"
									value={colorOf(z)}
									oninput={(e) => {
										z.color = e.currentTarget.value;
										z.texture = null;
									}}
								/>
							</label>
						</div>

						<div class="border-t border-white/10 pt-3">
							{#if z.texture}
								<div class="flex items-center gap-3" in:fade={{ duration: 150 }}>
									<img src={z.texture} alt="" class="size-8 shrink-0 rounded-lg object-cover ring-1 ring-white/15" />
									<Slider type="single" min={20} max={300} bind:value={z.scale} aria-label="Texture scale" />
									<Button variant="ghost" size="icon-sm" aria-label="Remove texture" onclick={() => (z.texture = null)}><X /></Button>
								</div>
							{:else}
								<label class="{buttonVariants({ variant: 'ghost', size: 'sm' })} w-full cursor-pointer justify-start text-white/70">
									<ImageUp />
									Use an image as texture
									<input type="file" accept="image/*" hidden onchange={(e) => pick(e, (url) => (z.texture = url))} />
								</label>
							{/if}
						</div>
					</PopoverContent>
				</Popover>
			</div>
		{/each}

		<div
			role="group"
			aria-label="Sponsor"
			class="border-t border-white/[0.06] pt-3.5"
			onpointerenter={() => (hover = 'sponsor')}
			onpointerleave={() => (hover = null)}
		>
			<div class="flex items-center justify-between">
				<span class="label">Sponsor</span>
				{#if sponsor}
					<Button variant="ghost" size="icon-sm" aria-label="Remove sponsor" onclick={() => (sponsor = null)}><X /></Button>
				{/if}
			</div>
			{#if sponsor}
				<div class="mt-3 flex items-center gap-3" in:fade={{ duration: 150 }}>
					<img src={sponsor} alt="" class="checker h-8 w-12 shrink-0 rounded-lg object-contain p-1 ring-1 ring-white/15" />
					<Slider type="single" min={60} max={200} bind:value={sponsorSize} aria-label="Sponsor size" />
				</div>
			{:else}
				<label
					class="mt-3 flex cursor-pointer flex-col items-center gap-1 rounded-2xl border border-dashed border-white/15 px-4 py-5 text-center text-xs text-white/50 transition-colors duration-200 hover:border-white/30 hover:text-white/70 focus-within:border-white/40"
				>
					<ImageUp class="mb-1 size-4" />
					<span>Upload a logo</span>
					<span class="text-white/30">or drop it on the kit</span>
					<input type="file" accept="image/*" class="sr-only" onchange={(e) => pick(e, (url) => (sponsor = url))} />
				</label>
			{/if}
		</div>

		<Button class="mt-6 w-full rounded-full" onclick={exportPng} disabled={exportState === 'busy'}>
			{#if exportState === 'busy'}
				<LoaderCircle class="animate-spin" /> Rendering
			{:else if exportState === 'done'}
				<Check /> Saved
			{:else}
				<Download /> Download kit
			{/if}
		</Button>
	</aside>
</main>

<style>
	.stage {
		background:
			radial-gradient(ellipse 60% 55% at 50% 45%, rgb(255 255 255 / 0.16), transparent 70%),
			radial-gradient(ellipse 40% 6% at 50% 93%, rgb(0 0 0 / 0.6), transparent),
			#0b0b0b;
	}

	.stage.dragging {
		border-color: rgb(255 255 255 / 0.35);
		box-shadow: inset 0 0 0 1px rgb(255 255 255 / 0.15);
	}

	.paint {
		transition:
			fill 0.35s ease,
			stroke 0.35s ease;
	}

	.hl {
		animation: pulse 1.6s ease-in-out infinite;
	}

	@keyframes pulse {
		0%,
		100% {
			opacity: 0.12;
		}
		50% {
			opacity: 0.35;
		}
	}

	.label {
		font: 0.8125rem 'Bebas Neue', sans-serif;
		letter-spacing: 0.2em;
		color: rgb(255 255 255 / 0.6);
	}

	.custom {
		background: conic-gradient(#f43f5e, #f59e0b, #84cc16, #06b6d4, #6366f1, #d946ef, #f43f5e);
	}

	.custom.selected {
		box-shadow: 0 0 0 2px #000, 0 0 0 3.5px rgb(255 255 255 / 0.7);
	}

	.checker {
		background: repeating-conic-gradient(rgb(255 255 255 / 0.08) 0 25%, transparent 0 50%) 0 0 / 8px 8px;
	}

	@media (prefers-reduced-motion: reduce) {
		.paint {
			transition: none;
		}
		.hl {
			animation: none;
			opacity: 0.25;
		}
	}
</style>
