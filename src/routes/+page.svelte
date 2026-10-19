<script lang="ts">
	import { onMount } from 'svelte';
	import type { Material, Texture } from 'three';
	import panoramaImage from '#lib/assets/Ililikha2.jpg';

	let viewer: HTMLDivElement;
	let isFullscreen = $state(false);
	let isDragging = $state(false);

	function zoomCamera(direction: number) {
		window.dispatchEvent(new CustomEvent('panorama-zoom', { detail: direction }));
	}

	async function toggleFullscreen() {
		if (!viewer) return;

		if (document.fullscreenElement) {
			await document.exitFullscreen();
		} else {
			await viewer.requestFullscreen();
		}
	}

	onMount(() => {
		let disposed = false;
		let frameId = 0;
		let yaw = 0;
		let pitch = 0;
		let previousX = 0;
		let previousY = 0;
		let pointerId: number | null = null;

		const initialize = async () => {
			const THREE = await import('three');
			if (disposed) return;

			const scene = new THREE.Scene();
			const camera = new THREE.PerspectiveCamera(75, 1, 0.1, 1100);
			const renderer = new THREE.WebGLRenderer({ antialias: true, powerPreference: 'high-performance' });
			renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
			renderer.outputColorSpace = THREE.SRGBColorSpace;
			renderer.domElement.className = 'panorama-canvas';
			renderer.domElement.setAttribute('aria-label', 'Interactive 360 degree panorama');
			viewer.appendChild(renderer.domElement);

			const texture = await new THREE.TextureLoader().loadAsync(panoramaImage);
			if (disposed) {
				texture.dispose();
				renderer.dispose();
				return;
			}
			texture.colorSpace = THREE.SRGBColorSpace;
			const sphere = new THREE.Mesh(
				new THREE.SphereGeometry(500, 96, 64),
				new THREE.MeshBasicMaterial({ map: texture, side: THREE.BackSide })
			);
			scene.add(sphere);

			const resize = () => {
				const { clientWidth, clientHeight } = viewer;
				camera.aspect = clientWidth / clientHeight;
				camera.updateProjectionMatrix();
				renderer.setSize(clientWidth, clientHeight, false);
			};
			const draw = () => {
				camera.rotation.set(pitch, yaw, 0, 'YXZ');
				renderer.render(scene, camera);
				frameId = requestAnimationFrame(draw);
			};
			const onPointerDown = (event: PointerEvent) => {
				pointerId = event.pointerId;
				previousX = event.clientX;
				previousY = event.clientY;
				isDragging = true;
				renderer.domElement.setPointerCapture(event.pointerId);
			};
			const onPointerMove = (event: PointerEvent) => {
				if (event.pointerId !== pointerId) return;
				yaw -= (event.clientX - previousX) * 0.0025;
				pitch = Math.max(-Math.PI / 2 + 0.05, Math.min(Math.PI / 2 - 0.05, pitch + (event.clientY - previousY) * 0.0025));
				previousX = event.clientX;
				previousY = event.clientY;
			};
			const onPointerUp = (event: PointerEvent) => {
				if (event.pointerId !== pointerId) return;
				pointerId = null;
				isDragging = false;
			};
			const onWheel = (event: WheelEvent) => {
				camera.fov = Math.max(35, Math.min(90, camera.fov + event.deltaY * 0.025));
				camera.updateProjectionMatrix();
			};
			const onZoom = (event: Event) => {
				camera.fov = Math.max(35, Math.min(90, camera.fov - (event as CustomEvent<number>).detail * 5));
				camera.updateProjectionMatrix();
			};
			const onKeyDown = (event: KeyboardEvent) => {
				const step = 0.08;
				if (event.key === 'ArrowLeft') yaw += step;
				else if (event.key === 'ArrowRight') yaw -= step;
				else if (event.key === 'ArrowUp') pitch = Math.min(Math.PI / 2 - 0.05, pitch + step);
				else if (event.key === 'ArrowDown') pitch = Math.max(-Math.PI / 2 + 0.05, pitch - step);
				else if (event.key === '+' || event.key === '=') onZoom(new CustomEvent('zoom', { detail: 1 }));
				else if (event.key === '-') onZoom(new CustomEvent('zoom', { detail: -1 }));
			};
			const onFullscreenChange = () => {
				isFullscreen = document.fullscreenElement === viewer;
				resize();
			};

			resize();
			draw();
			window.addEventListener('resize', resize);
			window.addEventListener('keydown', onKeyDown);
			window.addEventListener('panorama-zoom', onZoom);
			document.addEventListener('fullscreenchange', onFullscreenChange);
			renderer.domElement.addEventListener('pointerdown', onPointerDown);
			renderer.domElement.addEventListener('pointermove', onPointerMove);
			renderer.domElement.addEventListener('pointerup', onPointerUp);
			renderer.domElement.addEventListener('pointercancel', onPointerUp);
			renderer.domElement.addEventListener('wheel', onWheel, { passive: true });

			return () => {
				cancelAnimationFrame(frameId);
				window.removeEventListener('resize', resize);
				window.removeEventListener('keydown', onKeyDown);
				window.removeEventListener('panorama-zoom', onZoom);
				document.removeEventListener('fullscreenchange', onFullscreenChange);
				renderer.domElement.removeEventListener('pointerdown', onPointerDown);
				renderer.domElement.removeEventListener('pointermove', onPointerMove);
				renderer.domElement.removeEventListener('pointerup', onPointerUp);
				renderer.domElement.removeEventListener('pointercancel', onPointerUp);
				renderer.domElement.removeEventListener('wheel', onWheel);
				scene.remove(sphere);
				sphere.geometry.dispose();
				(texture as Texture).dispose();
				(sphere.material as Material).dispose();
				renderer.dispose();
				renderer.domElement.remove();
			};
		};

		let cleanup: (() => void) | undefined;
		initialize().then((dispose) => {
			cleanup = dispose;
		});

		return () => {
			disposed = true;
			cleanup?.();
		};
	});
</script>

<svelte:head>
	<title>Ililikha | 360 Panorama</title>
	<meta name="description" content="Explore the Ililikha 360 degree panorama." />
</svelte:head>

<main class="experience">
	<div class="viewer" bind:this={viewer} class:dragging={isDragging}>
		<div class="vignette" aria-hidden="true"></div>
		<header class="identity">
			<span class="eyebrow">360° · IMMERSIVE VIEW</span>
			<h1>Ililikha</h1>
		</header>
		<div class="coordinates" aria-hidden="true"><span></span> PANORAMA <span></span></div>
		<nav class="controls" aria-label="Panorama controls">
			<div class="zoom-control" aria-label="Zoom controls">
				<button type="button" aria-label="Zoom in" title="Zoom in" onclick={() => zoomCamera(1)}>+</button>
				<span class="divider"></span>
				<button type="button" aria-label="Zoom out" title="Zoom out" onclick={() => zoomCamera(-1)}>−</button>
			</div>
			<button class="fullscreen" type="button" aria-label={isFullscreen ? 'Exit fullscreen' : 'Enter fullscreen'} title={isFullscreen ? 'Exit fullscreen' : 'Enter fullscreen'} onclick={toggleFullscreen}>
				{#if isFullscreen}
					<svg viewBox="0 0 24 24" aria-hidden="true"><path d="M8 3v5H3M16 3v5h5M8 21v-5H3m13 5v-5h5" /></svg>
				{:else}
					<svg viewBox="0 0 24 24" aria-hidden="true"><path d="M3 9V3h6M21 9V3h-6M3 15v6h6m12-6v6h-6" /></svg>
				{/if}
			</button>
		</nav>
	</div>
</main>

<style>
	:global(*) { box-sizing: border-box; }
	:global(html), :global(body) { width: 100%; height: 100%; margin: 0; overflow: hidden; background: #101512; }
	:global(body) { font-family: 'Helvetica Neue', Helvetica, sans-serif; }
	:global(button) { font: inherit; }
	.experience, .viewer { width: 100%; height: 100%; min-height: 100svh; }
	.viewer { position: relative; overflow: hidden; background: #171d18; cursor: grab; }
	.viewer.dragging { cursor: grabbing; }
	:global(.panorama-canvas) { display: block; width: 100%; height: 100%; touch-action: none; outline: none; }
	.vignette { position: absolute; z-index: 1; inset: 0; pointer-events: none; background: linear-gradient(180deg, rgb(8 13 10 / 58%) 0%, transparent 24%, transparent 67%, rgb(8 13 10 / 35%) 100%); }
	.identity { position: absolute; z-index: 2; top: clamp(26px, 5vw, 58px); left: clamp(24px, 5vw, 68px); color: #fff; pointer-events: none; }
	.eyebrow { display: block; margin-bottom: 9px; color: rgb(255 255 255 / 72%); font-size: 10px; font-weight: 600; letter-spacing: 0.2em; }
	h1 { margin: 0; font-family: Georgia, 'Times New Roman', serif; font-size: clamp(30px, 4vw, 49px); font-weight: 400; line-height: 1; }
	.coordinates { position: absolute; z-index: 2; bottom: clamp(32px, 5vw, 58px); left: 50%; display: flex; align-items: center; gap: 12px; transform: translateX(-50%); color: rgb(255 255 255 / 77%); font-size: 9px; font-weight: 600; letter-spacing: 0.22em; pointer-events: none; white-space: nowrap; }
	.coordinates span { width: 28px; height: 1px; background: rgb(255 255 255 / 55%); }
	.controls { position: absolute; z-index: 2; right: clamp(20px, 4vw, 48px); bottom: clamp(22px, 4vw, 40px); display: flex; align-items: center; gap: 9px; }
	.zoom-control, .fullscreen { height: 44px; border: 1px solid rgb(255 255 255 / 28%); background: rgb(20 27 22 / 55%); color: white; backdrop-filter: blur(12px); }
	.zoom-control { display: flex; align-items: center; border-radius: 3px; }
	.zoom-control button, .fullscreen { display: grid; width: 44px; padding: 0; place-items: center; border: 0; color: inherit; background: transparent; cursor: pointer; }
	.zoom-control button { font-size: 22px; font-weight: 300; }
	.zoom-control button:hover, .fullscreen:hover { background: rgb(255 255 255 / 12%); }
	.zoom-control button:focus-visible, .fullscreen:focus-visible { outline: 2px solid #d7c8a0; outline-offset: 2px; }
	.zoom-control .divider { width: 1px; height: 19px; background: rgb(255 255 255 / 25%); }
	.fullscreen { border: 1px solid rgb(255 255 255 / 28%); border-radius: 3px; backdrop-filter: blur(12px); }
	.fullscreen svg { width: 19px; height: 19px; fill: none; stroke: currentColor; stroke-linecap: square; stroke-linejoin: miter; stroke-width: 1.5; }
	@media (max-width: 600px) {
		.identity { top: 25px; left: 23px; }
		.controls { right: 19px; bottom: 20px; }
		.coordinates { bottom: 36px; left: 24px; transform: none; }
		.coordinates span:last-child { display: none; }
		.zoom-control, .fullscreen { height: 42px; }
		.zoom-control button, .fullscreen { width: 42px; }
	}
</style>
