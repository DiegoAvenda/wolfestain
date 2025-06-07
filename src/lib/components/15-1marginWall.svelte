<script>
	import { onMount } from 'svelte';

	// ========================================
	// CONSTANTES Y CONFIGURACIÓN
	// ========================================
	const MAP_SIZE = 600;
	const CANVAS_WIDTH = MAP_SIZE;
	const CANVAS_HEIGHT = MAP_SIZE;
	const SQUARE_SIZE = MAP_SIZE / 10;
	const MIDDLE_Y = CANVAS_HEIGHT / 2;
	const PLAYER_RADIUS = 50;
	const VELOCITY = 3;
	const FOV = Math.PI / 3;
	const RAYS = 100;
	const STAGGER_FRAME = 6;
	const COLLISION_LOOK_AHEAD = 15;

	// ========================================
	// ESTADOS
	// ========================================
	let canvas;
	let playerX = $state(CANVAS_WIDTH / 2);
	let playerY = $state(CANVAS_HEIGHT / 2 + 60);
	let angle = $state(0);
	let health = $state(100);
	let enemyX = $state(400);
	let enemyY = $state(400);
	let gameFrame = $state(0);
	let enemyFrameX = $state(0);
	let enemyFrameY = $state(0);
	let playerFrameX = $state(0);
	let isPlayerFiring = $state(false);
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
	const ENEMY_ATTACKING_SPRITE_COLUMNS = 2;
	const PLAYER_SPRITE_WIDTH = 204;
	const PLAYER_SPRITE_HEIGHT = 216;
	const PLAYER_SPRITE_COLUMNS = 3;
	const PLAYER_FRAME_Y = 0;

	let enemyImage;
	let playerImage;

	// ========================================
	// MAPA
	// ========================================
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

	// ========================================
	// MÁQUINA DE ESTADOS DEL ENEMIGO
	// ========================================
	let currentEnemyState = $state('walking');

	const enemyStateHandlers = {
		walking: () => {
			const { dx, dy, distance } = getEnemyDistanceAndDirection();
			const ENEMY_VELOCITY = 2;

			if (distance <= PLAYER_RADIUS + 10) {
				currentEnemyState = 'attacking';
				enemyFrameX = 0;
				return;
			}

			enemyX += (dx / distance) * ENEMY_VELOCITY;
			enemyY += (dy / distance) * ENEMY_VELOCITY;
			enemyFrameY = 0;
			if (gameFrame % STAGGER_FRAME === 0) {
				enemyFrameX = (enemyFrameX + 1) % ENEMY_SPRITE_COLUMNS;
			}
		},
		attacking: () => {
			const { distance } = getEnemyDistanceAndDirection();
			if (distance > PLAYER_RADIUS + 10) {
				currentEnemyState = 'walking';
				enemyFrameX = 0;
				return;
			}
			enemyFrameY = 1;
			if (gameFrame % (STAGGER_FRAME * 4) === 0) {
				enemyFrameX = (enemyFrameX + 1) % ENEMY_ATTACKING_SPRITE_COLUMNS;
			}
		},
		taking_damage: () => {
			enemyFrameY = 2;
			if (gameFrame % STAGGER_FRAME === 0) {
				enemyFrameX++;
				if (enemyFrameX >= ENEMY_SPRITE_COLUMNS) {
					currentEnemyState = 'dead';
					enemyFrameX = 3;
				}
			}
		},
		dead: () => {
			enemyFrameY = 2;
			enemyFrameX = 3;
		}
	};

	// ========================================
	// FUNCIONES AUXILIARES
	// ========================================
	const cos = (angle) => Math.cos(angle);
	const sin = (angle) => Math.sin(angle);

	function detectCollision(x, y) {
		const cellX = Math.floor(x / SQUARE_SIZE);
		const cellY = Math.floor(y / SQUARE_SIZE);
		return map[cellY]?.[cellX] === 1;
	}

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
	// CONTROLES
	// ========================================
	function handleKeydown(event) {
		if (event.key in keysPressed) {
			keysPressed[event.key] = true;
		}
		if (event.key === 'f') {
			isPlayerFiring = true;
		}
	}

	function handleKeyup(event) {
		if (event.key in keysPressed) {
			keysPressed[event.key] = false;
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

		const nextX = playerX + deltaX * COLLISION_LOOK_AHEAD;
		const nextY = playerY + deltaY * COLLISION_LOOK_AHEAD;

		if (!detectCollision(nextX, playerY)) playerX += deltaX;
		if (!detectCollision(playerX, nextY)) playerY += deltaY;
	}

	function updatePlayerAnimation() {
		if (isPlayerFiring && gameFrame % STAGGER_FRAME === 0) {
			playerFrameX++;
			if (playerFrameX >= PLAYER_SPRITE_COLUMNS) {
				isPlayerFiring = false;
				playerFrameX = 0;
			}
		}
	}

	function drawPlayer(ctx) {
		const PLAYER_IMAGE_SIZE = 128 * 2;
		ctx.drawImage(
			playerImage,
			playerFrameX * PLAYER_SPRITE_WIDTH,
			PLAYER_FRAME_Y * PLAYER_SPRITE_HEIGHT,
			PLAYER_SPRITE_WIDTH,
			PLAYER_SPRITE_HEIGHT,
			CANVAS_WIDTH / 2 - PLAYER_IMAGE_SIZE / 2,
			CANVAS_HEIGHT - PLAYER_IMAGE_SIZE,
			PLAYER_IMAGE_SIZE,
			PLAYER_IMAGE_SIZE
		);
	}

	// ========================================
	// LÓGICA DEL ENEMIGO
	// ========================================
	function damageEnemy() {
		if (currentEnemyState !== 'dead') {
			currentEnemyState = 'taking_damage';
			enemyFrameX = 0;
		}
	}

	function processPlayerShooting() {
		if (!isPlayerFiring || currentEnemyState === 'dead') return;

		const dx = enemyX - playerX;
		const dy = enemyY - playerY;
		const enemyAngle = Math.atan2(dy, dx);
		const angleDifference = enemyAngle - angle;

		if (angleDifference < FOV / 32) {
			damageEnemy();
		}
	}

	function drawEnemy(ctx, enemySpriteX, enemySpriteY, enemySize) {
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
	}

	// ========================================
	// RAYCASTING
	// ========================================
	function castSingleRay(rayAngle) {
		let rayX = playerX;
		let rayY = playerY;
		let distance = 0;

		while (!detectCollision(rayX, rayY)) {
			rayX += cos(rayAngle);
			rayY += sin(rayAngle);
			distance += 1;
		}

		return distance * cos(rayAngle - angle);
	}

	function drawWallColumn(ctx, columnIndex, wallDistance) {
		const columnHeight = (SQUARE_SIZE * CANVAS_HEIGHT) / wallDistance;
		const columnWidth = CANVAS_WIDTH / RAYS;
		const columnX = columnIndex * columnWidth;

		ctx.fillRect(columnX, MIDDLE_Y - columnHeight / 2, columnWidth, columnHeight);
	}

	function renderEnemy(ctx, wallDistancePerRay) {
		const dx = enemyX - playerX;
		const dy = enemyY - playerY;
		const enemyDistance = calculateDistance(playerX, playerY, enemyX, enemyY);
		const enemyAngle = Math.atan2(dy, dx);
		const enemySize = (SQUARE_SIZE * CANVAS_HEIGHT) / enemyDistance;
		const angleDifference = enemyAngle - angle;
		const enemySpriteX = (angleDifference / FOV + 0.5) * CANVAS_WIDTH;
		const enemySpriteY = MIDDLE_Y - enemySize / 2;

		processPlayerShooting();

		const spriteColumn = Math.floor((enemySpriteX + enemySize / 2) / (CANVAS_WIDTH / RAYS));
		if (wallDistancePerRay[spriteColumn] > enemyDistance) {
			drawEnemy(ctx, enemySpriteX, enemySpriteY, enemySize);
		}
	}

	function performRaycasting(ctx) {
		const rayStep = FOV / RAYS;
		const wallDistancePerRay = [];

		for (let i = 0; i < RAYS; i++) {
			const rayAngle = angle - FOV / 2 + i * rayStep;
			const wallDistance = castSingleRay(rayAngle);
			wallDistancePerRay[i] = wallDistance;
			drawWallColumn(ctx, i, wallDistance);
		}

		renderEnemy(ctx, wallDistancePerRay);
	}

	// ========================================
	// MINIMAPA
	// ========================================
	function drawMiniMap(ctx) {
		const MINI_MAP_SIZE = 200;
		const miniMapScale = MINI_MAP_SIZE / MAP_SIZE;

		for (let y = 0; y < map.length; y++) {
			for (let x = 0; x < map[y].length; x++) {
				const cellX = SQUARE_SIZE * x * miniMapScale;
				const cellY = SQUARE_SIZE * y * miniMapScale;
				const cellSize = SQUARE_SIZE * miniMapScale;

				if (map[y][x] === 1) {
					ctx.fillRect(cellX, cellY, cellSize, cellSize);
				}
				ctx.strokeRect(cellX, cellY, cellSize, cellSize);
			}
		}

		ctx.beginPath();
		ctx.arc(
			playerX * miniMapScale,
			playerY * miniMapScale,
			PLAYER_RADIUS * miniMapScale,
			0,
			2 * Math.PI
		);
		ctx.stroke();

		ctx.beginPath();
		ctx.moveTo(playerX * miniMapScale, playerY * miniMapScale);
		ctx.lineTo(
			(playerX + PLAYER_RADIUS * 2 * cos(angle)) * miniMapScale,
			(playerY + PLAYER_RADIUS * 2 * sin(angle)) * miniMapScale
		);
		ctx.stroke();
	}

	// ========================================
	// BUCLE PRINCIPAL
	// ========================================
	function gameLoop() {
		const ctx = canvas.getContext('2d');
		ctx.clearRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);
		ctx.strokeRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);

		updatePlayerRotation();
		updatePlayerMovement();
		enemyStateHandlers[currentEnemyState]?.();
		updatePlayerAnimation();
		gameFrame++;

		performRaycasting(ctx);
		drawMiniMap(ctx);
		drawPlayer(ctx);

		requestAnimationFrame(gameLoop);
	}

	// ========================================
	// INICIALIZACIÓN
	// ========================================
	onMount(() => {
		playerImage = new Image();
		playerImage.src = '/weapon.png';
		enemyImage = new Image();
		enemyImage.src = '/boss.png';
		gameLoop();
	});
</script>

<canvas bind:this={canvas} width={CANVAS_WIDTH} height={CANVAS_HEIGHT}></canvas>
<svelte:window onkeydown={handleKeydown} onkeyup={handleKeyup} />
<p>Health: {health}</p>
