<script lang="ts">
	import { onMount } from 'svelte';
	import { writable } from 'svelte/store';

	type Rectangle = {
		id: string;
		x: number;
		y: number;
		width: number;
		height: number;
		color: string;
		strokeColor: string;
		strokeWidth: number;
	};

	type Connection = {
		from: Rectangle;
		to: Rectangle;
	};

	let rectangles: Rectangle[] = [];
	let connections: Connection[] = [];
	let canvas: HTMLCanvasElement;
	let ctx: CanvasRenderingContext2D | null;

	const selectedRectangle = writable<Rectangle | null>(null);
	let isDragging = false;
	let offsetX = 0;
	let offsetY = 0;
	let arrowStartRect: Rectangle | null = null;
	let isCreatingArrow = false;

	function addRectangle() {
		const newRect: Rectangle = {
			id: `rect${rectangles.length + 1}`,
			x: 100,
			y: 100,
			width: 100,
			height: 50,
			color: '#fff',
			strokeColor: '#000000',
			strokeWidth: 2
		};
		rectangles = [...rectangles, newRect];
		renderCanvas();
	}

	function renderCanvas() {
		if (!ctx) return;
		ctx.clearRect(0, 0, canvas.width, canvas.height);

		for (const rect of rectangles) {
			ctx.fillStyle = rect.color;
			ctx.fillRect(rect.x, rect.y, rect.width, rect.height);
			ctx.lineWidth = rect.strokeWidth;
			ctx.strokeStyle = rect.strokeColor;
			ctx.strokeRect(rect.x, rect.y, rect.width, rect.height);
		}

		for (const { from, to } of connections) {
			const [startX, startY] = getBorderPoint(from, to.x, to.y);
			const [endX, endY] = getBorderPoint(to, from.x, from.y);
			drawArrow(startX, startY, endX, endY);
		}
	}

	function drawArrow(x1: number, y1: number, x2: number, y2: number) {
		if (!ctx) return;
		ctx.strokeStyle = '#000';
		ctx.lineWidth = 2;

		const headlen = 10;
		const angle = Math.atan2(y2 - y1, x2 - x1);

		ctx.beginPath();
		ctx.moveTo(x1, y1);
		ctx.lineTo(x2, y2);
		ctx.lineTo(
			x2 - headlen * Math.cos(angle - Math.PI / 6),
			y2 - headlen * Math.sin(angle - Math.PI / 6)
		);
		ctx.moveTo(x2, y2);
		ctx.lineTo(
			x2 - headlen * Math.cos(angle + Math.PI / 6),
			y2 - headlen * Math.sin(angle + Math.PI / 6)
		);
		ctx.stroke();
	}

	function getBorderPoint(rect: Rectangle, targetX: number, targetY: number): [number, number] {
		const cx = rect.x + rect.width / 2;
		const cy = rect.y + rect.height / 2;
		const dx = targetX - cx;
		const dy = targetY - cy;
		const aspectRatio = rect.width / rect.height;

		let borderX = cx,
			borderY = cy;
		if (Math.abs(dx / dy) > aspectRatio) {
			borderX = dx > 0 ? rect.x + rect.width : rect.x;
			borderY = cy + (dy * (borderX - cx)) / dx;
		} else {
			borderY = dy > 0 ? rect.y + rect.height : rect.y;
			borderX = cx + (dx * (borderY - cy)) / dy;
		}

		return [borderX, borderY];
	}

	function updateRectangle(property: string, value: string | number) {
		selectedRectangle.update((rect) => {
			if (rect) {
				(rect as any)[property] = value;
				renderCanvas();
			}
			return rect;
		});
	}

	function onMouseDown(event: MouseEvent) {
		const { offsetX: mouseX, offsetY: mouseY } = event;
		const clickedRect = rectangles.find(
			(rect) =>
				mouseX >= rect.x &&
				mouseX <= rect.x + rect.width &&
				mouseY >= rect.y &&
				mouseY <= rect.y + rect.height
		);

		if (clickedRect) {
			selectedRectangle.set(clickedRect);
			if (isCreatingArrow) {
				arrowStartRect = clickedRect;
			} else {
				isDragging = true;
				offsetX = mouseX - clickedRect.x;
				offsetY = mouseY - clickedRect.y;
			}
		}
	}

	function onMouseMove(event: MouseEvent) {
		if (isDragging) {
			const { offsetX: mouseX, offsetY: mouseY } = event;
			selectedRectangle.update((rect) => {
				if (rect) {
					rect.x = mouseX - offsetX;
					rect.y = mouseY - offsetY;
					renderCanvas();
				}
				return rect;
			});
		} else if (isCreatingArrow && arrowStartRect) {
			renderCanvas();
			const [startX, startY] = getBorderPoint(arrowStartRect, event.offsetX, event.offsetY);
			drawArrow(startX, startY, event.offsetX, event.offsetY);
		}
	}

	function onMouseUp(event: MouseEvent) {
		isDragging = false;

		if (isCreatingArrow && arrowStartRect) {
			const { offsetX: mouseX, offsetY: mouseY } = event;
			const targetRect = rectangles.find(
				(rect) =>
					mouseX >= rect.x &&
					mouseX <= rect.x + rect.width &&
					mouseY >= rect.y &&
					mouseY <= rect.y + rect.height &&
					rect !== arrowStartRect
			);

			if (targetRect) {
				connections = [...connections, { from: arrowStartRect, to: targetRect }];
			}
			isCreatingArrow = false;
			arrowStartRect = null;
			renderCanvas();
		}
	}

	function startArrowCreation() {
		isCreatingArrow = true;
	}

	onMount(() => {
		canvas = document.querySelector('#canvas') as HTMLCanvasElement;
		ctx = canvas.getContext('2d');
		renderCanvas();
	});
</script>

<div class="flex">
	<!-- Toolbar for adding shapes -->
	<div class="toolbar w-32 bg-gray-200 p-4">
		<button on:click={addRectangle} class="mb-2 rounded bg-blue-500 p-2 text-white">
			Add Rectangle
		</button>
		<button on:click={startArrowCreation} class="mt-2 rounded bg-red-500 p-2 text-white">
			Create Arrow
		</button>
	</div>

	<!-- Canvas Area -->
	<div class="relative h-screen w-full">
		<canvas
			id="canvas"
			width="800"
			height="600"
			on:mousedown={onMouseDown}
			on:mousemove={onMouseMove}
			on:mouseup={onMouseUp}
			class="border border-gray-300"
		></canvas>
	</div>
</div>

<style>
	.toolbar {
		display: flex;
		flex-direction: column;
		gap: 1rem;
	}
</style>


