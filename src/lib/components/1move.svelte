<script>
	import { onMount } from 'svelte';

	let canvas;
	const canvasWidth = 600;
	const canvasHeight = 600;
	let playerX = $state(canvasWidth / 2);
	let playerY = $state(canvasHeight / 2);
	const playerRadius = 50;
	let angle = $state(0);
	let cos = $derived(Math.cos(angle));
	let sin = $derived(Math.sin(angle));
	const velocity = 3;

	let keysPressed = $state({
		ArrowRight: false,
		ArrowLeft: false,
		ArrowUp: false,
		ArrowDown: false
	});

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
			playerX += cos * velocity;
			playerY += sin * velocity;
		}
		if (keysPressed.ArrowDown) {
			playerX -= cos * velocity;
			playerY -= sin * velocity;
		}
	}

	function gameLoop() {
		const ctx = canvas.getContext('2d');
		ctx.clearRect(0, 0, canvasWidth, canvasHeight);
		ctx.strokeRect(0, 0, canvasWidth, canvasHeight);

		ctx.beginPath();
		ctx.arc(playerX, playerY, playerRadius, 0, 2 * Math.PI);
		ctx.stroke();

		ctx.beginPath();
		ctx.moveTo(playerX, playerY);
		ctx.lineTo(playerX + playerRadius * 2 * cos, playerY + playerRadius * 2 * sin);
		ctx.stroke();

		movePlayer();

		requestAnimationFrame(gameLoop);
	}

	onMount(() => {
		gameLoop();
	});
</script>

<p>angle: {angle}</p>
<p>cos: {cos}</p>
<p>sin: {sin}</p>
<canvas bind:this={canvas} width={canvasWidth} height={canvasHeight}></canvas>
<svelte:window onkeydown={handleKeydown} onkeyup={handleKeyup} />
