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

	// Función para dibujar el minimapa
	function drawMinimap(ctx) {
		// Configuración del minimapa
		const minimapSize = 200;
		const minimapX = canvasWidth - minimapSize - 20;
		const minimapY = 20;
		const minimapScale = minimapSize / mapSideSize;

		// Fondo del minimapa
		ctx.fillStyle = 'rgba(0, 0, 0, 0.5)';
		ctx.fillRect(minimapX, minimapY, minimapSize, minimapSize);
		ctx.strokeStyle = 'white';
		ctx.strokeRect(minimapX, minimapY, minimapSize, minimapSize);

		// Dibujar el mapa en el minimapa
		ctx.fillStyle = 'white';
		for (let y = 0; y < map.length; y++) {
			for (let x = 0; x < map[y].length; x++) {
				if (map[y][x] === 1) {
					ctx.fillRect(
						minimapX + x * squareSize * minimapScale,
						minimapY + y * squareSize * minimapScale,
						squareSize * minimapScale,
						squareSize * minimapScale
					);
				}
				ctx.strokeRect(
					minimapX + x * squareSize * minimapScale,
					minimapY + y * squareSize * minimapScale,
					squareSize * minimapScale,
					squareSize * minimapScale
				);
			}
		}

		// Dibujar al jugador en el minimapa
		ctx.fillStyle = 'red';
		ctx.beginPath();
		ctx.arc(
			minimapX + playerX * minimapScale,
			minimapY + playerY * minimapScale,
			playerRadius * minimapScale,
			0,
			2 * Math.PI
		);
		ctx.fill();

		// Dibujar la dirección del jugador
		ctx.strokeStyle = 'yellow';
		ctx.lineWidth = 2;
		ctx.beginPath();
		ctx.moveTo(minimapX + playerX * minimapScale, minimapY + playerY * minimapScale);
		ctx.lineTo(
			minimapX + (playerX + cos(angle) * playerRadius * 1.5) * minimapScale,
			minimapY + (playerY + sin(angle) * playerRadius * 1.5) * minimapScale
		);
		ctx.stroke();
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

	function drawPlayer(ctx) {
		ctx.beginPath();
		ctx.arc(playerX, playerY, playerRadius, 0, 2 * Math.PI);
		ctx.stroke();
	}

	function detectCollision(x, y) {
		let cellX = Math.floor(x / squareSize);
		let cellY = Math.floor(y / squareSize);

		if (map[cellY][cellX] === 1) {
			return true;
		}

		return false;
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
		const rays = 100;
		const fov = Math.PI / 3;
		const rayStep = fov / rays;

		const middleY = canvasHeight / 2;

		for (let i = 0; i < rays; i++) {
			let rayAngle = angle - fov / 2 + i * rayStep;

			let rayX = playerX;
			let rayY = playerY;
			let distance = 0;

			while (true) {
				rayX += cos(rayAngle);
				rayY += sin(rayAngle);
				distance += 1;

				if (detectCollision(rayX, rayY)) {
					break;
				}
			}
			const correctedDistance = distance * cos(rayAngle - angle);

			const columnHeight = (squareSize * canvasHeight) / correctedDistance;
			const columnWidth = canvasWidth / rays;
			const columnX = i * columnWidth;

			ctx.fillRect(columnX, middleY - columnHeight / 2, columnWidth, columnHeight);
		}
	}

	function gameLoop() {
		const ctx = canvas.getContext('2d');
		ctx.clearRect(0, 0, canvasWidth, canvasHeight);
		ctx.strokeRect(0, 0, canvasWidth, canvasHeight);

		movePlayer();
		ray(ctx);

		// Dibujar el minimapa
		drawMinimap(ctx);

		requestAnimationFrame(gameLoop);
	}

	onMount(() => {
		gameLoop();
	});
</script>

<canvas bind:this={canvas} width={canvasWidth} height={canvasHeight}></canvas>
<svelte:window onkeydown={handleKeydown} onkeyup={handleKeyup} />

<style>
	canvas {
		background-color: black;
	}
</style>
