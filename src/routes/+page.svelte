<script>
	let playerX = 20;
	let playerY = 20;
	const playerRadius = 10;

	let canvas;

	$effect(() => {
		const ctx = canvas.getContext('2d');
		//ctx.clearReact(0, 0, canvas.width, canvas.height)

		ctx.strokeRect(0, 0, canvas.width, canvas.height);

		ctx.beginPath();
		ctx.arc(playerX, playerY, playerRadius, 0, Math.PI * 2);
		ctx.stroke();

		ctx.beginPath();
		ctx.moveTo(playerX, playerY);
		ctx.lineTo(playerX + playerRadius * 2, playerY);
		ctx.stroke();
	});
</script>

<canvas bind:this={canvas}></canvas>
