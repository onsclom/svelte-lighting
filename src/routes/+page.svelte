<script lang="ts">
	import { onMount } from 'svelte';

	const WIDTH = 400;
	const HEIGHT = 224;
	const MAX_RADIUS = 400;

	let canvas: HTMLCanvasElement;
	let ambientLight = 0.5;
	let lightRadius = 0.5;
	let lightStrength = 0.5;
	let raf: number = 0;

	let mouseX = 0;
	let mouseY = 0;

	async function loadImage(url: string) {
		const response = await fetch(url);
		const blob = await response.blob();
		const image = await createImageBitmap(blob);
		return image;
	}

	let texture: ImageBitmap;
	let normal: ImageBitmap;
	let normalData: ImageData;

	function render() {
		const ctx = canvas.getContext('2d')!;
		ctx.drawImage(texture, 0, 0);
		const imageData = ctx.getImageData(0, 0, WIDTH, HEIGHT);

		const lightX = Math.floor(mouseX - canvas.getBoundingClientRect().left);
		const lightY = Math.floor(mouseY - canvas.getBoundingClientRect().top);

		const mousePixel = [Math.floor(lightX), Math.floor(lightY)];

		for (let i = 0; i < imageData.data.length; i += 4) {
			const x = (i / 4) % WIDTH;
			const y = Math.floor(i / 4 / WIDTH);
			const normalIndex = (y * WIDTH + x) * 4;
			const normalX = (normalData.data[normalIndex] * 2) / 255 - 1;
			const normalY = (normalData.data[normalIndex + 1] * 2) / 255 - 1;
			const normalZ = (normalData.data[normalIndex + 2] * 2) / 255;
			const r = imageData.data[i];
			const g = imageData.data[i + 1];
			const b = imageData.data[i + 2];

			let light = 0;

			const lightDist = Math.sqrt((x - lightX) ** 2 + (y - lightY) ** 2);
			const lightFactor = Math.max(0, 1 - lightDist / (MAX_RADIUS * lightRadius));
			light += lightFactor * lightStrength;

			const [normalToLightX, normalToLightY] = normalized(x - lightX, y - lightY);
			// const dotMult = dot(normalToLightX, normalToLightY, normalX, normalY);
			let dotMult = normalToLightX * -normalX + normalToLightY * normalY;
			dotMult = Math.max(0, dotMult);

			light *= dotMult;
			light += ambientLight;
			light = Math.min(1, light);

			imageData.data[i] = r * light;
			imageData.data[i + 1] = g * light;
			imageData.data[i + 2] = b * light;
		}
		ctx.putImageData(imageData, 0, 0);
		requestAnimationFrame(render);
	}

	function dot(x1: number, y1: number, x2: number, y2: number) {
		return x1 * x2 + y1 * y2;
	}

	function normalized(x: number, y: number) {
		const length = Math.sqrt(x * x + y * y);
		return [x / length, y / length];
	}

	async function setup() {
		canvas.width = WIDTH;
		canvas.height = HEIGHT;
		texture = await loadImage('/texturetest2.png');
		normal = await loadImage('/LevelNormal.png');
		const ctx = canvas.getContext('2d')!;
		ctx.drawImage(normal, 0, 0);
		normalData = ctx.getImageData(0, 0, WIDTH, HEIGHT);
		raf = requestAnimationFrame(render);
	}

	onMount(() => {
		setup();
		return () => cancelAnimationFrame(raf);
	});
</script>

<svelte:body
	on:pointermove={(e) => {
		mouseX = e.clientX;
		mouseY = e.clientY;
	}}
/>

<main>
	<canvas bind:this={canvas} />
</main>

<div class="ui">
	<div>
		<label style="display: flex; gap: 1rem">
			ambient light <input type="range" min="0" max="1" step="0.01" bind:value={ambientLight} />
		</label>
	</div>
	<div>
		<label style="display: flex; gap: 1rem">
			light radius <input type="range" min="0" max="1" step="0.01" bind:value={lightRadius} />
		</label>
	</div>
	<div>
		<label style="display: flex; gap: 1rem">
			light strength <input type="range" min="0" max="1" step="0.01" bind:value={lightStrength} />
		</label>
	</div>
</div>

<style>
	:global(html, body) {
		width: 100%;
		height: 100%;
		margin: 0;
		font-family: Menlo, Consolas, Monaco, Liberation Mono, Lucida Console, monospace;
	}

	.ui {
		position: fixed;
		top: 0;
		left: 0;
		background-color: rgba(0, 0, 0, 0.5);
		color: white;
		padding: 0.5rem;
	}

	main {
		width: 100%;
		height: 100%;
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
	}

	canvas {
		image-rendering: pixelated;
	}
</style>
