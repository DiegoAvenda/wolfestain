<script>
	import { onMount } from 'svelte';

	let canvas;
	let enemyImage;
	let playerImage;
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
	const rays = 100;
	let enemyX = 400;
	let enemyY = 400;

	let health = $state(100);

	const enemySpriteWidth = 71;
	const enemySpriteHeight = 66;
	let enemyFrameX = 0;
	let enemyFrameY = 0;
	const enemySpriteColumns = 4;

	let gameFrame = 0;
	const staggerFrame = 6;

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

	let isPlayerFiring = false;

	function handleKeydown(event) {
		if (event.key in keysPressed) {
			keysPressed[event.key] = true;
		}
		if (event.key === 'f') isPlayerFiring = true;
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

	let isEnemyDead = false;
	const enemyAttackingSpriteColumns = 2;
	function moveEnemy(ctx) {
		const enemyVelocity = 2;
		const dx = playerX - enemyX;
		const dy = playerY - enemyY;
		const playerDistance = Math.sqrt(dx * dx + dy * dy);

		if (!isEnemyDead && !enemigoRecibiendoDisparo) {
			if (playerDistance <= playerRadius + 10) {
				// Atacando
				enemyFrameY = 1;
				if (gameFrame % (staggerFrame * 4) == 0) {
					enemyFrameX = (enemyFrameX + 1) % enemyAttackingSpriteColumns;
				}
			} else {
				// Caminando
				enemyFrameY = 0;
				enemyX += (dx / playerDistance) * enemyVelocity;
				enemyY += (dy / playerDistance) * enemyVelocity;
				if (gameFrame % staggerFrame == 0) {
					enemyFrameX = (enemyFrameX + 1) % enemySpriteColumns;
				}
			}
		}

		if (enemigoRecibiendoDisparo) {
			if (gameFrame % staggerFrame == 0) {
				enemyFrameX++;
				if (enemyFrameX >= enemySpriteColumns) {
					enemigoRecibiendoDisparo = false;
					enemyFrameX = 3;
					isEnemyDead = true;
				}
			}
		}

		gameFrame++;
	}

	let playerFrameX = 0;
	const playerFrameY = 0;
	const playerSpriteWidth = 204;
	const playerSpriteHeight = 216;
	const playerSpriteColumns = 3;

	function drawPlayer(ctx) {
		const playerImageSize = 128 * 2;

		ctx.drawImage(
			playerImage,
			playerFrameX * playerSpriteWidth,
			playerFrameY * playerSpriteHeight,
			playerSpriteWidth,
			playerSpriteHeight,
			canvasWidth / 2 - playerImageSize / 2,
			canvasHeight - playerImageSize,
			playerImageSize,
			playerImageSize
		);
	}

	let enemigoRecibiendoDisparo = false;

	function enemy(ctx, wallDistancePerRay) {
		const dx = enemyX - playerX;
		const dy = enemyY - playerY;
		const enemyDistance = Math.sqrt(dx * dx + dy * dy);
		const enemyAngle = Math.atan2(dy, dx);
		const enemySize = (squareSize * canvasHeight) / enemyDistance;
		const angleDifference = enemyAngle - angle;
		const enemySpriteX = (angleDifference / fov + 0.5) * canvasWidth;
		const enemySpriteY = middleY - enemySize / 2;

		if (isPlayerFiring) {
			if (gameFrame % staggerFrame == 0) {
				playerFrameX++;
				if (playerFrameX >= playerSpriteColumns) {
					isPlayerFiring = false;
					playerFrameX = 0;
				}
			}
			if (!isEnemyDead) {
				if (angleDifference < fov / 32) {
					enemigoRecibiendoDisparo = true;
					enemyFrameY = 2;
					enemyFrameX = 0;
				}
			}
		}

		const spriteColumn = (enemySpriteX + enemySize / 2) / (canvasWidth / rays);
		if (wallDistancePerRay[Math.floor(spriteColumn)] > enemyDistance) {
			ctx.drawImage(
				enemyImage,
				enemyFrameX * enemySpriteWidth,
				enemyFrameY * enemySpriteHeight,
				enemySpriteWidth,
				enemySpriteHeight,
				enemySpriteX,
				enemySpriteY,
				enemySize,
				enemySize
			);
		}
	}

	function ray(ctx) {
		const rayStep = fov / rays;
		const wallDistancePerRay = [];

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
			wallDistancePerRay[i] = correctedDistance;
			const columnHeight = (squareSize * canvasHeight) / correctedDistance;
			const columnWidth = canvasWidth / rays;
			const columnX = i * columnWidth;

			ctx.fillRect(columnX, middleY - columnHeight / 2, columnWidth, columnHeight);
		}
		enemy(ctx, wallDistancePerRay);
	}

	function gameLoop() {
		const ctx = canvas.getContext('2d');
		ctx.clearRect(0, 0, canvasWidth, canvasHeight);
		ctx.strokeRect(0, 0, canvasWidth, canvasHeight);

		movePlayer();
		moveEnemy(ctx);
		ray(ctx);
		drawMiniMap(ctx);
		drawPlayer(ctx);

		requestAnimationFrame(gameLoop);
	}

	onMount(() => {
		playerImage = new Image();
		playerImage.src = '/weapon.png';
		enemyImage = new Image();
		enemyImage.src = '/boss.png';
		gameLoop();
	});
</script>

<canvas bind:this={canvas} width={canvasWidth} height={canvasHeight}></canvas>
<svelte:window onkeydown={handleKeydown} onkeyup={handleKeyup} />
<p>Health {health}</p>
