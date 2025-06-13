<script>
	import { onMount } from 'svelte';

	let canvas;
	let canvasWidth = $state(0);
	let canvasHeight = $state(0);
	let squareSize = $state(0);
	let playerRadius = $derived(squareSize / 4);
	let angle = $state(0);
	const VELOCITY = 3;
	const FOV = Math.PI / 3; // 60° field of view
	const RAYS = 100;
	let enemyX = $state(0);
	let enemyY = $state(0);
	// Posición del jugador ahora es estado mutable
	let playerX = $state(0);
	let playerY = $state(0);
	let middleY = $derived(canvasHeight / 2);
	let enemyFrameX = $state(0);
	let enemyFrameY = $state(0);
	let enemyImage;
	let enemyImageLoaded = $state(false);
	let keysPressed = $state({
		ArrowRight: false,
		ArrowLeft: false,
		ArrowUp: false,
		ArrowDown: false
	});

	// ========================================
	// CONFIGURACIÓN DE SPRITES
	// ========================================
	const ENEMY_SPRITE_WIDTH = 71;
	const ENEMY_SPRITE_HEIGHT = 66;
	const ENEMY_SPRITE_COLUMNS = 4;

	const map = [
		[1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 1, 0, 1],
		[1, 0, 0, 0, 0, 0, 0, 0, 0, 1],
		[1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
	];

	// ========================================
	// FUNCIONES AUXILIARES
	// ========================================
	const cos = (angle) => Math.cos(angle);
	const sin = (angle) => Math.sin(angle);

	function calculateDistance(x1, y1, x2, y2) {
		const dx = x2 - x1;
		const dy = y2 - y1;
		return Math.sqrt(dx * dx + dy * dy);
	}

	function getEnemyDistanceAndDirection() {
		const dx = playerX - enemyX;
		const dy = playerY - enemyY;
		const distance = calculateDistance(enemyX, enemyY, playerX, playerY);
		return { dx, dy, distance };
	}

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
	// MÁQUINA DE ESTADOS DEL ENEMIGO
	// ========================================

	let currentEnemyState = $state('walking');
	let enemyAnimationFrame = $state(0);
	let enemyAnimationTimer = $state(0);

	const enemyStateHandlers = {
		walking: () => {
			const { dx, dy, distance } = getEnemyDistanceAndDirection();
			const ENEMY_VELOCITY = 1.5; // Velocidad más lenta para debugging

			console.log(`Enemy position: (${enemyX.toFixed(2)}, ${enemyY.toFixed(2)})`);
			console.log(`Player position: (${playerX.toFixed(2)}, ${playerY.toFixed(2)})`);
			console.log(`Distance: ${distance.toFixed(2)}`);

			if (distance <= playerRadius + 15) {
				currentEnemyState = 'attacking';
				enemyFrameX = 0;
				console.log('Enemy attacking!');
				return;
			}

			if (distance > 0) {
				// Normalizar el vector de dirección
				const normalizedDx = dx / distance;
				const normalizedDy = dy / distance;

				// Calcular nueva posición
				const newEnemyX = enemyX + normalizedDx * ENEMY_VELOCITY;
				const newEnemyY = enemyY + normalizedDy * ENEMY_VELOCITY;

				// Verificar colisiones antes de mover
				if (!detectCollision(newEnemyX, enemyY)) {
					enemyX = newEnemyX;
				}
				if (!detectCollision(enemyX, newEnemyY)) {
					enemyY = newEnemyY;
				}

				// Animar sprite
				enemyAnimationTimer++;
				if (enemyAnimationTimer >= 10) {
					// Cambiar frame cada 10 updates
					enemyAnimationFrame = (enemyAnimationFrame + 1) % ENEMY_SPRITE_COLUMNS;
					enemyFrameX = enemyAnimationFrame;
					enemyAnimationTimer = 0;
				}
			}
		},

		attacking: () => {
			console.log('Enemy in attack state');
			// Lógica de ataque aquí
			const { distance } = getEnemyDistanceAndDirection();
			if (distance > playerRadius + 20) {
				currentEnemyState = 'walking';
				console.log('Enemy back to walking');
			}
		}
	};

	// ========================================
	// RAYCASTING
	// ========================================
	function castRay(rayAngle) {
		const rayDirX = cos(rayAngle);
		const rayDirY = sin(rayAngle);

		// Evitar división por cero
		const deltaDistX = Math.abs(1 / rayDirX);
		const deltaDistY = Math.abs(1 / rayDirY);

		// Posición actual en la grid
		let mapX = Math.floor(playerX / squareSize);
		let mapY = Math.floor(playerY / squareSize);

		// Dirección del paso y distancia inicial
		let stepX, stepY;
		let sideDistX, sideDistY;

		if (rayDirX < 0) {
			stepX = -1;
			sideDistX = (playerX / squareSize - mapX) * deltaDistX;
		} else {
			stepX = 1;
			sideDistX = (mapX + 1.0 - playerX / squareSize) * deltaDistX;
		}

		if (rayDirY < 0) {
			stepY = -1;
			sideDistY = (playerY / squareSize - mapY) * deltaDistY;
		} else {
			stepY = 1;
			sideDistY = (mapY + 1.0 - playerY / squareSize) * deltaDistY;
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
			perpWallDist = (mapX - playerX / squareSize + (1 - stepX) / 2) / rayDirX;
		} else {
			perpWallDist = (mapY - playerY / squareSize + (1 - stepY) / 2) / rayDirY;
		}

		// Calcular punto exacto de intersección
		const hitX = playerX + perpWallDist * rayDirX * squareSize;
		const hitY = playerY + perpWallDist * rayDirY * squareSize;

		return {
			x: hitX,
			y: hitY,
			distance: perpWallDist * squareSize,
			side: side // 0 = vertical, 1 = horizontal
		};
	}

	function drawWallColumn(ctx, columnIndex, wallDistance) {
		const columnHeight = (squareSize * canvasHeight) / wallDistance;
		const columnWidth = canvasWidth / RAYS;
		const columnX = columnIndex * columnWidth;

		// Color diferente para paredes según el lado
		ctx.fillStyle = wallDistance > 0 ? '#666666' : '#999999';
		ctx.fillRect(columnX, canvasHeight / 2 - columnHeight / 2, columnWidth, columnHeight);
	}

	function renderEnemy(ctx, wallDistancePerRay) {
		if (!enemyImageLoaded) return;

		const dx = enemyX - playerX;
		const dy = enemyY - playerY;
		const enemyDistance = calculateDistance(playerX, playerY, enemyX, enemyY);
		const enemyAngle = Math.atan2(dy, dx);

		// Normalizar el ángulo para que esté dentro del rango [-PI, PI]
		let angleDifference = enemyAngle - angle;
		while (angleDifference > Math.PI) angleDifference -= 2 * Math.PI;
		while (angleDifference < -Math.PI) angleDifference += 2 * Math.PI;

		// Solo renderizar si el enemigo está dentro del campo de visión
		if (Math.abs(angleDifference) < FOV / 2) {
			const enemySize = (squareSize * canvasHeight) / enemyDistance;
			const enemySpriteX = (angleDifference / FOV + 0.5) * canvasWidth - enemySize / 2;
			const enemySpriteY = middleY - enemySize / 2;

			// Dibujar enemigo
			drawEnemy(ctx, enemySpriteX, enemySpriteY, enemySize);
		}
	}

	function performRaycasting(ctx) {
		const rayStep = FOV / RAYS;
		const wallDistancePerRay = [];

		// Lanzar rayos y dibujar paredes
		for (let i = 0; i < RAYS; i++) {
			const rayAngle = angle - FOV / 2 + i * rayStep;
			const wallResult = castRay(rayAngle);
			wallDistancePerRay[i] = wallResult;
			drawWallColumn(ctx, i, wallResult.distance);
		}

		// Renderizar enemigo después de las paredes
		renderEnemy(ctx, wallDistancePerRay);
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

		if (enemyX === 0 && enemyY === 0) {
			enemyX = squareSize * 7; // Posición del enemigo más alejada para testing
			enemyY = squareSize * 7;
		}

		middleY = canvasHeight / 2;
	}

	// ========================================
	// LÓGICA DEL JUGADOR
	// ========================================
	function updatePlayerRotation() {
		if (keysPressed.ArrowRight) angle += 0.05;
		if (keysPressed.ArrowLeft) angle -= 0.05;
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

	// ========================================
	// LÓGICA DEL ENEMIGO
	// ========================================

	function drawEnemy(ctx, enemySpriteX, enemySpriteY, enemySize) {
		if (enemyImageLoaded) {
			ctx.drawImage(
				enemyImage,
				enemyFrameX * ENEMY_SPRITE_WIDTH,
				enemyFrameY * ENEMY_SPRITE_HEIGHT,
				ENEMY_SPRITE_WIDTH,
				ENEMY_SPRITE_HEIGHT,
				enemySpriteX,
				enemySpriteY,
				enemySize,
				enemySize
			);
		} else {
			// Fallback: dibujar un rectángulo rojo si no hay imagen
			ctx.fillStyle = '#FF0000';
			ctx.fillRect(enemySpriteX, enemySpriteY, enemySize, enemySize);
		}
	}

	function gameLoop() {
		const ctx = canvas.getContext('2d');
		ctx.clearRect(0, 0, canvasWidth, canvasHeight);

		updatePlayerRotation();
		updatePlayerMovement();

		// Ejecutar lógica del enemigo
		if (enemyStateHandlers[currentEnemyState]) {
			enemyStateHandlers[currentEnemyState]();
		}

		performRaycasting(ctx);

		requestAnimationFrame(gameLoop);
	}

	onMount(() => {
		updateCanvasSize(); // Configurar tamaño inicial

		// Cargar imagen del enemigo
		enemyImage = new Image();
		enemyImage.onload = () => {
			enemyImageLoaded = true;
			console.log('Enemy image loaded successfully');
		};
		enemyImage.onerror = () => {
			console.log('Failed to load enemy image, using fallback');
			enemyImageLoaded = false;
		};
		enemyImage.src = '/boss.png';

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
<svelte:window onkeydown={handleKeydown} onkeyup={handleKeyup} onresize={updateCanvasSize} />

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
