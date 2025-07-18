<template>
    <div class="game-container w-full h-full flex flex-col items-center justify-center p-4">
        <div class="absolute top-5 text-center">
            <h1 class="text-2xl font-bold text-gray-700">Alis Logo Reconstruction</h1>
            <p v-if="gameState === 'playing' || gameState === 'finished'" class="text-4xl font-mono text-red-500 mt-2">{{ formattedTime }}</p>
        </div>

        <svg :width="gameArea.size" :height="gameArea.size" class="bg-white rounded-2xl shadow-lg">
            <!-- Render filled/locked dots -->
            <circle
                v-for="p in placeholders.filter(p => p.isFilled)"
                :key="'locked-' + p.id"
                :cx="p.x"
                :cy="p.y"
                :r="dotRadius"
                class="locked-dot"
            />

            <g v-if="gameState === 'playing'">
                <!-- Render unfilled placeholders -->
                <circle
                    v-for="p in placeholders.filter(p => !p.isFilled)"
                    :key="'placeholder-' + p.id"
                    :cx="p.x"
                    :cy="p.y"
                    :r="dotRadius"
                    :class="{
                        'placeholder-dot': currentTarget && p.id !== currentTarget.id,
                        'target-dot': currentTarget && p.id === currentTarget.id
                    }"
                />
                <!-- Player-controlled dot -->
                <circle
                    :cx="playerDot.x"
                    :cy="playerDot.y"
                    :r="dotRadius"
                    class="player-dot"
                />
            </g>
        </svg>

        <div class="absolute inset-0 flex items-center justify-center bg-black bg-opacity-50" v-if="gameState !== 'playing'">
            <div class="text-center text-white p-8 bg-gray-800 rounded-2xl shadow-2xl fade-in-up">
                <div v-if="gameState === 'instructions'">
                    <h2 class="text-3xl font-bold mb-4">Welcome!</h2>
                    <p class="mb-2">A new rule has been added!</p>
                    <p class="mb-6">Move the red dot to the <span class="font-bold text-red-400">pulsating target</span> to fill the logo.</p>
                    <button @click="startGame" class="px-8 py-3 bg-red-500 text-white font-bold rounded-lg hover:bg-red-600 transition-colors transform hover:scale-105">
                        Start Game
                    </button>
                </div>
                <div v-if="gameState === 'finished'">
                    <h2 class="text-3xl font-bold mb-2">Complete!</h2>
                    <p class="text-xl mb-4">Your time:</p>
                    <p class="text-5xl font-mono text-red-400 mb-6">{{ formattedTime }}</p>
                    <button @click="startGame" class="px-8 py-3 bg-red-500 text-white font-bold rounded-lg hover:bg-red-600 transition-colors transform hover:scale-105">
                        Play Again
                    </button>
                </div>
            </div>
        </div>
    </div>
</template>

<script setup lang="ts">
import { ref, reactive, onMounted, computed, watch } from 'vue';
import { useEventListener, useWindowSize } from '@vueuse/core';
import { gsap } from 'gsap';

// --- STATE MANAGEMENT ---

const gameState = ref('instructions');
const timer = ref(0);
let timerInterval: number | null = null;
const playerDot = reactive({ x: 50, y: 50 });
const { width: windowWidth, height: windowHeight } = useWindowSize();

const logoCoords = [
    { x: 30, y: 60 }, { x: 30, y: 80 }, { x: 30, y: 100 },
    { x: 50, y: 20 }, { x: 50, y: 40 }, { x: 50, y: 60 }, { x: 50, y: 80 }, { x: 50, y: 100 },
    { x: 70, y: 20 }, { x: 70, y: 40 }, { x: 70, y: 60 },
];

const placeholders = ref<any[]>([]);
const targetOrder = ref<any[]>([]);
const currentTargetIndex = ref(0);

// --- COMPUTED PROPERTIES ---

const gameArea = computed(() => {
    const padding = 60;
    const availableWidth = windowWidth.value - padding;
    const availableHeight = windowHeight.value - padding;
    const size = Math.min(availableWidth, availableHeight, 600);
    return { size };
});

const scaleToPixels = (coord: number, axis: 'x' | 'y') => {
    const { size } = gameArea.value;
    const gridMax = axis === 'y' ? 120 : 100;
    const scaleFactor = 0.8;
    const offset = (1 - scaleFactor) / 2;
    return (coord / gridMax) * size * scaleFactor + size * offset;
};

const dotRadius = computed(() => gameArea.value.size * 0.045);
const collisionThreshold = computed(() => dotRadius.value * 0.8);

const currentTarget = computed(() => {
    if (gameState.value !== 'playing' || currentTargetIndex.value >= targetOrder.value.length) {
        return null;
    }
    const targetId = targetOrder.value[currentTargetIndex.value];
    return placeholders.value.find(p => p.id === targetId);
});

// --- GAME LOGIC ---

const shuffleArray = (array: any[]) => {
    for (let i = array.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [array[i], array[j]] = [array[j], array[i]];
    }
    return array;
};

const initializeGame = () => {
    placeholders.value = logoCoords.map((coord, index) => ({
        id: index,
        x: scaleToPixels(coord.x, 'x'),
        y: scaleToPixels(coord.y, 'y'),
        isFilled: false,
    }));

    targetOrder.value = shuffleArray(placeholders.value.map(p => p.id));
    currentTargetIndex.value = 0;

    if (timerInterval) clearInterval(timerInterval);
    timer.value = 0;

    spawnNewPlayerDot();
};

const startTimer = () => {
    if (timerInterval) clearInterval(timerInterval);
    const startTime = Date.now();
    timerInterval = setInterval(() => {
        timer.value = Date.now() - startTime;
    }, 10);
};

const spawnNewPlayerDot = () => {
    const { size } = gameArea.value;
    const r = dotRadius.value;
    let x, y;
    const edge = Math.floor(Math.random() * 4);

    switch (edge) {
        case 0: x = Math.random() * (size - 2 * r) + r; y = r; break;
        case 1: x = size - r; y = Math.random() * (size - 2 * r) + r; break;
        case 2: x = Math.random() * (size - 2 * r) + r; y = size - r; break;
        case 3: x = r; y = Math.random() * (size - 2 * r) + r; break;
    }

    gsap.killTweensOf(playerDot);
    gsap.set(playerDot, { x, y });
};

const checkCollision = () => {
    if (!currentTarget.value) return;

    const p = currentTarget.value;
    const distance = Math.sqrt(
        Math.pow(playerDot.x - p.x, 2) + Math.pow(playerDot.y - p.y, 2)
    );

    if (distance < collisionThreshold.value) {
        lockDotInPlace(p);
    }
};

const lockDotInPlace = (placeholder: any) => {
    placeholder.isFilled = true;
    currentTargetIndex.value++;

    gsap.to(playerDot, {
        x: placeholder.x,
        y: placeholder.y,
        duration: 0.1,
        ease: 'power2.out',
        onComplete: () => {
            if (currentTargetIndex.value >= targetOrder.value.length) {
                gameState.value = 'finished';
                if (timerInterval) clearInterval(timerInterval);
            } else {
                spawnNewPlayerDot();
            }
        }
    });
};

const movePlayer = (direction: 'up' | 'down' | 'left' | 'right') => {
    if (gameState.value !== 'playing') return;

    if (timer.value === 0 && !timerInterval) {
        startTimer();
    }

    const moveAmount = gameArea.value.size * 0.05;
    let targetX = playerDot.x;
    let targetY = playerDot.y;

    if (direction === 'up') targetY -= moveAmount;
    if (direction === 'down') targetY += moveAmount;
    if (direction === 'left') targetX -= moveAmount;
    if (direction === 'right') targetX += moveAmount;

    const { size } = gameArea.value;
    const r = dotRadius.value;
    targetX = Math.max(r, Math.min(size - r, targetX));
    targetY = Math.max(r, Math.min(size - r, targetY));

    gsap.to(playerDot, {
        x: targetX,
        y: targetY,
        duration: 0.2,
        ease: 'power1.out',
        onUpdate: checkCollision
    });
};

const startGame = () => {
    initializeGame();
    gameState.value = 'playing';
};

useEventListener(document, 'keydown', (e) => {
    if (e.key === 'ArrowUp') movePlayer('up');
    if (e.key === 'ArrowDown') movePlayer('down');
    if (e.key === 'ArrowLeft') movePlayer('left');
    if (e.key === 'ArrowRight') movePlayer('right');
});

watch(gameArea, () => {
    initializeGame();
    if (gameState.value !== 'instructions') {
        gameState.value = 'playing';
    }
}, { deep: true });

onMounted(initializeGame);

const formattedTime = computed(() => {
    const ms = timer.value;
    const minutes = Math.floor(ms / 60000);
    const seconds = Math.floor((ms % 60000) / 1000);
    const milliseconds = Math.floor((ms % 1000) / 10);
    return `${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}.${String(milliseconds).padStart(2, '0')}`;
});

</script>

<style>
/* Custom font and base styles */
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;700&display=swap');
body {
    font-family: 'Inter', sans-serif;
    overflow: hidden;
    background-color: #f3f4f6; /* A light gray background */
}

/* Styling for the game elements for a clean look */
.game-container {
    touch-action: none; /* Disable touch actions like pinch-zoom */
}

.placeholder-dot {
    fill: #e0e0e0; /* Light gray for placeholders */
    transition: fill 0.3s ease;
}

.player-dot {
    fill: #ef4444; /* A vibrant red */
    stroke: #dc2626;
    stroke-width: 2px;
    filter: drop-shadow(0 4px 6px rgba(239, 68, 68, 0.4));
}

.locked-dot {
    fill: #ef4444; /* The same vibrant red */
    transition: r 0.3s ease, opacity 0.3s ease;
}

/* Style for the active target placeholder */
.target-dot {
    fill: #fef2f2; /* A very light red to look like a hole */
    stroke: #ef4444;
    stroke-width: 2px;
    /* MODIFIED: Reverted speed to original value */
    animation: pulse 1.5s infinite;
}

/* Animation for the win message */
.fade-in-up {
    animation: fadeInUp 0.5s ease-out forwards;
}

/* MODIFIED: Pulse animation has subtle distance change */
@keyframes pulse {
    0% {
        transform: scale(1);
        opacity: 1;
    }
    50% {
        transform: scale(1.03); /* Kept the reduced scale */
        opacity: 0.85;        /* Kept the reduced opacity change */
    }
    100% {
        transform: scale(1);
        opacity: 1;
    }
}

@keyframes fadeInUp {
    from {
        opacity: 0;
        transform: translateY(20px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}
</style>
