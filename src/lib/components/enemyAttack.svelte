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
	let playerHealth = $state(100);

	// Enemigo
	let enemy = $state({
		x: 3 * squareSize + squareSize / 2,
		y: 3 * squareSize + squareSize / 2,
		size: squareSize / 2,
		speed: 1.5,
		health: 100,
		damage: 10,
		attackCooldown: 0,
		state: 'patrol', // 'patrol', 'chase', 'attack'
		color: 'red'
	});

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

		if (cellX < 0 || cellX >= map[0].length || cellY < 0 || cellY >= map.length) {
			return true;
		}

		return map[cellY][cellX] === 1;
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

	// Detección de visión del enemigo
	function canEnemySeePlayer() {
		// Ángulo hacia el jugador
		const angleToPlayer = Math.atan2(playerY - enemy.y, playerX - enemy.x);

		// Lanzar un rayo desde el enemigo hacia el jugador
		let rayX = enemy.x;
		let rayY = enemy.y;
		let distance = 0;
		const maxDistance = 10 * squareSize;

		while (distance < maxDistance) {
			rayX += cos(angleToPlayer) * 5;
			rayY += sin(angleToPlayer) * 5;
			distance += 5;

			// Si choca con una pared, no puede ver al jugador
			if (detectCollision(rayX, rayY)) {
				return false;
			}

			// Si llega cerca del jugador, lo ha visto
			const distToPlayer = Math.sqrt((playerX - rayX) ** 2 + (playerY - rayY) ** 2);
			if (distToPlayer < playerRadius) {
				return true;
			}
		}
		return false;
	}

	// IA del enemigo
	function updateEnemy() {
		const distToPlayer = Math.sqrt((playerX - enemy.x) ** 2 + (playerY - enemy.y) ** 2);

		// Lógica de estados
		if (distToPlayer < squareSize) {
			enemy.state = 'attack';
		} else if (canEnemySeePlayer() || distToPlayer < 3 * squareSize) {
			enemy.state = 'chase';
		} else {
			enemy.state = 'patrol';
		}

		// Comportamiento según estado
		if (enemy.state === 'chase') {
			// Moverse hacia el jugador
			const angleToPlayer = Math.atan2(playerY - enemy.y, playerX - enemy.x);
			const moveX = cos(angleToPlayer) * enemy.speed;
			const moveY = sin(angleToPlayer) * enemy.speed;

			if (!detectCollision(enemy.x + moveX, enemy.y)) {
				enemy.x += moveX;
			}
			if (!detectCollision(enemy.x, enemy.y + moveY)) {
				enemy.y += moveY;
			}
		} else if (enemy.state === 'attack' && enemy.attackCooldown <= 0) {
			// Atacar al jugador
			playerHealth -= enemy.damage;
			enemy.attackCooldown = 60; // 1 segundo (60 frames)
		}

		// Enfriamiento de ataque
		if (enemy.attackCooldown > 0) {
			enemy.attackCooldown--;
		}
	}

	function ray(ctx) {
		const rays = 100;
		const fov = Math.PI / 3;
		const rayStep = fov / rays;
		const middleY = canvasHeight / 2;

		// Array para almacenar información de los rayos de paredes
		let wallRayInfo = [];

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
			wallRayInfo.push({ distance: correctedDistance, angle: rayAngle });

			const columnHeight = (squareSize * canvasHeight) / correctedDistance;
			const columnWidth = canvasWidth / rays;
			const columnX = i * columnWidth;

			// Dibujar pared
			ctx.fillStyle = 'gray';
			ctx.fillRect(columnX, middleY - columnHeight / 2, columnWidth, columnHeight);
		}

		// Dibujar enemigo como sprite 3D
		drawEnemySprite(ctx, wallRayInfo);
	}

	function drawEnemySprite(ctx, wallRayInfo) {
		// Calcular posición relativa del enemigo
		const relX = enemy.x - playerX;
		const relY = enemy.y - playerY;

		// Calcular distancia y ángulo del enemigo
		const enemyDist = Math.sqrt(relX * relX + relY * relY);
		const enemyAngle = Math.atan2(relY, relX);

		// Ajustar ángulo para que esté en el rango del jugador
		let angleDiff = enemyAngle - angle;
		while (angleDiff > Math.PI) angleDiff -= 2 * Math.PI;
		while (angleDiff < -Math.PI) angleDiff += 2 * Math.PI;

		// Solo dibujar si el enemigo está en el campo de visión
		const fov = Math.PI / 3;
		if (Math.abs(angleDiff) < fov / 2 && enemyDist > 0) {
			// Calcular posición en pantalla
			const screenX = ((angleDiff + fov / 2) / fov) * canvasWidth;
			const spriteHeight = (squareSize * canvasHeight) / (enemyDist * cos(angleDiff));

			// Verificar si el enemigo está oculto por una pared
			let isVisible = true;
			const rayIndex = Math.floor(screenX / (canvasWidth / wallRayInfo.length));

			if (rayIndex >= 0 && rayIndex < wallRayInfo.length) {
				if (enemyDist > wallRayInfo[rayIndex].distance) {
					isVisible = false;
				}
			}

			if (isVisible) {
				ctx.fillStyle = enemy.color;
				ctx.fillRect(
					screenX - spriteHeight / 4,
					canvasHeight / 2 - spriteHeight / 2,
					spriteHeight / 2,
					spriteHeight
				);
			}
		}
	}

	function drawMinimap(ctx) {
		const minimapSize = 150;
		const scale = minimapSize / mapSideSize;
		const offset = 10;

		// Dibujar mapa
		ctx.fillStyle = 'white';
		ctx.fillRect(offset, offset, minimapSize, minimapSize);

		// Dibujar paredes
		ctx.fillStyle = 'black';
		for (let y = 0; y < map.length; y++) {
			for (let x = 0; x < map[y].length; x++) {
				if (map[y][x] === 1) {
					ctx.fillRect(
						offset + x * squareSize * scale,
						offset + y * squareSize * scale,
						squareSize * scale,
						squareSize * scale
					);
				}
			}
		}

		// Dibujar jugador
		ctx.fillStyle = 'blue';
		ctx.beginPath();
		ctx.arc(
			offset + playerX * scale,
			offset + playerY * scale,
			playerRadius * scale,
			0,
			Math.PI * 2
		);
		ctx.fill();

		// Dibujar dirección del jugador
		ctx.strokeStyle = 'blue';
		ctx.lineWidth = 2;
		ctx.beginPath();
		ctx.moveTo(offset + playerX * scale, offset + playerY * scale);
		ctx.lineTo(
			offset + (playerX + cos(angle) * playerRadius * 2) * scale,
			offset + (playerY + sin(angle) * playerRadius * 2) * scale
		);
		ctx.stroke();

		// Dibujar enemigo
		ctx.fillStyle = 'red';
		ctx.beginPath();
		ctx.arc(offset + enemy.x * scale, offset + enemy.y * scale, enemy.size * scale, 0, Math.PI * 2);
		ctx.fill();
	}

	function drawHUD(ctx) {
		// Barra de salud
		ctx.fillStyle = 'black';
		ctx.fillRect(20, canvasHeight - 40, 200, 20);
		ctx.fillStyle = playerHealth > 30 ? 'green' : 'red';
		ctx.fillRect(20, canvasHeight - 40, 200 * (playerHealth / 100), 20);
		ctx.strokeStyle = 'white';
		ctx.strokeRect(20, canvasHeight - 40, 200, 20);

		// Texto de salud
		ctx.fillStyle = 'white';
		ctx.font = '16px Arial';
		ctx.fillText(`Salud: ${playerHealth}%`, 30, canvasHeight - 25);
	}

	function gameLoop() {
		const ctx = canvas.getContext('2d');
		ctx.clearRect(0, 0, canvasWidth, canvasHeight);
		ctx.strokeRect(0, 0, canvasWidth, canvasHeight);

		movePlayer();
		updateEnemy();
		ray(ctx);
		drawMinimap(ctx);
		drawHUD(ctx);

		// Game over si salud llega a 0
		if (playerHealth <= 0) {
			ctx.fillStyle = 'rgba(0, 0, 0, 0.7)';
			ctx.fillRect(0, 0, canvasWidth, canvasHeight);
			ctx.fillStyle = 'red';
			ctx.font = '48px Arial';
			ctx.textAlign = 'center';
			ctx.fillText('GAME OVER', canvasWidth / 2, canvasHeight / 2);
			ctx.font = '24px Arial';
			ctx.fillText('Recarga la página para jugar de nuevo', canvasWidth / 2, canvasHeight / 2 + 50);
			return;
		}

		requestAnimationFrame(gameLoop);
	}

	onMount(() => {
		gameLoop();
	});
</script>

<canvas bind:this={canvas} width={canvasWidth} height={canvasHeight}></canvas>
<svelte:window on:keydown={handleKeydown} on:keyup={handleKeyup} />

<style>
	canvas {
		display: block;
		margin: 0 auto;
		background-color: black;
	}
</style>
