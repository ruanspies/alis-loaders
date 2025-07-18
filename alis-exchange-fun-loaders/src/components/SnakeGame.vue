<template>
    <div class="w-full max-w-sm md:max-w-md p-4">
        <div class="game-casing p-4">
            <!-- Game Header: Title and Score -->
            <div class="flex justify-between items-center mb-4 px-2">
                <h1 class="game-title text-lg">SNAKE</h1>
                <div class="score-display text-2xl">
                    <span>SCORE:</span>
                    <span id="score">0</span>
                </div>
            </div>

            <!-- Game Board Canvas -->
            <canvas id="game-board"></canvas>

            <!-- Game Over Screen -->
            <div id="game-over-screen" class="absolute inset-0 items-center justify-center overlay hidden">
                <div class="overlay-content p-8 rounded-lg text-center shadow-2xl">
                    <h2 class="text-3xl font-bold mb-2">GAME OVER</h2>
                    <p class="text-xl mb-6">FINAL SCORE: <span id="final-score">0</span></p>
                    <button id="restart-button" class="restart-button text-slate-800 font-bold py-3 px-6 rounded-lg text-lg">
                        PLAY AGAIN
                    </button>
                </div>
            </div>
        </div>
    </div>
</template>

<script setup lang="ts">
import { onMounted } from 'vue';

onMounted(() => {
    // --- DOM Elements ---
    const board = document.getElementById('game-board') as HTMLCanvasElement;
    const ctx = board.getContext('2d');
    const scoreElement = document.getElementById('score');
    const gameOverScreen = document.getElementById('game-over-screen');
    const finalScoreElement = document.getElementById('final-score');
    const restartButton = document.getElementById('restart-button');

    // --- Game Configuration ---
    const GRID_SIZE = 20; // Size of each snake segment and food item in pixels
    let boardSize: number; // Will be calculated based on container width
    let TILE_COUNT: number; // How many tiles fit on the board

    // --- Game State ---
    let snake: { x: number, y: number }[];
    let food: { x: number, y: number };
    let direction: { x: number, y: number };
    let score: number;
    let isGameOver: boolean;
    let gameLoopInterval: number;

    // --- Touch Controls State ---
    let touchStartX = 0;
    let touchStartY = 0;
    let touchEndX = 0;
    let touchEndY = 0;

    /**
     * Resizes the canvas to fit its container and calculates grid dimensions.
     */
    function setupCanvas() {
        const casing = document.querySelector('.game-casing') as HTMLElement;
        // Calculate the largest possible square that fits and is a multiple of GRID_SIZE
        const availableWidth = casing.clientWidth - (2 * 16 + 2 * 4); // Casing padding + board border
        boardSize = Math.floor(availableWidth / GRID_SIZE) * GRID_SIZE;
        TILE_COUNT = boardSize / GRID_SIZE;

        board.width = boardSize;
        board.height = boardSize;
    }

    /**
     * Initializes or resets the game to its starting state.
     */
    function initializeGame() {
        // Reset game state variables
        snake = [{ x: Math.floor(TILE_COUNT / 2), y: Math.floor(TILE_COUNT / 2) }];
        direction = { x: 0, y: 0 }; // Snake starts stationary
        score = 0;
        isGameOver = false;

        // Update UI
        if (scoreElement) scoreElement.textContent = '0';
        if (gameOverScreen) {
            gameOverScreen.classList.add('hidden');
            gameOverScreen.classList.remove('flex');
        }

        // Place the first piece of food
        placeFood();

        // Clear any existing game loop and start a new one
        if (gameLoopInterval) clearInterval(gameLoopInterval);
        gameLoopInterval = setInterval(gameLoop, 100); // Game speed: update every 100ms
    }

    /**
     * The main game loop, which updates the game state and redraws the board.
     */
    function gameLoop() {
        if (isGameOver) {
            clearInterval(gameLoopInterval);
            showGameOver();
            return;
        }

        update();
        draw();
    }

    /**
     * Updates the snake's position, checks for collisions, and handles food eating.
     */
    function update() {
        // Don't do anything if the game hasn't started (snake is not moving)
        if (direction.x === 0 && direction.y === 0) {
            return;
        }

        // Calculate new head position based on the current direction
        const head = { x: snake[0].x + direction.x, y: snake[0].y + direction.y };

        // Check for wall collision
        if (head.x < 0 || head.x >= TILE_COUNT || head.y < 0 || head.y >= TILE_COUNT) {
            isGameOver = true;
            return;
        }

        // Check for self collision (if the new head hits any part of the existing snake)
        for (let i = 0; i < snake.length; i++) {
            if (head.x === snake[i].x && head.y === snake[i].y) {
                isGameOver = true;
                return;
            }
        }

        // If no collisions, add the new head
        snake.unshift(head);

        // Check for food collision
        if (head.x === food.x && head.y === food.y) {
            // If food is eaten, increase score, place new food, and don't remove the tail (so the snake grows)
            score++;
            if (scoreElement) scoreElement.textContent = score.toString();
            placeFood();
        } else {
            // If no food is eaten, remove the tail segment to simulate movement
            snake.pop();
        }
    }

    /**
     * Draws all game elements onto the canvas.
     */
    function draw() {
        if (!ctx) return;
        // Clear the board with the background color
        ctx.fillStyle = '#c7d2fe'; // Same as #game-board background
        ctx.fillRect(0, 0, board.width, board.height);

        // Draw the snake
        ctx.fillStyle = '#1e293b'; // Dark slate color for the snake
        snake.forEach(segment => {
            ctx.fillRect(segment.x * GRID_SIZE, segment.y * GRID_SIZE, GRID_SIZE, GRID_SIZE);
        });

        // Draw the food
        ctx.fillStyle = '#ef4444'; // Red color for the food
        ctx.fillRect(food.x * GRID_SIZE, food.y * GRID_SIZE, GRID_SIZE, GRID_SIZE);
         // Add a little shine to the food
        ctx.fillStyle = 'rgba(255, 255, 255, 0.5)';
        ctx.fillRect(food.x * GRID_SIZE + 3, food.y * GRID_SIZE + 3, 6, 6);
    }

    /**
     * Finds a random empty spot on the board and places the food there.
     */
    function placeFood() {
        let newFoodPosition;
        do {
            newFoodPosition = {
                x: Math.floor(Math.random() * TILE_COUNT),
                y: Math.floor(Math.random() * TILE_COUNT)
            };
        } while (isPositionOnSnake(newFoodPosition)); // Ensure food doesn't spawn on the snake
        food = newFoodPosition;
    }

    /**
     * Checks if a given position is currently occupied by any part of the snake.
     * @param {object} position - The position to check, with x and y properties.
     * @returns {boolean} - True if the position is on the snake, false otherwise.
     */
    function isPositionOnSnake(position: { x: number, y: number }) {
        return snake.some(segment => segment.x === position.x && segment.y === position.y);
    }

    /**
     * Displays the game over screen.
     */
    function showGameOver() {
        if (finalScoreElement) finalScoreElement.textContent = score.toString();
        if (gameOverScreen) {
            gameOverScreen.classList.remove('hidden');
            gameOverScreen.classList.add('flex');
        }
    }

    /**
     * Handles keyboard input to change the snake's direction.
     * @param {KeyboardEvent} e - The keydown event object.
     */
    function handleKeyDown(e: KeyboardEvent) {
        switch (e.key) {
            case 'ArrowUp':
            case 'w':
                // Prevent the snake from reversing on itself
                if (direction.y === 0) direction = { x: 0, y: -1 };
                break;
            case 'ArrowDown':
            case 's':
                if (direction.y === 0) direction = { x: 0, y: 1 };
                break;
            case 'ArrowLeft':
            case 'a':
                if (direction.x === 0) direction = { x: -1, y: 0 };
                break;
            case 'ArrowRight':
            case 'd':
                if (direction.x === 0) direction = { x: 1, y: 0 };
                break;
        }
    }

    /**
     * Handles touch gestures for mobile control.
     */
    function handleGesture() {
        const deltaX = touchEndX - touchStartX;
        const deltaY = touchEndY - touchStartY;

        // Determine if the swipe was more horizontal or vertical
        if (Math.abs(deltaX) > Math.abs(deltaY)) {
            // Horizontal swipe
            if (deltaX > 0 && direction.x === 0) { // Right
                 direction = { x: 1, y: 0 };
            } else if (deltaX < 0 && direction.x === 0) { // Left
                 direction = { x: -1, y: 0 };
            }
        } else {
            // Vertical swipe
            if (deltaY > 0 && direction.y === 0) { // Down
                direction = { x: 0, y: 1 };
            } else if (deltaY < 0 && direction.y === 0) { // Up
                direction = { x: 0, y: -1 };
            }
        }
    }


    // --- Event Listeners ---
    document.addEventListener('keydown', handleKeyDown);
    if (restartButton) {
        restartButton.addEventListener('click', () => {
            setupCanvas();
            initializeGame();
        });
    }

    // Touch Controls
    board.addEventListener('touchstart', e => {
        touchStartX = e.changedTouches[0].screenX;
        touchStartY = e.changedTouches[0].screenY;
    }, { passive: true });

    board.addEventListener('touchend', e => {
        touchEndX = e.changedTouches[0].screenX;
        touchEndY = e.changedTouches[0].screenY;
        handleGesture();
    }, { passive: true });

    // Resize listener to make the game responsive
    window.addEventListener('resize', () => {
        setupCanvas();
        // We reset the game on resize to avoid weird state issues
        initializeGame();
    });

    // --- Initial Setup ---
    setupCanvas();
    initializeGame();
});
</script>

<style>
/* Custom styles for the game's retro aesthetic */
body {
    background-color: #334155; /* Slate-700 */
    font-family: 'Press Start 2P', cursive;
    touch-action: none; /* Disable double-tap to zoom on mobile */
}

/* The main container for the game, giving it a device-like frame */
.game-casing {
    background-color: #94a3b8; /* Slate-400 */
    border: 8px solid #1e293b; /* Slate-800 */
    border-radius: 20px;
    box-shadow: 0 10px 20px rgba(0,0,0,0.3), inset 0 0 15px rgba(0,0,0,0.4);
}

/* The game screen itself */
#game-board {
    background-color: #c7d2fe; /* Indigo-200, a classic LCD screen color */
    border: 4px solid #475569; /* Slate-600 */
    border-radius: 8px;
}

/* Styling for the score and title text */
.game-title, .score-display {
    color: #1e293b; /* Slate-800 */
    text-shadow: 1px 1px 0px #cbd5e1; /* Slate-300 */
}

/* Styling for the game over overlay */
.overlay {
    background-color: rgba(0, 0, 0, 0.75);
    animation: fadeIn 0.3s ease-out;
}

.overlay-content {
    background-color: #c7d2fe;
    border: 4px solid #475569;
    color: #1e293b;
}

.restart-button {
    background-color: #4ade80; /* Green-400 */
    border-bottom: 4px solid #22c55e; /* Green-500 */
    transition: all 0.1s ease;
}

.restart-button:active {
    transform: translateY(2px);
    border-bottom-width: 2px;
}

@keyframes fadeIn {
    from { opacity: 0; }
    to { opacity: 1; }
}
</style>
