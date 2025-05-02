<script>
	import { onMount } from 'svelte';

	let canvas;
	const mapSideSize = 600;
	const canvasWidth = mapSideSize;
	const canvasHeight = mapSideSize;
	const squareSize = mapSideSize / 10;
	let playerX = $state(canvasWidth / 2 + 100);
	let playerY = $state(canvasHeight / 2);
	const playerRadius = 20;
	let angle = $state(0);
	let cos = $derived(Math.cos(angle));
	let sin = $derived(Math.sin(angle));
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

	// Función para verificar colisión entre un círculo y un rectángulo
	function checkCollision(x, y, radius) {
		const cellX = Math.floor(x / squareSize);
		const cellY = Math.floor(y / squareSize);

		for (let cy = Math.max(0, cellY - 1); cy <= Math.min(map.length - 1, cellY + 1); cy++) {
			for (let cx = Math.max(0, cellX - 1); cx <= Math.min(map[0].length - 1, cellX + 1); cx++) {
				if (map[cy][cx] === 1) {
					const rectX = cx * squareSize;
					const rectY = cy * squareSize;

					// Calcular el punto más cercano del rectángulo al centro del círculo
					const closestX = Math.max(rectX, Math.min(x, rectX + squareSize));
					const closestY = Math.max(rectY, Math.min(y, rectY + squareSize));

					// Calcular la distancia a ese punto
					const distX = x - closestX;
					const distY = y - closestY;
					const distSquared = distX * distX + distY * distY;

					if (distSquared < radius * radius) {
						return true;
					}
				}
			}
		}
		return false;
	}

	// Función para manejar colisiones y permitir deslizamiento
	function checkCollisionAndSlide(newX, newY) {
		const currentX = playerX;
		const currentY = playerY;

		// Intentar movimiento horizontal
		const canMoveX = !checkCollision(newX, currentY, playerRadius);
		// Intentar movimiento vertical
		const canMoveY = !checkCollision(currentX, newY, playerRadius);

		// Aplicar movimientos permitidos
		if (canMoveX) {
			playerX = newX;
		}

		if (canMoveY) {
			playerY = newY;
		}

		// Si ambos movimientos están bloqueados, el jugador se queda en su lugar
	}

	function drawMap(ctx) {
		ctx.fillStyle = 'gray';
		ctx.strokeStyle = 'black';

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
		// Rotar jugador
		if (keysPressed.ArrowRight) {
			angle += 0.1;
		}
		if (keysPressed.ArrowLeft) {
			angle -= 0.1;
		}

		// Mover jugador
		if (keysPressed.ArrowUp) {
			const newX = playerX + cos * velocity;
			const newY = playerY + sin * velocity;
			checkCollisionAndSlide(newX, newY);
		}
		if (keysPressed.ArrowDown) {
			const newX = playerX - cos * velocity;
			const newY = playerY - sin * velocity;
			checkCollisionAndSlide(newX, newY);
		}

		// Mantener al jugador dentro de los límites del canvas
		playerX = Math.max(playerRadius, Math.min(canvasWidth - playerRadius, playerX));
		playerY = Math.max(playerRadius, Math.min(canvasHeight - playerRadius, playerY));
	}

	function gameLoop() {
		const ctx = canvas.getContext('2d');
		ctx.clearRect(0, 0, canvasWidth, canvasHeight);

		// Dibujar borde del canvas
		ctx.strokeStyle = 'black';
		ctx.strokeRect(0, 0, canvasWidth, canvasHeight);

		// Dibujar mapa primero
		drawMap(ctx);

		// Dibujar jugador después
		ctx.fillStyle = 'blue';
		ctx.strokeStyle = 'black';

		// Dibujar círculo del jugador
		ctx.beginPath();
		ctx.arc(playerX, playerY, playerRadius, 0, 2 * Math.PI);
		ctx.fill();
		ctx.stroke();

		// Dibujar dirección del jugador
		ctx.beginPath();
		ctx.moveTo(playerX, playerY);
		ctx.lineTo(playerX + playerRadius * 2 * cos, playerY + playerRadius * 2 * sin);
		ctx.stroke();

		// Actualizar posición del jugador
		movePlayer();

		// Continuar el bucle
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
		display: block;
		margin: 20px auto;
		border: 1px solid #000;
	}
</style>
