<script>
	import { onMount } from 'svelte';

	let canvas;
	const mapSideSize = 600;
	const canvasWidth = mapSideSize;
	const canvasHeight = mapSideSize;
	const squareSize = mapSideSize / 10;
	let playerX = $state(canvasWidth / 2);
	let playerY = $state(canvasHeight / 2 + 60);
	let angle = $state(0);
	const velocity = 3;

	// Un solo enemigo, movido más lejos del cubo central
	let enemy = $state({ x: 400, y: 400, color: 'red', size: 50 });

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

		if (cellX < 0 || cellY < 0 || cellX >= 10 || cellY >= 10) {
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
		const zBuffer = [];

		// Dibujar las paredes sin sombreado ni colores
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

			const fishEyeAngleCorrection = cos(rayAngle - angle);
			const correctedDistance = distance * fishEyeAngleCorrection;
			zBuffer[i] = correctedDistance;

			const columnHeight = (squareSize * canvasHeight) / correctedDistance;
			const columnWidth = canvasWidth / rays;
			const columnX = i * columnWidth;

			// Dibujar las paredes en negro
			ctx.fillStyle = 'black';
			ctx.fillRect(columnX, middleY - columnHeight / 2, columnWidth + 1, columnHeight);
		}

		// Renderizamos el enemigo
		renderSprite(ctx, zBuffer, rays);
	}

	function renderSprite(ctx, zBuffer, rays) {
		// Distancia y ángulo del enemigo
		const dx = enemy.x - playerX;
		const dy = enemy.y - playerY;
		const distance = Math.sqrt(dx * dx + dy * dy);
		let spriteAngle = Math.atan2(dy, dx);

		// Ajustar ángulo
		while (spriteAngle - angle > Math.PI) spriteAngle -= 2 * Math.PI;
		while (spriteAngle - angle < -Math.PI) spriteAngle += 2 * Math.PI;

		// Campo de visión
		const fov = Math.PI / 3;
		if (Math.abs(spriteAngle - angle) > fov / 2) return;

		// Pantalla
		const spriteSize = (canvasHeight / distance) * 50;
		const spriteX = ((spriteAngle - angle) / fov + 0.5) * canvasWidth - spriteSize / 2;
		const spriteY = canvasHeight / 2 - spriteSize / 2;

		// Verificar oclusión
		const spriteColumn = Math.floor(spriteX + spriteSize / 2) / (canvasWidth / rays);
		if (spriteColumn >= 0 && spriteColumn < rays && distance < zBuffer[Math.floor(spriteColumn)]) {
			ctx.fillStyle = enemy.color;
			ctx.fillRect(spriteX, spriteY, spriteSize, spriteSize);
		}
	}

	function gameLoop() {
		const ctx = canvas.getContext('2d');
		ctx.clearRect(0, 0, canvasWidth, canvasHeight);

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
