<script>
	let playerX = $state(20);
	let playerY = $state(20);
	let angle = $state(0); // Ángulo de rotación en radianes
	const playerRadius = 10;

	let canvas;

	// Manejar eventos de teclado
	function handleKeydown(event) {
		if (event.key === 'ArrowLeft') {
			angle -= 0.1; // Rotar hacia la izquierda
		} else if (event.key === 'ArrowRight') {
			angle += 0.1; // Rotar hacia la derecha
		}
	}

	$effect(() => {
		const ctx = canvas.getContext('2d');
		ctx.clearRect(0, 0, canvas.width, canvas.height); // Limpiar el canvas

		// Dibujar el borde del canvas
		ctx.strokeRect(0, 0, canvas.width, canvas.height);

		// Dibujar el jugador
		ctx.beginPath();
		ctx.arc(playerX, playerY, playerRadius, 0, Math.PI * 2);
		ctx.stroke();

		// Dibujar la línea de dirección basada en el ángulo
		ctx.beginPath();
		ctx.moveTo(playerX, playerY);
		ctx.lineTo(
			playerX + Math.cos(angle) * playerRadius * 2,
			playerY + Math.sin(angle) * playerRadius * 2
		);
		ctx.stroke();
	});
</script>

<canvas bind:this={canvas}></canvas>
<svelte:window onkeydown={handleKeydown} />
