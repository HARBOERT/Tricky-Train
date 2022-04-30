<script lang="ts">
	import { browser } from '$app/env';

	import { range } from 'functional-utilities';

	import { sample } from 'lodash-es';

	const x_width = 10;
	const y_width = 10;

	interface element_type {
		type: 'body' | 'head' | 'food' | 'empty' | 'cornerUp' | 'cornerDown';
		rotation: rotation;
		metadata: string;
	}

	interface position {
		x: number;
		y: number;
	}

	type rotation = 0 | 90 | 180 | 270;

	interface tail {
		position: position;
		rotation: rotation;
		type: 'straight' | 'up' | 'down';
	}

	let score = 1
	let highScore = undefined

	let snakePosition: position = {
		x: 2,
		y: 5
	};
	let lastSnakeRotation: rotation = 0;
	let snake_rotation: rotation = 0;

	let snakeLength = 0; 

	let snakeTail: tail[] = [];
	let grid: element_type[][];

	const possibleTrainLooks = [
		'GmbH-und-CoKGanhängerdone',
		'HWanhängerDone',
		'LJanhängerDone',
		'MoneyanhängerDone'
	];
	const trainLooks: typeof possibleTrainLooks[number][] = [];

	clear_board();

	function getRandomPosition(): position {
		const position = {
			x: Math.floor(Math.random() * x_width),
			y: Math.floor(Math.random() * y_width)
		};
		if (grid[position.y][position.x].type !== 'empty') {
			return getRandomPosition();
		}
		return position;
	}

	let foodPosition: position = getRandomPosition();

	function clear_board() {
		grid = range(0, x_width).map(() =>
			range(0, y_width).map(() => ({
				type: 'empty',
				rotation: 0,
				metadata: ''
			}))
		);
	}

	function draw_board() {
		clear_board();
		grid[snakePosition.y][snakePosition.x] = {
			type: 'head',
			rotation: snake_rotation,
			metadata: ''
		};
		for (const [index, tail] of snakeTail.entries()) {
			grid[tail.position.y][tail.position.x] = {
				type: tail.type === 'straight' ? 'body' : tail.type === 'up' ? 'cornerUp' : 'cornerDown',
				rotation: tail.rotation,
				metadata: trainLooks[index]
			};
		}
		grid[foodPosition.y][foodPosition.x] = {
			type: 'food',
			rotation: 0,
			metadata: ''
		};
	}

	let gameIsRunning = false;
	function toggleGame() {
		if (gameIsRunning == true) {
			gameIsRunning = false;
		} else {
			gameIsRunning = true;
		}
	}

	function updateTail() {
		const tailDirectionOffset = absoluteDegrees(lastSnakeRotation - snake_rotation);
		if (tailDirectionOffset === 0) {
			snakeTail.push({
				position: {
					x: snakePosition.x,
					y: snakePosition.y
				},
				rotation: snake_rotation,
				type: 'straight'
			});
		} else if (tailDirectionOffset === 90) {
			snakeTail.push({
				position: {
					x: snakePosition.x,
					y: snakePosition.y
				},
				rotation: snake_rotation,
				type: 'down'
			});
		} else if (tailDirectionOffset === 270) {
			snakeTail.push({
				position: {
					x: snakePosition.x,
					y: snakePosition.y
				},
				rotation: snake_rotation,
				type: 'up'
			});
		}
		if (snakeTail.length > snakeLength) {
			snakeTail.shift();
		}

		lastSnakeRotation = snake_rotation;
	}

	function absoluteDegrees(degrees: number): number {
		return (degrees + 360) % 360;
	}

	function collectFood() {
		if (snakePosition.x === foodPosition.x && snakePosition.y === foodPosition.y) {
			snakeLength += 1;
			foodPosition = getRandomPosition();
			const chosenLook = sample(possibleTrainLooks);
			if (chosenLook !== undefined) {
				trainLooks.push(chosenLook);
			}
			score += 1
		}
	}

	function restart() {
		snakePosition = {
			x: 2,
			y: 5
		};
		snakeLength = 0;
		snake_rotation = 0;
		snakeTail = [];
		foodPosition = getRandomPosition();
		draw_board();
		gameIsRunning = false;
		score = 0;
	}

	function moveSnake() {
		if (snake_rotation === 0) {
			if (snakePosition.x >= x_width - 1) {
				restart();
			}
			snakePosition.x += 1;
		} else if (snake_rotation === 90) {
			if (snakePosition.y >= y_width - 1) {
				restart();
			}
			snakePosition.y += 1;
		} else if (snake_rotation === 180) {
			if (snakePosition.x <= 0) {
				restart();
			}
			snakePosition.x -= 1;
		} else if (snake_rotation === 270) {
			if (snakePosition.y <= 0) {
				restart();
			}
			snakePosition.y -= 1;
		}
	}

	function checkForWin() {
		for (const row of grid) {
			for (const element of row) {
				if (element.type == 'empty') {
					return false;
				}
			}
		}
		return true;
	}

	function colliteSnake() {
		for (const tail of snakeTail) {
			if (snakePosition.x === tail.position.x && snakePosition.y === tail.position.y) {
				restart();
			}
		}
	}

	let hasWon = false;
	const moveList: rotation[] = [];
	function update() {
		if (gameIsRunning == true) {
			turnSnake();
			updateTail();
			moveSnake();
			collectFood();
			colliteSnake();
			draw_board();
			if (checkForWin()) {
				hasWon = true;
				gameIsRunning = false;
			}
		}
	}
	function keypress(event: KeyboardEvent) {
		const key = event.key;
		if (moveList.length < 3) {
			if ((key === 'w' || key === 'ArrowUp') && snake_rotation !== 90) {
				moveList.push(270);
			} else if ((key === 's' || key === 'ArrowDown') && snake_rotation !== 270) {
				moveList.push(90);
			} else if ((key === 'd' || key === 'ArrowRight') && snake_rotation !== 180) {
				moveList.push(0);
			} else if ((key === 'a' || key === 'ArrowLeft') && snake_rotation !== 0) {
				moveList.push(180);
			} else if (key === ' ') {
				toggleGame();
			}
		}
	}

	function rotationToTransform(rotation: rotation): string {
		if (rotation === 0) {
			return 'rotate(0deg)';
		} else if (rotation === 90) {
			return 'rotate(90deg)';
		} else if (rotation === 180) {
			return 'scaleX(-1)';
		} else if (rotation === 270) {
			return 'rotate(270deg)';
		}
		throw new Error('[rotationToTransform] this should not happen');
	}

	function turnSnake() {
		const nextRotation = moveList.shift();
		if (nextRotation !== undefined) {
			lastSnakeRotation = snake_rotation;
			snake_rotation = nextRotation;
		}
	}



	if (browser) {
		setInterval(update, 350);
	}

	draw_board();
</script>

<svelte:window on:keydown={keypress} />

<div class="main">
	<div class="score">Score: {score}     </div>
	<div class="game-board">
		{#each grid as row}
			<div class="row">
				{#each row as element}
					<div
						class="element"
						class:snakeHead={element.type == 'head'}
						style:transform={rotationToTransform(element.rotation)}
					>
						{#if element.type == 'head'}
							<img src="/zugHead.png" alt="0_0" />
						{:else if element.type == 'body'}
							<img src={`/${element.metadata}.png`} alt="===" />
						{:else if element.type == 'food'}
							<img src="/FragezeichenAnhängerDone.png" alt="===" />
						{:else if element.type == 'cornerUp' && element.rotation == 180}
							<img class="cornerUpFlipped" src={`/${element.metadata}.png`} alt="_|" />
						{:else if element.type == 'cornerDown' && element.rotation == 180}
							<img class="cornerDownFlipped" src={`/${element.metadata}.png`} alt="_|" />
						{:else if element.type == 'cornerUp'}
							<img class="cornerUp" src={`/${element.metadata}.png`} alt="_|" />
						{:else if element.type == 'cornerDown'}
							<img class="cornerDown" src={`/${element.metadata}.png`} alt="|_" />
						{/if}
					</div>
				{/each}
			</div>
		{/each}
	</div>
	{#if hasWon}
		<div class="youwintext">You win!</div>
	{/if}
</div>

<style>
	@import url('https://fonts.googleapis.com/css2?family=Press+Start+2P&display=swap');
	:global(body) {
		margin: 0px;
	}

	.game-board {
		display: flex;
		flex-direction: column;
		width: 100vmin;
		height: 100vmin;
		background-image: url("Fingselvemadebackground.png");
		justify-content: space-evenly;
		background-size: 100%;

	}

	.row {
		display: flex;
		height: 100vmin;
		justify-content: space-evenly;
	}
	.element {
		width: 100%;
		display: flex;
		justify-content: center;
		align-items: flex-end;
		background-image: url('UntitlesSchiene1.png');
		image-rendering: pixelated;
		background-size: 100%;
	}

	img {
		width: 100%;
	}

	.main {
		position: relative;
		display: flex;
		justify-content: center;
		background-color: black;
	}

	.youwintext {
		position: absolute;
		left: 50%;
		top: 50%;
		transform: translate(-50%, -50%);
		font-size: 150px;
		font-family: 'Press Start 2P', cursive;
		text-align: center;
		color: rgb(32, 32, 49);
		filter: drop-shadow(0px -2px 15px rgb(255, 0, 242));
	}

	.cornerUp {
		transform: rotate(-45deg);
	}

	.cornerDown {
		transform: rotate(45deg);
	}

	.cornerUpFlipped {
		transform: rotate(45deg);
	}

	.cornerDownFlipped {
		transform: rotate(-45deg);
	}

	.score {
		font-size: 96px;
		padding: 20px;
		height: fit-content;
		color: #FEAE34;
		background-color: #262B44;
		border-radius: 10px;
		border: 5px solid #6F5F5F;
		font-family: 'Press Start 2P', cursive;
	}
</style>