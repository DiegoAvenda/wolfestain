<script>
	import { onMount } from 'svelte';
	import { browser } from '$app/environment';

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
	const fov = Math.PI / 3;
	const middleY = canvasHeight / 2;

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

	// Declarar la imagen pero solo inicializarla en el cliente
	let enemyImage = null;

	function drawMiniMap(ctx) {
		const miniMapSize = 200;
		const miniMapScale = miniMapSize / mapSideSize;

		for (let y = 0; y < map.length; y++) {
			for (let x = 0; x < map[y].length; x++) {
				if (map[y][x] === 1) {
					ctx.fillRect(
						squareSize * x * miniMapScale,
						squareSize * y * miniMapScale,
						squareSize * miniMapScale,
						squareSize * miniMapScale
					);
				}
				ctx.strokeRect(
					squareSize * x * miniMapScale,
					squareSize * y * miniMapScale,
					squareSize * miniMapScale,
					squareSize * miniMapScale
				);
			}
		}

		ctx.beginPath();
		ctx.arc(
			playerX * miniMapScale,
			playerY * miniMapScale,
			playerRadius * miniMapScale,
			0,
			2 * Math.PI
		);
		ctx.stroke();

		ctx.beginPath();
		ctx.moveTo(playerX * miniMapScale, playerY * miniMapScale);
		ctx.lineTo(
			(playerX + playerRadius * 2 * cos(angle)) * miniMapScale,
			(playerY + playerRadius * 2 * sin(angle)) * miniMapScale
		);
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

	function enemy(ctx) {
		const enemyX = 400;
		const enemyY = 400;
		const dx = enemyX - playerX;
		const dy = enemyY - playerY;
		const enemyDistance = Math.sqrt(dx * dx + dy * dy);
		const enemyAngle = Math.atan2(dy, dx);
		const enemySize = (squareSize * canvasHeight) / enemyDistance;
		const angleDifference = enemyAngle - angle;
		const spriteX = (angleDifference / fov + 0.5) * canvasWidth;
		const spriteY = middleY - enemySize / 2;

		// Solo intentar dibujar la imagen si existe y está completa
		if (enemyImage && enemyImage.complete) {
			ctx.drawImage(enemyImage, spriteX, spriteY, enemySize, enemySize);
		} else {
			// Fallback: dibujar un rectángulo
			ctx.fillStyle = 'red';
			ctx.fillRect(spriteX, spriteY, enemySize, enemySize);
			ctx.fillStyle = 'black'; // Restaurar color por defecto
		}
	}

	function ray(ctx) {
		const rays = 100;
		const rayStep = fov / rays;

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

			enemy(ctx);
		}
	}

	function gameLoop() {
		if (!canvas) return;

		const ctx = canvas.getContext('2d');
		ctx.clearRect(0, 0, canvasWidth, canvasHeight);
		ctx.strokeRect(0, 0, canvasWidth, canvasHeight);

		movePlayer();
		ray(ctx);
		drawMiniMap(ctx);

		requestAnimationFrame(gameLoop);
	}

	onMount(() => {
		// Solo ejecutar en el cliente
		if (browser) {
			// Inicializar la imagen del enemigo
			enemyImage = new Image();
			enemyImage.src = '/enemy.png';

			// Iniciar el game loop
			gameLoop();
		}
	});
</script>

<canvas bind:this={canvas} width={canvasWidth} height={canvasHeight}></canvas>
<svelte:window onkeydown={handleKeydown} onkeyup={handleKeyup} />
