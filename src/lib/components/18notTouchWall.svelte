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
	// Factor de anticipación para la detección de colisiones (como en el segundo código)
	const collisionLookahead = 20;

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

	// Función de detección de colisiones (mantiene la original para raycasting)
	function detectCollision(x, y) {
		let cellX = Math.floor(x / squareSize);
		let cellY = Math.floor(y / squareSize);

		// Verificar límites del mapa
		if (cellX < 0 || cellX >= map[0].length || cellY < 0 || cellY >= map.length) {
			return true;
		}

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
			// Calcular el offset de movimiento basado en el ángulo
			let offsetX = cos(angle) * velocity;
			let offsetY = sin(angle) * velocity;

			// Verificar posición futura con anticipación (como en el segundo código)
			let futureX = playerX + offsetX * collisionLookahead;
			let futureY = playerY + offsetY * collisionLookahead;

			// Solo mover si la posición futura está libre
			if (!detectCollision(futureX, playerY)) {
				playerX += offsetX;
			}
			if (!detectCollision(playerX, futureY)) {
				playerY += offsetY;
			}
		}
		if (keysPressed.ArrowDown) {
			// Calcular el offset de movimiento basado en el ángulo (hacia atrás)
			let offsetX = cos(angle) * velocity;
			let offsetY = sin(angle) * velocity;

			// Verificar posición futura con anticipación (como en el segundo código)
			let futureX = playerX - offsetX * collisionLookahead;
			let futureY = playerY - offsetY * collisionLookahead;

			// Solo mover si la posición futura está libre
			if (!detectCollision(futureX, playerY)) {
				playerX -= offsetX;
			}
			if (!detectCollision(playerX, futureY)) {
				playerY -= offsetY;
			}
		}
	}

	function drawMiniMap(ctx) {
		const minimapSize = 200;
		const minimapScale = minimapSize / mapSideSize;

		// Establecer color para los muros
		ctx.fillStyle = '#333';
		ctx.strokeStyle = '#666';

		for (let y = 0; y < map.length; y++) {
			for (let x = 0; x < map[y].length; x++) {
				if (map[y][x] === 1) {
					ctx.fillRect(
						x * squareSize * minimapScale,
						y * squareSize * minimapScale,
						squareSize * minimapScale,
						squareSize * minimapScale
					);
				}
				ctx.strokeRect(
					x * squareSize * minimapScale,
					y * squareSize * minimapScale,
					squareSize * minimapScale,
					squareSize * minimapScale
				);
			}
		}

		// Dibujar al jugador en el minimapa
		ctx.fillStyle = 'red';
		ctx.beginPath();
		ctx.arc(
			playerX * minimapScale,
			playerY * minimapScale,
			playerRadius * minimapScale,
			0,
			2 * Math.PI
		);
		ctx.fill();

		// Dibujar la dirección del jugador
		ctx.strokeStyle = 'red';
		ctx.lineWidth = 2;
		ctx.beginPath();
		ctx.moveTo(playerX * minimapScale, playerY * minimapScale);
		ctx.lineTo(
			(playerX + cos(angle) * playerRadius * 1.5) * minimapScale,
			(playerY + sin(angle) * playerRadius * 1.5) * minimapScale
		);
		ctx.stroke();

		// Opcional: Dibujar el jugador con su radio de colisión
		ctx.fillStyle = 'rgba(255, 255, 0, 0.3)';
		ctx.beginPath();
		ctx.arc(
			playerX * minimapScale,
			playerY * minimapScale,
			(playerRadius + collisionLookahead) * minimapScale,
			0,
			2 * Math.PI
		);
		ctx.fill();
	}

	function ray(ctx) {
		const rays = 100;
		const fov = Math.PI / 3;
		const rayStep = fov / rays;

		const middleY = canvasHeight / 2;

		ctx.fillStyle = '#666';

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
		ctx.strokeStyle = '#000';
		ctx.strokeRect(0, 0, canvasWidth, canvasHeight);

		movePlayer();
		ray(ctx);
		drawMiniMap(ctx);
		requestAnimationFrame(gameLoop);
	}

	onMount(() => {
		gameLoop();
	});
</script>

<canvas bind:this={canvas} width={canvasWidth} height={canvasHeight}></canvas>
<svelte:window onkeydown={handleKeydown} onkeyup={handleKeyup} />
