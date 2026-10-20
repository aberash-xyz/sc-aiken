<svelte:head>
	<title>Sporting Club Aiken</title>
</svelte:head>

<main>
	<video src="/logo-video-glass.mp4" autoplay muted loop playsinline aria-label="Sporting Club Aiken logo"></video>
</main>

<footer>Sporting Club Aiken</footer>

<style>
	:global(html, body) {
		margin: 0;
		background: #000;
	}

	main {
		display: grid;
		place-items: center;
		min-height: 100dvh;
	}

	video {
		width: min(540px, 90vw);
		height: auto;
	}

	footer {
		position: fixed;
		inset: auto 0 1.5rem;
		width: fit-content;
		margin: auto;
		padding: 0.6em 1.4em 0.6em 1.6em; /* extra left offsets trailing letter-spacing */
		border: 1px solid rgb(255 255 255 / 0.1);
		border-radius: 999px;
		background: linear-gradient(rgb(170 190 200 / 0.12), rgb(170 190 200 / 0.04));
		box-shadow: inset 0 1px rgb(255 255 255 / 0.12);
		backdrop-filter: blur(12px);
		font: 1rem 'Bebas Neue', sans-serif;
		letter-spacing: 0.2em;
		text-transform: uppercase;
		color: rgb(160, 172, 176);
	}
</style>
