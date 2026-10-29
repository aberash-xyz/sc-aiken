<script lang="ts">
	let video: HTMLVideoElement;
</script>

<svelte:head>
	<title>Sporting Club Aiken</title>
</svelte:head>

<!-- mobile browsers pause video when backgrounded; resume on return -->
<svelte:document onvisibilitychange={() => !document.hidden && video.play()} />

<main>
	<h1>Sporting Club Aiken</h1>
	<video bind:this={video} src="/logo-video-glass.mp4" autoplay muted loop playsinline aria-label="Sporting Club Aiken logo"></video>
</main>

<style>
	:global(html, body) {
		margin: 0;
		background: #000;
	}

	main {
		display: grid;
		place-content: center;
		justify-items: center;
		gap: 1.5rem;
		min-height: 100dvh;
	}

	video {
		width: min(270px, 45vw);
		height: auto;
	}

	h1 {
		margin: 0;
		font: 0.8125rem 'Bebas Neue', sans-serif;
		letter-spacing: 0.2em;
		color: rgb(255 255 255 / 0.6);
	}
</style>
