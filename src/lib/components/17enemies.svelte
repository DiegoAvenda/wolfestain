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

	// ========================================
	// ESTADOS
	// ========================================
	let canvas;
	let playerX = $state(CANVAS_WIDTH / 2);
	let playerY = $state(CANVAS_HEIGHT / 2 + 60);
	let angle = $state(0);
	let health = $state(100);
	let gameFrame = $state(0);
	let playerFrameX = $state(0);
	let isPlayerFiring = $state(false);
	let keysPressed = $state({
		ArrowRight: false,
		ArrowLeft: false,
		ArrowUp: false,
		ArrowDown: false
	});

	// ========================================
	// SISTEMA DE ENEMIGOS MÚLTIPLES
	// ========================================
	let enemies = $state([]);

	// Configuración inicial de enemigos
	const INITIAL_ENEMIES = [
		{ x: 400, y: 400, id: 1 },
		{ x: 150, y: 150, id: 2 },
		{ x: 450, y: 150, id: 3 },
		{ x: 150, y: 450, id: 4 }
	];

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
	const enemyStates = {
		WALKING: 'walking',
		ATTACKING: 'attacking',
		TAKING_DAMAGE: 'taking_damage',
		DEAD: 'dead'
	};

	// ========================================
	// FUNCIONES DE ENEMIGOS
	// ========================================
	function createEnemy(x, y, id) {
		return {
			x,
			y,
			id,
			state: enemyStates.WALKING,
			frameX: 0,
			frameY: 0,
			health: 30
		};
	}

	function initializeEnemies() {
		enemies = INITIAL_ENEMIES.map((config) => createEnemy(config.x, config.y, config.id));
	}

	function addEnemy(x, y) {
		const newId = Math.max(...enemies.map((e) => e.id), 0) + 1;
		enemies.push(createEnemy(x, y, newId));
	}

	function removeDeadEnemies() {
		enemies = enemies.filter((enemy) => enemy.state !== enemyStates.DEAD);
	}

	function getEnemyDistanceAndDirection(enemy) {
		const dx = playerX - enemy.x;
		const dy = playerY - enemy.y;
		const distance = calculateDistance(enemy.x, enemy.y, playerX, playerY);
		return { dx, dy, distance };
	}

	const enemyStateHandlers = {
		[enemyStates.WALKING]: (enemy) => {
			const { dx, dy, distance } = getEnemyDistanceAndDirection(enemy);
			const ENEMY_VELOCITY = 2;

			if (distance <= PLAYER_RADIUS + 10) {
				enemy.state = enemyStates.ATTACKING;
				enemy.frameX = 0;
				return;
			}

			enemy.x += (dx / distance) * ENEMY_VELOCITY;
			enemy.y += (dy / distance) * ENEMY_VELOCITY;
			enemy.frameY = 0;
			if (gameFrame % STAGGER_FRAME === 0) {
				enemy.frameX = (enemy.frameX + 1) % ENEMY_SPRITE_COLUMNS;
			}
		},
		[enemyStates.ATTACKING]: (enemy) => {
			const { distance } = getEnemyDistanceAndDirection(enemy);
			if (distance > PLAYER_RADIUS + 10) {
				enemy.state = enemyStates.WALKING;
				enemy.frameX = 0;
				return;
			}
			enemy.frameY = 1;
			if (gameFrame % (STAGGER_FRAME * 5) === 0) {
				enemy.frameX = (enemy.frameX + 1) % ENEMY_ATTACKING_SPRITE_COLUMNS;
				if (enemy.frameX === 1) {
					health -= 10;
					if (health <= 0) alert('Game Over!');
				}
			}
		},
		[enemyStates.TAKING_DAMAGE]: (enemy) => {
			enemy.frameY = 2;
			if (gameFrame % STAGGER_FRAME === 0) {
				enemy.frameX++;
				if (enemy.frameX >= ENEMY_SPRITE_COLUMNS) {
					enemy.health -= 10;
					if (enemy.health <= 0) {
						enemy.state = enemyStates.DEAD;
						enemy.frameX = 3;
						// Programar eliminación del enemigo después de un tiempo
						setTimeout(() => removeDeadEnemies(), 2000);
					} else {
						enemy.state = enemyStates.WALKING;
						enemy.frameX = 0;
					}
				}
			}
		},
		[enemyStates.DEAD]: (enemy) => {
			enemy.frameY = 2;
			enemy.frameX = 3;
		}
	};

	function updateAllEnemies() {
		enemies.forEach((enemy) => {
			if (enemyStateHandlers[enemy.state]) {
				enemyStateHandlers[enemy.state](enemy);
			}
		});
	}

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
		// Tecla para agregar enemigo (solo para pruebas)
		if (event.key === 'e') {
			const randomX = Math.random() * (MAP_SIZE - 100) + 50;
			const randomY = Math.random() * (MAP_SIZE - 100) + 50;
			if (!detectCollision(randomX, randomY)) {
				addEnemy(randomX, randomY);
			}
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

		const nextX = playerX + deltaX;
		const nextY = playerY + deltaY;

		if (!detectCollision(nextX, playerY)) playerX = nextX;
		if (!detectCollision(playerX, nextY)) playerY = nextY;
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
	// SISTEMA DE DISPAROS MEJORADO
	// ========================================
	function damageEnemy(enemy) {
		if (enemy.state !== enemyStates.DEAD) {
			enemy.state = enemyStates.TAKING_DAMAGE;
			enemy.frameX = 0;
		}
	}

	function processPlayerShooting() {
		if (!isPlayerFiring) return;

		// Buscar el enemigo más cercano en la línea de mira
		let closestEnemy = null;
		let closestDistance = Infinity;

		enemies.forEach((enemy) => {
			if (enemy.state === enemyStates.DEAD) return;

			const dx = enemy.x - playerX;
			const dy = enemy.y - playerY;
			const distance = calculateDistance(playerX, playerY, enemy.x, enemy.y);
			const enemyAngle = Math.atan2(dy, dx);
			let angleDifference = enemyAngle - angle;

			// Normalizar la diferencia de ángulo
			while (angleDifference > Math.PI) angleDifference -= 2 * Math.PI;
			while (angleDifference < -Math.PI) angleDifference += 2 * Math.PI;

			if (Math.abs(angleDifference) < FOV / 32 && distance < closestDistance) {
				closestDistance = distance;
				closestEnemy = enemy;
			}
		});

		if (closestEnemy) {
			damageEnemy(closestEnemy);
		}
	}

	function drawEnemy(ctx, enemy, enemySpriteX, enemySpriteY, enemySize) {
		ctx.drawImage(
			enemyImage,
			enemy.frameX * ENEMY_SPRITE_WIDTH,
			enemy.frameY * ENEMY_SPRITE_HEIGHT,
			ENEMY_SPRITE_WIDTH,
			ENEMY_SPRITE_HEIGHT,
			enemySpriteX,
			enemySpriteY,
			enemySize,
			enemySize
		);
	}

	// ========================================
	// RAYCASTING MEJORADO
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

	function renderAllEnemies(ctx, wallDistancePerRay) {
		// Ordenar enemigos por distancia (más lejanos primero)
		const sortedEnemies = [...enemies].sort((a, b) => {
			const distA = calculateDistance(playerX, playerY, a.x, a.y);
			const distB = calculateDistance(playerX, playerY, b.x, b.y);
			return distB - distA;
		});

		processPlayerShooting();

		sortedEnemies.forEach((enemy) => {
			const dx = enemy.x - playerX;
			const dy = enemy.y - playerY;
			const enemyDistance = calculateDistance(playerX, playerY, enemy.x, enemy.y);
			const enemyAngle = Math.atan2(dy, dx);
			const enemySize = (SQUARE_SIZE * CANVAS_HEIGHT) / enemyDistance;
			let angleDifference = enemyAngle - angle;

			// Normalizar la diferencia de ángulo
			while (angleDifference > Math.PI) angleDifference -= 2 * Math.PI;
			while (angleDifference < -Math.PI) angleDifference += 2 * Math.PI;

			// Solo renderizar si está en el campo de visión
			if (Math.abs(angleDifference) < FOV / 2 + 0.5) {
				const enemySpriteX = (angleDifference / FOV + 0.5) * CANVAS_WIDTH - enemySize / 2;
				const enemySpriteY = MIDDLE_Y - enemySize / 2;

				const spriteColumn = Math.floor((enemySpriteX + enemySize / 2) / (CANVAS_WIDTH / RAYS));
				if (
					spriteColumn >= 0 &&
					spriteColumn < RAYS &&
					wallDistancePerRay[spriteColumn] > enemyDistance
				) {
					drawEnemy(ctx, enemy, enemySpriteX, enemySpriteY, enemySize);
				}
			}
		});
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

		renderAllEnemies(ctx, wallDistancePerRay);
	}

	// ========================================
	// MINIMAPA MEJORADO
	// ========================================
	function drawMiniMap(ctx) {
		const MINI_MAP_SIZE = 200;
		const miniMapScale = MINI_MAP_SIZE / MAP_SIZE;

		// Dibujar mapa
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

		// Dibujar jugador
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

		// Dibujar enemigos en el minimapa
		ctx.fillStyle = 'red';
		enemies.forEach((enemy) => {
			if (enemy.state !== enemyStates.DEAD) {
				ctx.beginPath();
				ctx.arc(enemy.x * miniMapScale, enemy.y * miniMapScale, 3, 0, 2 * Math.PI);
				ctx.fill();
			}
		});
		ctx.fillStyle = 'black'; // Restaurar color
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
		updateAllEnemies();
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
		initializeEnemies();
		gameLoop();
	});
</script>

<canvas bind:this={canvas} width={CANVAS_WIDTH} height={CANVAS_HEIGHT}></canvas>
<svelte:window onkeydown={handleKeydown} onkeyup={handleKeyup} />
<div>
	<p>Health: {health}</p>
	<p>Enemies: {enemies.filter((e) => e.state !== enemyStates.DEAD).length}</p>
	<p>Controls: Arrow keys to move, F to shoot, E to add enemy (test)</p>
</div>
