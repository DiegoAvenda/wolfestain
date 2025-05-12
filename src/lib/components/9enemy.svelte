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
		y: 3 * squareSize + squareSize / 2
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

	function gameLoop() {
		const ctx = canvas.getContext('2d');
		ctx.clearRect(0, 0, canvasWidth, canvasHeight);
		ctx.strokeRect(0, 0, canvasWidth, canvasHeight);

		movePlayer();
		ray(ctx);

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
