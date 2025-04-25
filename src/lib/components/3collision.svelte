<script>
	import { onMount } from 'svelte';

	let canvas;
	const sideCanvasSize = 600;
	const canvasWidth = sideCanvasSize;
	const canvasHeight = sideCanvasSize;

	const squareSize = sideCanvasSize / 10;

	let playerX = $state(canvasWidth / 2);
	let playerY = $state(canvasHeight / 2);
	const playerRadius = squareSize / 2;
	let angle = $state(0);
	let cos = $derived(Math.cos(angle));
	let sin = $derived(Math.sin(angle));
	const velocity = 3;
	const tileSize = 10;
	let keysPressed = $state({
		ArrowRight: false,
		ArrowLeft: false,
		ArrowUp: false,
		ArrowDown: false
	});

	const map = [
		[1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 1, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
	];

	function generateMap(ctx) {
		for (let x = 0; x < map[0].length; x++) {
			for (let y = 0; y < map.length; y++) {
				if (map[y][x] === 1) {
					ctx.fillRect(squareSize * x, squareSize * y, squareSize, squareSize);
				} else {
				}
				ctx.strokeRect(squareSize * x, squareSize * y, squareSize, squareSize);
			}
		}
	}

	function handleKeydown(event) {
		if (event.key in keysPressed) {
			keysPressed[event.key] = true;
		}
	}

	function handleKeyup(event) {
		if (event.key in keysPressed) {
			keysPressed[event.key] = false;
		}
	}

	function movePlayer() {
		if (keysPressed.ArrowRight) {
			angle += 0.1;
		}
		if (keysPressed.ArrowLeft) {
			angle -= 0.1;
		}
		if (keysPressed.ArrowUp) {
			playerX = Math.min(playerX + cos * velocity, canvasWidth - squareSize - playerRadius);
			playerY = Math.min(playerY + sin * velocity, canvasHeight - squareSize - playerRadius);
		}
		if (keysPressed.ArrowDown) {
			playerX -= cos * velocity;
			playerY -= sin * velocity;
		}
	}

	function gameLoop() {
		const ctx = canvas.getContext('2d');
		ctx.clearRect(0, 0, canvasWidth, canvasHeight);
		ctx.strokeRect(0, 0, canvasWidth, canvasHeight);

		ctx.beginPath();
		ctx.arc(playerX, playerY, playerRadius, 0, 2 * Math.PI);
		ctx.stroke();

		ctx.beginPath();
		ctx.moveTo(playerX, playerY);
		ctx.lineTo(playerX + playerRadius * 2 * cos, playerY + playerRadius * 2 * sin);
		ctx.stroke();

		generateMap(ctx);
		movePlayer();

		requestAnimationFrame(gameLoop);
	}

	onMount(() => {
		gameLoop();
	});
</script>

<p>angle: {angle}</p>
<p>cos: {cos}</p>
<p>sin: {sin}</p>
<canvas bind:this={canvas} width={canvasWidth} height={canvasHeight}></canvas>
<svelte:window onkeydown={handleKeydown} onkeyup={handleKeyup} />
