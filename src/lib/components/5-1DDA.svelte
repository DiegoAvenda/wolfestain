<script>
	import { onMount } from 'svelte';

	let canvas;
	let canvasWidth = $state(0);
	let canvasHeight = $state(0);
	let squareSize = $state(0);
	let playerRadius = $derived(squareSize / 4);
	let angle = $state(0);
	const VELOCITY = 3;

	// Posición del jugador ahora es estado mutable
	let playerX = $state(0);
	let playerY = $state(0);

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

	// ========================================
	// FUNCIONES AUXILIARES
	// ========================================
	const cos = (angle) => Math.cos(angle);
	const sin = (angle) => Math.sin(angle);

	// ========================================
	// DETECCIÓN DE COLISIONES
	// ========================================
	function detectCollision(x, y) {
		const cellX = Math.floor(x / squareSize);
		const cellY = Math.floor(y / squareSize);
		return map[cellY]?.[cellX] === 1;
	}

	function checkCircleCollision(newX, newY) {
		// Verificar múltiples puntos alrededor del círculo del jugador
		const numPoints = 8;
		for (let i = 0; i < numPoints; i++) {
			const checkAngle = (i / numPoints) * 2 * Math.PI;
			const checkX = newX + cos(checkAngle) * playerRadius;
			const checkY = newY + sin(checkAngle) * playerRadius;

			if (detectCollision(checkX, checkY)) {
				return true;
			}
		}
		return false;
	}

	// ========================================
	// RAYCASTING
	// ========================================
	function castRay(startX, startY, rayAngle) {
		const rayDirX = cos(rayAngle);
		const rayDirY = sin(rayAngle);

		// Evitar división por cero
		const deltaDistX = Math.abs(1 / rayDirX);
		const deltaDistY = Math.abs(1 / rayDirY);

		// Posición actual en la grid
		let mapX = Math.floor(startX / squareSize);
		let mapY = Math.floor(startY / squareSize);

		// Dirección del paso y distancia inicial
		let stepX, stepY;
		let sideDistX, sideDistY;

		if (rayDirX < 0) {
			stepX = -1;
			sideDistX = (startX / squareSize - mapX) * deltaDistX;
		} else {
			stepX = 1;
			sideDistX = (mapX + 1.0 - startX / squareSize) * deltaDistX;
		}

		if (rayDirY < 0) {
			stepY = -1;
			sideDistY = (startY / squareSize - mapY) * deltaDistY;
		} else {
			stepY = 1;
			sideDistY = (mapY + 1.0 - startY / squareSize) * deltaDistY;
		}

		// Realizar DDA
		let hit = false;
		let side; // 0 = lado vertical, 1 = lado horizontal

		while (!hit) {
			// Saltar a la siguiente línea de la grid
			if (sideDistX < sideDistY) {
				sideDistX += deltaDistX;
				mapX += stepX;
				side = 0;
			} else {
				sideDistY += deltaDistY;
				mapY += stepY;
				side = 1;
			}

			// Verificar si golpeamos una pared
			if (map[mapY]?.[mapX] === 1) {
				hit = true;
			}
		}

		// Calcular distancia y punto de intersección
		let perpWallDist;
		if (side === 0) {
			perpWallDist = (mapX - startX / squareSize + (1 - stepX) / 2) / rayDirX;
		} else {
			perpWallDist = (mapY - startY / squareSize + (1 - stepY) / 2) / rayDirY;
		}

		// Calcular punto exacto de intersección
		const hitX = startX + perpWallDist * rayDirX * squareSize;
		const hitY = startY + perpWallDist * rayDirY * squareSize;

		return {
			x: hitX,
			y: hitY,
			distance: perpWallDist * squareSize,
			side: side // 0 = vertical, 1 = horizontal
		};
	}

	// ========================================
	// CONTROLES
	// ========================================
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

	function updateCanvasSize() {
		canvasWidth = window.innerWidth;
		canvasHeight = window.innerHeight;

		// Calcular el tamaño de celda
		const minDimension = Math.min(canvasWidth, canvasHeight);
		squareSize = minDimension / map.length;

		if (playerX === 0 && playerY === 0) {
			playerX = squareSize * 1.5; // Posición inicial en celda libre
			playerY = squareSize * 1.5;
		}
	}

	// ========================================
	// LÓGICA DEL JUGADOR
	// ========================================
	function updatePlayerRotation() {
		if (keysPressed.ArrowRight) angle += 0.1;
		if (keysPressed.ArrowLeft) angle -= 0.1;
	}

	function updatePlayerMovement() {
		let deltaX = 0;
		let deltaY = 0;

		if (keysPressed.ArrowUp) {
			deltaX = cos(angle) * VELOCITY;
			deltaY = sin(angle) * VELOCITY;
		}
		if (keysPressed.ArrowDown) {
			deltaX = -cos(angle) * VELOCITY;
			deltaY = -sin(angle) * VELOCITY;
		}

		const nextX = playerX + deltaX;
		const nextY = playerY + deltaY;

		// Usar los 8 puntos del círculo para verificación precisa
		if (!checkCircleCollision(nextX, playerY)) playerX = nextX;
		if (!checkCircleCollision(playerX, nextY)) playerY = nextY;
	}

	function drawPlayer(ctx) {
		// Dibujar el círculo del jugador
		ctx.strokeStyle = '#FF0000';
		ctx.lineWidth = 2;
		ctx.beginPath();
		ctx.arc(playerX, playerY, playerRadius, 0, 2 * Math.PI);
		ctx.stroke();

		// Lanzar y dibujar el rayo
		const rayResult = castRay(playerX, playerY, angle);

		// Color diferente según el lado golpeado (vertical vs horizontal)
		ctx.strokeStyle = rayResult.side === 0 ? '#FF6600' : '#FF9900';
		ctx.lineWidth = 2;
		ctx.beginPath();
		ctx.moveTo(playerX, playerY);
		ctx.lineTo(rayResult.x, rayResult.y);
		ctx.stroke();

		// Dibujar un punto en el final del rayo
		ctx.fillStyle = rayResult.side === 0 ? '#FF0000' : '#CC6600';
		ctx.beginPath();
		ctx.arc(rayResult.x, rayResult.y, 3, 0, 2 * Math.PI);
		ctx.fill();
	}

	function generateMap(ctx) {
		// Configurar estilos
		ctx.fillStyle = '#8B4513'; // Color café para paredes
		ctx.strokeStyle = '#654321';
		ctx.lineWidth = 1;

		for (let x = 0; x < map[0].length; x++) {
			for (let y = 0; y < map.length; y++) {
				const pixelX = squareSize * x;
				const pixelY = squareSize * y;

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
		updatePlayerRotation();
		updatePlayerMovement();
		drawPlayer(ctx);
		requestAnimationFrame(gameLoop);
	}

	onMount(() => {
		updateCanvasSize(); // Configurar tamaño inicial
		gameLoop();

		// Escuchar cambios de tamaño de ventana
		window.addEventListener('resize', updateCanvasSize);

		// Cleanup function
		return () => {
			window.removeEventListener('resize', updateCanvasSize);
		};
	});
</script>

<canvas bind:this={canvas} width={canvasWidth} height={canvasHeight}></canvas>
<svelte:window onkeydown={handleKeydown} onkeyup={handleKeyup} />

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
