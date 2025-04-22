<script>
	let canvas;

	const canvasWidth = 600;
	const canvasHeight = 300;
	let playerX = $state(canvasWidth / 2);
	let playerY = $state(canvasHeight / 2);
	const playerRadius = 20;
	let angle = $state(0);
	const velocity = 4;
	let animationId;

	// Track which keys are currently pressed
	const keysPressed = $state({
		ArrowUp: false,
		ArrowDown: false,
		ArrowLeft: false,
		ArrowRight: false
	});

	// Handle keydown events
	function handleKeydown(event) {
		if (['ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight'].includes(event.key)) {
			keysPressed[event.key] = true;
			// Prevent default to stop scrolling when using arrow keys
			event.preventDefault();
		}
	}

	// Handle keyup events
	function handleKeyup(event) {
		if (['ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight'].includes(event.key)) {
			keysPressed[event.key] = false;
		}
	}

	// Game loop for smooth movement
	function gameLoop() {
		// Rotate based on left/right keys
		if (keysPressed.ArrowLeft) {
			angle -= 0.05; // Reduced rotation speed for smoother turning
		}
		if (keysPressed.ArrowRight) {
			angle += 0.05;
		}

		// Move based on up/down keys
		if (keysPressed.ArrowUp) {
			playerX += velocity * Math.cos(angle);
			playerY += velocity * Math.sin(angle);
		}
		if (keysPressed.ArrowDown) {
			playerX -= velocity * Math.cos(angle);
			playerY -= velocity * Math.sin(angle);
		}

		// Add boundary checking
		playerX = Math.max(playerRadius, Math.min(canvasWidth - playerRadius, playerX));
		playerY = Math.max(playerRadius, Math.min(canvasHeight - playerRadius, playerY));

		// Draw everything
		const ctx = canvas.getContext('2d');
		ctx.clearRect(0, 0, canvasWidth, canvasHeight);

		// Draw the border of the canvas
		ctx.strokeRect(0, 0, canvasWidth, canvasHeight);

		// Draw the player
		ctx.beginPath();
		ctx.arc(playerX, playerY, playerRadius, 0, 2 * Math.PI);
		ctx.stroke();

		// Draw the direction indicator
		ctx.beginPath();
		ctx.moveTo(playerX, playerY);
		ctx.lineTo(
			playerX + Math.cos(angle) * playerRadius * 2,
			playerY + Math.sin(angle) * playerRadius * 2
		);
		ctx.stroke();

		// Request next frame
		animationId = requestAnimationFrame(gameLoop);
	}

	$effect(() => {
		if (!canvas) return;

		// Start the game loop
		animationId = requestAnimationFrame(gameLoop);

		// Return a cleanup function to cancel the animation frame when the component is unmounted
		return () => cancelAnimationFrame(animationId);
	});
</script>

<canvas bind:this={canvas} width={canvasWidth} height={canvasHeight}></canvas>
<svelte:window onkeydown={handleKeydown} onkeyup={handleKeyup} />
