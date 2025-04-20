<script>
	let canvas;

	$effect(() => {
		const ctx = canvas.getContext('2d');
		//ctx.clearReact(0, 0, canvas.width, canvas.height)

		ctx.strokeRect(0, 0, canvas.width, canvas.height);
		ctx.beginPath();
		ctx.arc(20, 20, 10, 0, Math.PI * 2);
		ctx.stroke();
	});
</script>

<canvas bind:this={canvas}></canvas>
