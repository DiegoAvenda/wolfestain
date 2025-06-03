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

	const enemyenemySpriteWidth = 71;
	const enemyenemySpriteHeight = 66;
	let enemyenemyFrameX = 0;
	const enemyenemyFrameY = 0;
	const enemySpriteColumns = 4;

	let gameFrame = 0;
	const staggerFrame = 5;

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

	function handleKeydown(event) {
		if (event.key in keysPressed) {
			keysPressed[event.key] = true;
		}
		if (event.key === 'f') {
			firePlayerWeapon();
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

	function moveEnemy(ctx) {
		const enemyVelocity = 2;
		const dx = playerX - enemyX;
		const dy = playerY - enemyY;
		const playerDistance = Math.sqrt(dx * dx + dy * dy);

		if (playerRadius + 10 < playerDistance) {
			enemyX += (dx / playerDistance) * enemyVelocity;
			enemyY += (dy / playerDistance) * enemyVelocity;
			if (gameFrame % staggerFrame == 0) {
				enemyenemyFrameX = (enemyenemyFrameX + 1) % enemySpriteColumns;
			}
		}
		gameFrame++;
	}

	const playerenemySpriteWidth = 204;
	const playerenemySpriteHeight = 216;
	let playerenemySpriteX = 0;
	const playerenemySpriteY = 0;

	let isFiring = false;
	const playerSpriteColumns = 3;

	function firePlayerWeapon() {
		isFiring = true;
		if (playerenemySpriteX <= playerSpriteColumns) {
			playerenemySpriteX++;
		}
	}

	function drawPlayer(ctx) {
		playerImage = new Image();
		playerImage.src = '/weapon.png';
		ctx.drawImage(
			playerImage,
			playerenemySpriteX * playerenemySpriteWidth,
			playerenemySpriteY,
			playerenemySpriteWidth,
			playerenemySpriteHeight,
			canvasWidth / 2 - playerenemySpriteWidth / 2,
			canvasHeight - playerenemySpriteHeight,
			playerenemySpriteWidth,
			playerenemySpriteHeight
		);
	}

	function enemy(ctx, wallDistancePerRay) {
		enemyImage = new Image();
		enemyImage.src = '/boss.png';
		const dx = enemyX - playerX;
		const dy = enemyY - playerY;
		const enemyDistance = Math.sqrt(dx * dx + dy * dy);
		const enemyAngle = Math.atan2(dy, dx);
		const enemySize = (squareSize * canvasHeight) / enemyDistance;
		const angleDifference = enemyAngle - angle;
		const enemyenemySpriteX = (angleDifference / fov + 0.5) * canvasWidth;
		const enemyenemySpriteY = middleY - enemySize / 2;

		const spriteColumn = (enemyenemySpriteX + enemySize / 2) / (canvasWidth / rays);
		if (wallDistancePerRay[Math.floor(spriteColumn)] > enemyDistance) {
			ctx.drawImage(
				enemyImage,
				enemyenemyFrameX * enemyenemySpriteWidth,
				enemyenemyFrameY * enemyenemySpriteHeight,
				enemyenemySpriteWidth,
				enemyenemySpriteHeight,
				enemyenemySpriteX,
				enemyenemySpriteY,
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
		gameLoop();
	});
</script>

<canvas bind:this={canvas} width={canvasWidth} height={canvasHeight}></canvas>
<svelte:window onkeydown={handleKeydown} onkeyup={handleKeyup} />
