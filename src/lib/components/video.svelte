<script>
	import { onMount } from 'svelte';

	let canvas;
	const mapSideSize = 600;
	const canvasWidth = mapSideSize;
	const canvasHeight = mapSideSize;
	const squareSize = mapSideSize / 10;
	let playerX = $state(canvasWidth / 2);
	let playerY = $state(canvasHeight / 2 + 60);
	const playerRadius = 50;
	let angle = $state(0);
	const velocity = 3;

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
		[1, 0, 0, 0, 0, 1, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
	];

	const cos = (angle) => Math.cos(angle);
	const sin = (angle) => Math.sin(angle);

	function detectCollision(x, y) {
		let cellX = Math.floor(x / squareSize);
		let cellY = Math.floor(y / squareSize);

		if (map[cellY][cellX] === 1) {
			return true;
		}

		return false;
	}

	function drawMap(ctx) {
		for (let y = 0; y < map.length; y++) {
			for (let x = 0; x < map[y].length; x++) {
				if (map[y][x] === 1) {
					ctx.fillRect(squareSize * x, squareSize * y, squareSize, squareSize);
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
			let nextX = playerX + cos(angle) * velocity;
			let nextY = playerY + sin(angle) * velocity;

			if (!detectCollision(nextX, playerY)) {
				playerX = nextX;
			}
			if (!detectCollision(playerX, nextY)) {
				playerY = nextY;
			}
		}
		if (keysPressed.ArrowDown) {
			let nextX = playerX - cos(angle) * velocity;
			let nextY = playerY - sin(angle) * velocity;

			if (!detectCollision(nextX, playerY)) {
				playerX = nextX;
			}
			if (!detectCollision(playerX, nextY)) {
				playerY = nextY;
			}
		}
	}

	function ray(ctx) {
		const rays = 50;
		const fov = Math.PI / 3;
		const rayStep = fov / rays;

		for (let i = 0; i < rays; i++) {
			let rayAngle = angle - fov / 2 + i * rayStep;

			let rayX = playerX;
			let rayY = playerY;

			ctx.beginPath();
			ctx.moveTo(playerX, playerY);

			while (true) {
				rayX += cos(rayAngle) * velocity;
				rayY += sin(rayAngle) * velocity;

				if (detectCollision(rayX, rayY)) {
					ctx.lineTo(rayX, rayY);
					break;
				}
			}
			ctx.stroke();
		}
	}

	function gameLoop() {
		const ctx = canvas.getContext('2d');
		ctx.clearRect(0, 0, canvasWidth, canvasHeight);
		ctx.strokeRect(0, 0, canvasWidth, canvasHeight);

		ctx.beginPath();
		ctx.arc(playerX, playerY, playerRadius, 0, 2 * Math.PI);
		ctx.stroke();

		drawMap(ctx);
		movePlayer();
		ray(ctx);

		requestAnimationFrame(gameLoop);
	}

	onMount(() => {
		gameLoop();
	});
</script>

<canvas bind:this={canvas} width={canvasWidth} height={canvasHeight}></canvas>
<svelte:window onkeydown={handleKeydown} onkeyup={handleKeyup} />
