<script>
	import { onMount } from 'svelte';

	let canvas;
	let canvasWidth;
	let canvasHeight;
	let squareSize;

	// Room simple 16x16 con un cubo en el centro
	const map = [
		[1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
	];

	function updateCanvasSize() {
		canvasWidth = window.innerWidth;
		canvasHeight = window.innerHeight;

		// Calcular el tamaño de celda para que el mapa cuadrado se ajuste bien
		const minDimension = Math.min(canvasWidth, canvasHeight);
		squareSize = minDimension / map.length;
	}

	function generateMap(ctx) {
		// Configurar estilos
		ctx.fillStyle = '#8B4513'; // Color café para paredes (más Wolfenstein)
		ctx.strokeStyle = '#654321';
		ctx.lineWidth = 1;

		// Centrar el mapa en el canvas
		const mapPixelSize = squareSize * map.length;
		const offsetX = (canvasWidth - mapPixelSize) / 2;
		const offsetY = (canvasHeight - mapPixelSize) / 2;

		for (let x = 0; x < map[0].length; x++) {
			for (let y = 0; y < map.length; y++) {
				const pixelX = offsetX + squareSize * x;
				const pixelY = offsetY + squareSize * y;

				if (map[y][x] === 1) {
					ctx.fillRect(pixelX, pixelY, squareSize, squareSize);
				}
				ctx.strokeRect(pixelX, pixelY, squareSize, squareSize);
			}
		}
	}

	function gameLoop() {
		const ctx = canvas.getContext('2d');
		ctx.clearRect(0, 0, canvasWidth, canvasHeight);

		// Borde del canvas
		ctx.strokeStyle = '#000000';
		ctx.lineWidth = 2;
		ctx.strokeRect(0, 0, canvasWidth, canvasHeight);

		generateMap(ctx);

		requestAnimationFrame(gameLoop);
	}

	function handleResize() {
		updateCanvasSize();
	}

	onMount(() => {
		updateCanvasSize(); // Configurar tamaño inicial
		gameLoop();

		// Escuchar cambios de tamaño de ventana
		window.addEventListener('resize', handleResize);

		// Cleanup function
		return () => {
			window.removeEventListener('resize', handleResize);
		};
	});
</script>

<canvas bind:this={canvas} width={canvasWidth} height={canvasHeight}></canvas>

<style>
	:global(body) {
		margin: 0;
		padding: 0;
		overflow: hidden;
	}

	canvas {
		display: block;
		background-color: #f5f5f5;
	}
</style>
