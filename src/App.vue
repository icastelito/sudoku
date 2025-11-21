<script setup lang="ts">
import { ref, onMounted, computed } from "vue";

type Cell = {
	value: number;
	isFixed: boolean;
};

type Difficulty = "tutorial" | "facil" | "medio" | "dificil" | "expert";

type HistoryEntry = {
	id: string;
	difficulty: Difficulty;
	time: number;
	hintsUsed: number;
	date: string;
	puzzleHash: string;
};

const board = ref<Cell[][]>([]);
const solution = ref<number[][]>([]);
const showModal = ref(true);
const showGiveUpModal = ref(false);
const showVictoryModal = ref(false);
const showHistoryModal = ref(false);
const hasGivenUp = ref(false);
const selectedDifficulty = ref<Difficulty>("medio");
const gameStarted = ref(false);
const isPaused = ref(false);
const elapsedTime = ref(0);
const hintsRemaining = ref(0);
const gameHistory = ref<HistoryEntry[]>([]);
const currentPuzzleHash = ref("");
let timerInterval: number | null = null;

const difficultySettings = {
	tutorial: 10, // 81 total - 78 removidos = 3 para adivinhar
	facil: 30,
	medio: 40,
	dificil: 50,
	expert: 64, // 81 total - 64 removidos = 17 preenchidos
};

const hintsSettings = {
	tutorial: 7,
	facil: 5,
	medio: 3,
	dificil: 1,
	expert: 0,
};

const formattedTime = computed(() => {
	const minutes = Math.floor(elapsedTime.value / 60);
	const seconds = elapsedTime.value % 60;
	return `${minutes.toString().padStart(2, "0")}:${seconds.toString().padStart(2, "0")}`;
});

// Funções de histórico
function loadHistory() {
	const stored = localStorage.getItem("sudoku-history");
	if (stored) {
		gameHistory.value = JSON.parse(stored);
	}
}

function saveHistory(entry: HistoryEntry) {
	gameHistory.value.unshift(entry);
	if (gameHistory.value.length > 50) {
		gameHistory.value = gameHistory.value.slice(0, 50);
	}
	localStorage.setItem("sudoku-history", JSON.stringify(gameHistory.value));
}

function hashPuzzle(puzzle: number[][]): string {
	return puzzle.map((row) => row.join("")).join("");
}

function isPuzzleUsed(hash: string): boolean {
	return gameHistory.value.some((entry) => entry.puzzleHash === hash);
}

function formatHistoryTime(seconds: number): string {
	const minutes = Math.floor(seconds / 60);
	const secs = seconds % 60;
	return `${minutes}:${secs.toString().padStart(2, "0")}`;
}

function toggleHistory() {
	showHistoryModal.value = !showHistoryModal.value;
	if (showHistoryModal.value && !isPaused.value && gameStarted.value) {
		// Pausar o jogo ao abrir o histórico
		if (timerInterval) {
			clearInterval(timerInterval);
			timerInterval = null;
		}
	} else if (!showHistoryModal.value && !isPaused.value && gameStarted.value) {
		// Retomar o jogo ao fechar o histórico
		timerInterval = setInterval(() => {
			elapsedTime.value++;
		}, 1000);
	}
}

// Gerar um Sudoku válido
function generateSudoku(difficulty: Difficulty): { puzzle: number[][]; solution: number[][] } {
	const grid: number[][] = Array(9)
		.fill(0)
		.map(() => Array(9).fill(0));

	// Preencher a grade com um Sudoku válido
	fillGrid(grid);

	// Guardar a solução completa
	const completeSolution = grid.map((row) => [...row]);

	// Remover alguns números para criar o puzzle
	removeNumbers(grid, difficultySettings[difficulty]);

	return { puzzle: grid, solution: completeSolution };
}

function isValid(grid: number[][], row: number, col: number, num: number): boolean {
	// Verificar linha
	for (let x = 0; x < 9; x++) {
		if (grid[row]?.[x] === num) return false;
	}

	// Verificar coluna
	for (let x = 0; x < 9; x++) {
		if (grid[x]?.[col] === num) return false;
	}

	// Verificar bloco 3x3
	const startRow = row - (row % 3);
	const startCol = col - (col % 3);
	for (let i = 0; i < 3; i++) {
		for (let j = 0; j < 3; j++) {
			if (grid[i + startRow]?.[j + startCol] === num) return false;
		}
	}

	return true;
}

function fillGrid(grid: number[][]): boolean {
	for (let row = 0; row < 9; row++) {
		for (let col = 0; col < 9; col++) {
			if (grid[row]?.[col] === 0) {
				const numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9].sort(() => Math.random() - 0.5);
				for (const num of numbers) {
					if (isValid(grid, row, col, num)) {
						grid[row]![col] = num;
						if (fillGrid(grid)) return true;
						grid[row]![col] = 0;
					}
				}
				return false;
			}
		}
	}
	return true;
}

function removeNumbers(grid: number[][], count: number) {
	let removed = 0;
	while (removed < count) {
		const row = Math.floor(Math.random() * 9);
		const col = Math.floor(Math.random() * 9);
		if (grid[row]?.[col] !== 0) {
			grid[row]![col] = 0;
			removed++;
		}
	}
}

function initBoard(difficulty: Difficulty) {
	let puzzle: number[][];
	let completeSolution: number[][];
	let hash: string;
	let attempts = 0;
	const maxAttempts = 10;

	// Tentar gerar um puzzle não repetido
	do {
		const generated = generateSudoku(difficulty);
		puzzle = generated.puzzle;
		completeSolution = generated.solution;
		hash = hashPuzzle(puzzle);
		attempts++;
	} while (isPuzzleUsed(hash) && attempts < maxAttempts);

	currentPuzzleHash.value = hash;
	solution.value = completeSolution;
	board.value = puzzle.map((row) =>
		row.map((value) => ({
			value,
			isFixed: value !== 0,
		}))
	);
}

function updateCell(row: number, col: number, event: Event) {
	const target = event.target as HTMLInputElement;
	const value = parseInt(target.value) || 0;

	if (board.value[row]?.[col] && !board.value[row][col].isFixed && value >= 0 && value <= 9) {
		board.value[row][col].value = value;
		checkVictory();
	}
}

function checkVictory() {
	// Verificar se todas as células estão preenchidas
	for (let row = 0; row < 9; row++) {
		for (let col = 0; col < 9; col++) {
			if (board.value[row]?.[col]?.value === 0) return;
		}
	}

	// Verificar se a solução está correta
	for (let row = 0; row < 9; row++) {
		for (let col = 0; col < 9; col++) {
			if (board.value[row]?.[col]?.value !== solution.value[row]?.[col]) return;
		}
	}

	// Jogo completo e correto!
	showVictoryModal.value = true;
	if (timerInterval) {
		clearInterval(timerInterval);
		timerInterval = null;
	}

	// Salvar no histórico
	const entry: HistoryEntry = {
		id: Date.now().toString(),
		difficulty: selectedDifficulty.value,
		time: elapsedTime.value,
		hintsUsed: hintsSettings[selectedDifficulty.value] - hintsRemaining.value,
		date: new Date().toLocaleString("pt-BR"),
		puzzleHash: currentPuzzleHash.value,
	};
	saveHistory(entry);
}

function startGame() {
	showModal.value = false;
	gameStarted.value = true;
	hintsRemaining.value = hintsSettings[selectedDifficulty.value];
	initBoard(selectedDifficulty.value);
	startTimer();
}

function startTimer() {
	elapsedTime.value = 0;
	if (timerInterval) clearInterval(timerInterval);
	timerInterval = setInterval(() => {
		elapsedTime.value++;
	}, 1000);
}

function togglePause() {
	isPaused.value = !isPaused.value;
	if (isPaused.value) {
		// Pausar
		if (timerInterval) {
			clearInterval(timerInterval);
			timerInterval = null;
		}
	} else {
		// Retomar
		timerInterval = setInterval(() => {
			elapsedTime.value++;
		}, 1000);
	}
}

function useHint() {
	if (hintsRemaining.value <= 0) return;

	// Encontrar células vazias
	const emptyCells: { row: number; col: number }[] = [];
	for (let row = 0; row < 9; row++) {
		for (let col = 0; col < 9; col++) {
			const cell = board.value[row]?.[col];
			if (cell && !cell.isFixed && cell.value === 0) {
				emptyCells.push({ row, col });
			}
		}
	}

	if (emptyCells.length === 0) return;

	// Escolher uma célula aleatória
	const randomIndex = Math.floor(Math.random() * emptyCells.length);
	const emptyCell = emptyCells[randomIndex];
	if (!emptyCell) return;
	const { row, col } = emptyCell;

	// Preencher com a solução
	const targetCell = board.value[row]?.[col];
	const solutionValue = solution.value[row]?.[col];
	if (targetCell && solutionValue !== undefined) {
		targetCell.value = solutionValue;
		targetCell.isFixed = true;
	}

	hintsRemaining.value--;
}

function askGiveUp() {
	showGiveUpModal.value = true;
	if (timerInterval) {
		clearInterval(timerInterval);
		timerInterval = null;
	}
}

function cancelGiveUp() {
	showGiveUpModal.value = false;
	if (!isPaused.value) {
		timerInterval = setInterval(() => {
			elapsedTime.value++;
		}, 1000);
	}
}

function confirmGiveUp() {
	showGiveUpModal.value = false;
	hasGivenUp.value = true;
	// Revelar toda a solução
	for (let row = 0; row < 9; row++) {
		for (let col = 0; col < 9; col++) {
			const cell = board.value[row]?.[col];
			const solutionValue = solution.value[row]?.[col];
			if (cell && solutionValue !== undefined) {
				cell.value = solutionValue;
				cell.isFixed = true;
			}
		}
	}
}

function newGame() {
	window.location.reload();
}

onMounted(() => {
	// Não inicia o jogo automaticamente, espera o modal
	loadHistory();

	// Inicializar anúncios do Google AdSense quando o jogo iniciar
	setTimeout(() => {
		try {
			(window as any).adsbygoogle = (window as any).adsbygoogle || [];
			const ads = document.querySelectorAll(".adsbygoogle");
			ads.forEach(() => {
				(window as any).adsbygoogle.push({});
			});
		} catch (e) {
			console.error("Erro ao carregar anúncios:", e);
		}
	}, 1000);
});
</script>

<template>
	<!-- Modal de Histórico -->
	<div v-if="showHistoryModal" class="modal-overlay" @click.self="toggleHistory">
		<div class="modal history-modal">
			<div class="history-header">
				<h2><i class="fa-solid fa-clock-rotate-left"></i> Histórico de Jogos</h2>
				<button class="close-history" @click="toggleHistory">
					<i class="fa-solid fa-xmark"></i>
				</button>
			</div>
			<div class="history-content">
				<div v-if="gameHistory.length === 0" class="no-history">
					<i class="fa-solid fa-inbox"></i>
					<p>Nenhum jogo completado ainda</p>
				</div>
				<div v-else class="history-list">
					<div v-for="entry in gameHistory" :key="entry.id" class="history-item">
						<div class="history-difficulty">
							<i class="fa-solid fa-trophy"></i>
							<span class="difficulty-badge" :class="entry.difficulty">{{ entry.difficulty }}</span>
						</div>
						<div class="history-stats">
							<span><i class="fa-regular fa-clock"></i> {{ formatHistoryTime(entry.time) }}</span>
							<span><i class="fa-solid fa-lightbulb"></i> {{ entry.hintsUsed }} dicas</span>
						</div>
						<div class="history-date"><i class="fa-solid fa-calendar"></i> {{ entry.date }}</div>
					</div>
				</div>
			</div>
		</div>
	</div>

	<!-- Modal de Vitória -->
	<div v-if="showVictoryModal" class="modal-overlay">
		<div class="modal victory-modal">
			<i class="fa-solid fa-trophy victory-icon"></i>
			<h2>Parabéns! Você venceu!</h2>
			<p class="victory-message">
				<i class="fa-solid fa-star"></i> Excelente trabalho!<br />
				<i class="fa-regular fa-clock"></i> Tempo: {{ formattedTime }}<br />
				<i class="fa-solid fa-lightbulb"></i> Dicas usadas:
				{{ hintsSettings[selectedDifficulty] - hintsRemaining }}
			</p>
			<button class="new-game-victory-button" @click="newGame">
				<i class="fa-solid fa-rotate"></i>
				Jogar Novamente
			</button>
		</div>
	</div>

	<!-- Modal de Confirmação de Desistência -->
	<div v-if="showGiveUpModal" class="modal-overlay">
		<div class="modal give-up-modal">
			<i class="fa-solid fa-triangle-exclamation warning-icon"></i>
			<h2>Tem certeza que deseja desistir?</h2>
			<p class="encouragement">
				<i class="fa-solid fa-star"></i> Você consegue! Não desista agora!<br />
				<i class="fa-solid fa-dumbbell"></i> Cada desafio te torna mais forte!<br />
				<i class="fa-solid fa-brain"></i> Use as dicas disponíveis se precisar de ajuda!
			</p>
			<div class="give-up-buttons">
				<button class="cancel-button" @click="cancelGiveUp">
					<i class="fa-solid fa-arrow-left"></i>
					Continuar Jogando
				</button>
				<button class="confirm-button" @click="confirmGiveUp">
					<i class="fa-solid fa-eye"></i>
					Ver Solução
				</button>
			</div>
		</div>
	</div>

	<!-- Modal de Instruções -->
	<div v-if="showModal" class="modal-overlay">
		<div class="modal">
			<h2>Bem-vindo ao Sudoku!</h2>
			<div class="instructions">
				<p><strong>Como jogar:</strong></p>
				<ul>
					<li>Preencha o tabuleiro 9×9 com números de 1 a 9</li>
					<li>Cada linha deve conter os números de 1 a 9 sem repetição</li>
					<li>Cada coluna deve conter os números de 1 a 9 sem repetição</li>
					<li>Cada bloco 3×3 deve conter os números de 1 a 9 sem repetição</li>
				</ul>
			</div>

			<div class="difficulty-selector">
				<p><strong>Selecione a dificuldade:</strong></p>
				<div class="difficulty-buttons">
					<button
						@click="selectedDifficulty = 'tutorial'"
						:class="{ active: selectedDifficulty === 'tutorial' }"
					>
						Tutorial
					</button>
					<button @click="selectedDifficulty = 'facil'" :class="{ active: selectedDifficulty === 'facil' }">
						Fácil
					</button>
					<button @click="selectedDifficulty = 'medio'" :class="{ active: selectedDifficulty === 'medio' }">
						Médio
					</button>
					<button
						@click="selectedDifficulty = 'dificil'"
						:class="{ active: selectedDifficulty === 'dificil' }"
					>
						Difícil
					</button>
					<button @click="selectedDifficulty = 'expert'" :class="{ active: selectedDifficulty === 'expert' }">
						Expert
					</button>
				</div>
			</div>

			<button class="start-button" @click="startGame">Iniciar Jogo</button>
		</div>
	</div>

	<div class="game-wrapper" v-if="gameStarted">
		<!-- Anúncio Esquerdo -->
		<div class="ad-left">
			<ins
				class="adsbygoogle"
				style="display: block"
				data-ad-client="ca-pub-1396718158024453"
				data-ad-slot="1234567890"
				data-ad-format="vertical"
			></ins>
		</div>

		<div class="sudoku-container">
			<div class="game-header">
				<h1>Sudoku</h1>
				<div class="controls">
					<button class="history-button" @click="toggleHistory">
						<i class="fa-solid fa-clock-rotate-left"></i>
					</button>
					<button class="hint-button" @click="useHint" :disabled="hintsRemaining === 0">
						<i class="fa-solid fa-lightbulb"></i>
						<span>{{ hintsRemaining }}</span>
					</button>
					<div class="timer">
						<i class="fa-regular fa-clock"></i>
						<span>{{ formattedTime }}</span>
						<button class="pause-button" @click="togglePause">
							<i :class="isPaused ? 'fa-solid fa-play' : 'fa-solid fa-pause'"></i>
						</button>
					</div>
				</div>
			</div>

			<div v-if="!isPaused" class="sudoku-grid">
				<div v-for="(row, rowIndex) in board" :key="rowIndex" class="sudoku-row">
					<div
						v-for="(cell, colIndex) in row"
						:key="colIndex"
						class="sudoku-cell"
						:class="{
							'cell-fixed': cell.isFixed,
							'border-right': (colIndex + 1) % 3 === 0 && colIndex < 8,
							'border-bottom': (rowIndex + 1) % 3 === 0 && rowIndex < 8,
						}"
					>
						<input
							type="number"
							min="1"
							max="9"
							:value="cell.value || ''"
							:disabled="cell.isFixed"
							@input="updateCell(rowIndex, colIndex, $event)"
						/>
					</div>
				</div>
			</div>
			<div v-if="isPaused" class="paused-overlay">
				<i class="fa-solid fa-pause pause-icon"></i>
				<p>Jogo Pausado</p>
			</div>

			<div class="action-buttons">
				<button v-if="!hasGivenUp" class="give-up-button" @click="askGiveUp">
					<i class="fa-solid fa-flag"></i>
					Desistir
				</button>
				<button v-if="hasGivenUp" class="new-game-button" @click="newGame">
					<i class="fa-solid fa-rotate"></i>
					Novo Jogo
				</button>
			</div>
		</div>

		<!-- Anúncio Direito -->
		<div class="ad-right">
			<ins
				class="adsbygoogle"
				style="display: block"
				data-ad-client="ca-pub-1396718158024453"
				data-ad-slot="0987654321"
				data-ad-format="vertical"
			></ins>
		</div>
	</div>
</template>

<style scoped>
.game-wrapper {
	display: flex;
	align-items: center;
	justify-content: center;
	gap: 20px;
	width: 100%;
	max-width: 100vw;
	padding: 20px;
	box-sizing: border-box;
}

.ad-left,
.ad-right {
	min-width: 160px;
	width: 160px;
	height: 600px;
	display: flex;
	align-items: center;
	justify-content: center;
	background: rgba(255, 255, 255, 0.5);
	border-radius: 8px;
	flex-shrink: 0;
}

.sudoku-container {
	display: flex;
	flex-direction: column;
	align-items: center;
	gap: 12px;
	padding: 2dvh 2dvw;
	background: rgba(255, 255, 255, 0.9);
	border-radius: 12px;
	box-shadow: 0 20px 60px rgba(0, 0, 0, 0.15), 0 0 0 1px rgba(0, 0, 0, 0.05);
	backdrop-filter: blur(10px);
	max-width: 100dvw;
	box-sizing: border-box;
}

h1 {
	margin: 0;
	font-size: 1.8em;
	font-weight: 700;
	color: #c73e3a;
	letter-spacing: 2px;
	text-shadow: 2px 2px 4px rgba(199, 62, 58, 0.1);
}

.sudoku-grid {
	display: inline-block;
	border: 3px solid #2d2d2d;
	background: #2d2d2d;
	box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2), 0 0 0 1px rgba(255, 255, 255, 0.1) inset;
	border-radius: 4px;
	overflow: hidden;
	max-width: min(90vw, 450px);
	max-height: calc(90dvh - 250px);
}

.sudoku-row {
	display: flex;
}

.sudoku-cell {
	width: calc(min(90vw, 500px) / 9 - 1px);
	height: calc(min(90vw, 500px) / 9 - 1px);
	max-width: 50px;
	max-height: 50px;
	border: 0.5px solid #d4b5a0;
	position: relative;
	transition: all 0.2s ease;
}

.sudoku-cell.border-right {
	border-right: 5px solid #000000;
}

.sudoku-cell.border-bottom {
	border-bottom: 5px solid #000000;
}
.sudoku-cell input {
	width: 100%;
	height: 100%;
	border: none;
	text-align: center;
	font-size: 1.4em;
	font-weight: 500;
	background: #fffbf7;
	color: #c73e3a;
	outline: none;
	font-family: "Noto Sans JP", sans-serif;
	transition: all 0.3s ease;
}

.sudoku-cell input::-webkit-inner-spin-button,
.sudoku-cell input::-webkit-outer-spin-button {
	-webkit-appearance: none;
	margin: 0;
}

.sudoku-cell input[type="number"] {
	-moz-appearance: textfield;
	appearance: textfield;
}

.sudoku-cell.cell-fixed input {
	background: #f0ebe5;
	color: #2d2d2d;
	font-weight: 700;
	cursor: not-allowed;
}

.sudoku-cell input:not(:disabled):focus {
	background: #fff;
	box-shadow: 0 0 0 2px #c73e3a inset;
	transform: scale(1.05);
	z-index: 10;
}

.sudoku-cell input:not(:disabled):hover {
	background: #fff;
}

.info {
	color: #888;
	font-size: 0.9em;
	margin: 10px 0 0 0;
}

/* Modal */
.modal-overlay {
	position: fixed;
	top: 0;
	left: 0;
	width: 100%;
	height: 100%;
	background: rgba(45, 45, 45, 0.85);
	display: flex;
	align-items: center;
	justify-content: center;
	z-index: 1000;
	backdrop-filter: blur(5px);
	padding: 20px;
	box-sizing: border-box;
}

.modal {
	background: linear-gradient(135deg, #fffbf7 0%, #fef5ed 100%);
	padding: 35px 25px;
	border-radius: 24px;
	max-width: 500px;
	width: 90%;
	max-height: 90vh;
	overflow-y: auto;
	box-shadow: 0 20px 60px rgba(0, 0, 0, 0.15), 0 0 0 1px rgba(199, 62, 58, 0.1);
	color: #2d2d2d;
	border: 3px solid rgba(199, 62, 58, 0.1);
	box-sizing: border-box;
}

.modal h2 {
	margin-top: 0;
	color: #c73e3a;
	text-align: center;
	font-size: 2.2em;
	font-weight: 700;
	letter-spacing: 1px;
	text-shadow: 1px 1px 2px rgba(199, 62, 58, 0.1);
}

.instructions {
	margin: 20px 0;
	text-align: left;
}

.instructions p {
	margin: 10px 0;
}

.instructions ul {
	margin: 10px 0;
	padding-left: 20px;
}

.instructions li {
	margin: 8px 0;
	line-height: 1.6;
}

.difficulty-selector {
	margin: 30px 0;
	text-align: center;
}

.difficulty-selector p {
	margin-bottom: 15px;
}

.difficulty-buttons {
	display: flex;
	gap: 10px;
	justify-content: center;
	flex-wrap: wrap;
}

.difficulty-buttons button {
	padding: 12px 20px;
	border: 2px solid #c73e3a;
	background: rgba(255, 255, 255, 0.9);
	color: #c73e3a;
	border-radius: 50px;
	cursor: pointer;
	font-weight: 600;
	transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
	box-shadow: 0 2px 8px rgba(199, 62, 58, 0.1);
	font-size: 0.95em;
	box-sizing: border-box;
}

.difficulty-buttons button:hover {
	background: #c73e3a;
	color: white;
	transform: translateY(-3px);
	box-shadow: 0 4px 15px rgba(199, 62, 58, 0.3);
}

.difficulty-buttons button.active {
	background: #c73e3a;
	color: white;
	box-shadow: 0 4px 15px rgba(199, 62, 58, 0.3);
}

.start-button {
	width: 100%;
	padding: 18px;
	background: linear-gradient(135deg, #c73e3a 0%, #e85d5d 100%);
	color: white;
	border: none;
	border-radius: 50px;
	font-size: 1.3em;
	font-weight: 700;
	cursor: pointer;
	margin-top: 25px;
	transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
	box-shadow: 0 6px 20px rgba(199, 62, 58, 0.3);
	text-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
}

.start-button:hover {
	transform: translateY(-3px);
	box-shadow: 0 8px 30px rgba(199, 62, 58, 0.4);
	background: linear-gradient(135deg, #a32e2a 0%, #c73e3a 100%);
}

/* Timer */
.game-header {
	display: flex;
	flex-direction: column;
	align-items: center;
	width: 100%;
	gap: 12px;
	margin-bottom: 10px;
}

.game-header h1 {
	margin: 0;
}

.controls {
	display: flex;
	align-items: center;
	gap: 15px;
	flex-wrap: wrap;
	justify-content: center;
	width: 100%;
	max-width: 100%;
	box-sizing: border-box;
	padding: 0 10px;
}

.hint-button {
	background: linear-gradient(135deg, #f6d365 0%, #fda085 100%);
	border: none;
	color: #fff;
	padding: 10px 20px;
	border-radius: 50px;
	cursor: pointer;
	display: flex;
	align-items: center;
	gap: 10px;
	font-size: 0.95em;
	font-weight: 600;
	transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
	box-shadow: 0 4px 15px rgba(253, 160, 133, 0.3);
	text-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
	box-sizing: border-box;
	max-width: 100%;
}

.hint-button:hover:not(:disabled) {
	transform: translateY(-3px);
	box-shadow: 0 6px 20px rgba(253, 160, 133, 0.4);
}

.hint-button:disabled {
	opacity: 0.4;
	cursor: not-allowed;
	filter: grayscale(50%);
}

.timer {
	font-size: 1.2em;
	font-weight: 600;
	color: #2d2d2d;
	background: rgba(255, 255, 255, 0.9);
	padding: 12px 24px;
	border-radius: 50px;
	display: flex;
	align-items: center;
	gap: 12px;
	box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
	border: 2px solid rgba(199, 62, 58, 0.2);
	box-sizing: border-box;
	max-width: 100%;
}

.pause-button {
	background: #c73e3a;
	border: none;
	color: white;
	width: 42px;
	height: 42px;
	border-radius: 50%;
	cursor: pointer;
	display: flex;
	align-items: center;
	justify-content: center;
	font-size: 1em;
	transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
	box-shadow: 0 3px 10px rgba(199, 62, 58, 0.3);
}

.pause-button:hover {
	background: #a32e2a;
	transform: scale(1.1);
	box-shadow: 0 5px 15px rgba(199, 62, 58, 0.4);
}

.paused-overlay {
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: center;
	min-height: 490px;
	width: 490px;
	background: rgba(255, 255, 255, 0.95);
	border-radius: 12px;
	color: #c73e3a;
	border: 3px solid rgba(199, 62, 58, 0.2);
	box-shadow: 0 10px 30px rgba(0, 0, 0, 0.1);
}

.pause-icon {
	font-size: 5em;
	margin-bottom: 20px;
	opacity: 0.7;
	color: #c73e3a;
}

.paused-overlay p {
	font-size: 1.6em;
	font-weight: 700;
	color: #2d2d2d;
}

/* Action Buttons */
.action-buttons {
	display: flex;
	justify-content: center;
	flex-wrap: wrap;
	width: 100%;
	max-width: 100%;
	box-sizing: border-box;
}

.give-up-button,
.new-game-button {
	padding: 14px 28px;
	border: none;
	border-radius: 50px;
	cursor: pointer;
	display: flex;
	align-items: center;
	justify-content: center;
	gap: 10px;
	font-size: 1em;
	font-weight: 600;
	transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
	text-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
	box-sizing: border-box;
	max-width: 100%;
}

.give-up-button {
	background: linear-gradient(135deg, #6c757d 0%, #5a6268 100%);
	color: white;
	box-shadow: 0 4px 15px rgba(108, 117, 125, 0.3);
}

.give-up-button:hover {
	background: linear-gradient(135deg, #5a6268 0%, #495057 100%);
	transform: translateY(-3px);
	box-shadow: 0 6px 20px rgba(108, 117, 125, 0.4);
}

.new-game-button {
	background: linear-gradient(135deg, #28a745 0%, #34ce57 100%);
	color: white;
	box-shadow: 0 4px 15px rgba(40, 167, 69, 0.3);
}

.new-game-button:hover {
	background: linear-gradient(135deg, #218838 0%, #28a745 100%);
	transform: translateY(-3px);
	box-shadow: 0 6px 20px rgba(40, 167, 69, 0.4);
}

/* Give Up Modal */
.give-up-modal {
	text-align: center;
	max-width: 450px;
}

.warning-icon {
	font-size: 4em;
	color: #ffc107;
	margin-bottom: 15px;
}

.encouragement {
	line-height: 2;
	margin: 20px 0;
	font-size: 1.1em;
	color: #555;
}

.give-up-buttons {
	display: flex;
	gap: 10px;
	margin-top: 30px;
}

.cancel-button,
.confirm-button {
	flex: 1;
	padding: 15px;
	border: none;
	border-radius: 8px;
	cursor: pointer;
	display: flex;
	align-items: center;
	justify-content: center;
	gap: 8px;
	font-size: 1em;
	font-weight: 600;
	transition: all 0.3s;
}

.cancel-button {
	background: #28a745;
	color: white;
}

.cancel-button:hover {
	background: #218838;
	transform: translateY(-2px);
}

.confirm-button {
	background: #6c757d;
	color: white;
}

.confirm-button:hover {
	background: #5a6268;
	transform: translateY(-2px);
}

/* Victory Modal */
.victory-modal {
	text-align: center;
	max-width: 450px;
}

.victory-icon {
	font-size: 5em;
	color: #ffd700;
	margin-bottom: 20px;
	animation: bounce 0.5s ease-in-out infinite alternate;
}

@keyframes bounce {
	from {
		transform: translateY(0);
	}
	to {
		transform: translateY(-10px);
	}
}

.victory-message {
	line-height: 2.5;
	margin: 25px 0;
	font-size: 1.2em;
	color: #555;
}

.victory-message i {
	margin-right: 8px;
	color: #667eea;
}

.new-game-victory-button {
	width: 100%;
	padding: 15px;
	background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
	color: white;
	border: none;
	border-radius: 8px;
	font-size: 1.2em;
	font-weight: 700;
	cursor: pointer;
	margin-top: 20px;
	transition: transform 0.2s;
	display: flex;
	align-items: center;
	justify-content: center;
	gap: 10px;
}

.new-game-victory-button:hover {
	transform: translateY(-2px);
	box-shadow: 0 5px 20px rgba(102, 126, 234, 0.4);
}

/* History Button */
.history-button {
	background: linear-gradient(135deg, #6c757d 0%, #5a6268 100%);
	border: none;
	color: white;
	width: 48px;
	height: 48px;
	min-width: 48px;
	border-radius: 50%;
	cursor: pointer;
	display: flex;
	align-items: center;
	justify-content: center;
	font-size: 1.2em;
	transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
	box-shadow: 0 4px 15px rgba(108, 117, 125, 0.3);
	flex-shrink: 0;
}

.history-button:hover {
	background: linear-gradient(135deg, #5a6268 0%, #495057 100%);
	transform: translateY(-3px);
	box-shadow: 0 6px 20px rgba(108, 117, 125, 0.4);
}

/* History Modal */
.history-modal {
	max-width: 600px;
	width: 90%;
	max-height: 80vh;
	display: flex;
	flex-direction: column;
}

.history-header {
	display: flex;
	justify-content: space-between;
	align-items: center;
	margin-bottom: 20px;
	border-bottom: 2px solid #e0e0e0;
	padding-bottom: 15px;
}

.history-header h2 {
	margin: 0;
	display: flex;
	align-items: center;
	gap: 10px;
	color: #667eea;
}

.close-history {
	background: none;
	border: none;
	font-size: 1.5em;
	cursor: pointer;
	color: #666;
	width: 40px;
	height: 40px;
	border-radius: 50%;
	display: flex;
	align-items: center;
	justify-content: center;
	transition: all 0.3s;
}

.close-history:hover {
	background: #f0f0f0;
	color: #333;
}

.history-content {
	overflow-y: auto;
	max-height: 60vh;
}

.no-history {
	text-align: center;
	padding: 40px;
	color: #999;
}

.no-history i {
	font-size: 4em;
	margin-bottom: 20px;
	opacity: 0.5;
}

.history-list {
	display: flex;
	flex-direction: column;
	gap: 15px;
}

.history-item {
	background: #f8f9fa;
	padding: 15px;
	border-radius: 10px;
	border-left: 4px solid #667eea;
	display: flex;
	flex-direction: column;
	gap: 10px;
	transition: all 0.3s;
}

.history-item:hover {
	background: #e9ecef;
	transform: translateX(5px);
}

.history-difficulty {
	display: flex;
	align-items: center;
	gap: 10px;
	font-weight: bold;
}

.difficulty-badge {
	padding: 4px 12px;
	border-radius: 12px;
	font-size: 0.85em;
	text-transform: uppercase;
	color: white;
}

.difficulty-badge.tutorial {
	background: #17a2b8;
}

.difficulty-badge.facil {
	background: #28a745;
}

.difficulty-badge.medio {
	background: #ffc107;
	color: #333;
}

.difficulty-badge.dificil {
	background: #fd7e14;
}

.difficulty-badge.expert {
	background: #dc3545;
}

.history-stats {
	display: flex;
	gap: 20px;
	color: #555;
	font-size: 0.95em;
}

.history-stats i {
	margin-right: 5px;
	color: #667eea;
}

.history-date {
	color: #888;
	font-size: 0.85em;
}

.history-date i {
	margin-right: 5px;
}

/* Responsividade */

/* Desktop Large (1920px+) */
@media (min-width: 1920px) {
	.sudoku-container {
		gap: 40px;
		margin: 10px;
	}

	h1 {
		font-size: 3.2em !important;
	}

	.sudoku-cell {
		width: 60px;
		height: 60px;
	}

	.sudoku-cell input {
		font-size: 1.8em;
	}

	.paused-overlay {
		min-height: 545px;
		width: 545px;
	}

	.timer {
		font-size: 1.6em;
	}

	.modal {
		max-width: 650px;
		padding: 50px 40px;
	}

	.modal h2 {
		font-size: 2.5em;
	}
}

/* Desktop (1200px - 1919px) */
@media (min-width: 1200px) and (max-width: 1919px) {
	.sudoku-container {
		padding: 45px 25px;
		gap: 35px;
	}

	h1 {
		font-size: 3em !important;
	}

	.sudoku-cell {
		width: 56px;
		height: 56px;
	}

	.sudoku-cell input {
		font-size: 1.7em;
	}

	.paused-overlay {
		min-height: 510px;
		width: 510px;
	}
}

/* Laptop (992px - 1199px) */
@media (min-width: 992px) and (max-width: 1199px) {
	.sudoku-container {
		padding: 40px 20px;
		gap: 30px;
	}

	h1 {
		font-size: 2.6em !important;
	}

	.sudoku-cell {
		width: 52px;
		height: 52px;
	}

	.sudoku-cell input {
		font-size: 1.55em;
	}

	.paused-overlay {
		min-height: 475px;
		width: 475px;
	}
}

/* Tablet Landscape (768px - 991px) */
@media (min-width: 768px) and (max-width: 991px) {
	.sudoku-container {
		padding: 35px 18px;
		gap: 25px;
	}

	h1 {
		font-size: 2.3em !important;
	}

	.sudoku-cell {
		width: 48px;
		height: 48px;
	}

	.sudoku-cell input {
		font-size: 1.45em;
	}

	.paused-overlay {
		min-height: 440px;
		width: 440px;
		max-width: 90vw;
	}

	.controls {
		gap: 12px;
	}

	.hint-button {
		padding: 11px 20px;
		font-size: 1.05em;
	}

	.history-button {
		width: 46px;
		height: 46px;
	}

	.timer {
		font-size: 1.3em;
		padding: 11px 20px;
	}

	.modal {
		padding: 35px 28px;
	}

	.modal h2 {
		font-size: 2em;
	}

	.history-modal {
		max-width: 550px;
	}
}

/* Tablet Portrait (576px - 767px) */
@media (min-width: 576px) and (max-width: 767px) {
	.sudoku-container {
		padding: 30px 15px;
		gap: 20px;
	}

	h1 {
		font-size: 2em !important;
	}

	.sudoku-cell {
		width: 44px;
		height: 44px;
	}

	.sudoku-cell input {
		font-size: 1.35em;
	}

	.paused-overlay {
		min-height: 400px;
		width: 400px;
		max-width: 90vw;
	}

	.controls {
		gap: 10px;
		flex-wrap: wrap;
	}

	.hint-button {
		padding: 10px 18px;
		font-size: 1em;
	}

	.history-button {
		width: 44px;
		height: 44px;
	}

	.timer {
		font-size: 1.2em;
		padding: 10px 18px;
	}

	.pause-button {
		width: 38px;
		height: 38px;
	}

	.difficulty-buttons button {
		padding: 10px 16px;
		font-size: 0.85em;
	}

	.modal {
		padding: 28px 22px;
	}

	.modal h2 {
		font-size: 1.9em;
	}

	.give-up-button,
	.new-game-button {
		padding: 12px 24px;
		font-size: 0.95em;
	}

	.history-modal {
		max-width: 90%;
	}

	.history-item {
		padding: 12px;
	}

	.history-stats {
		flex-wrap: wrap;
		gap: 15px;
	}
}

/* Mobile Large (480px - 575px) */
@media (min-width: 480px) and (max-width: 575px) {
	.sudoku-container {
		padding: 25px 12px;
		gap: 18px;
	}

	h1 {
		font-size: 1.8em !important;
	}

	.sudoku-cell {
		width: 40px;
		height: 40px;
	}

	.sudoku-cell input {
		font-size: 1.25em;
	}

	.paused-overlay {
		min-height: 365px;
		width: 365px;
		max-width: 90vw;
	}

	.controls {
		gap: 8px;
		flex-wrap: wrap;
	}

	.hint-button {
		padding: 9px 16px;
		font-size: 0.95em;
	}

	.history-button {
		width: 42px;
		height: 42px;
		font-size: 1.1em;
	}

	.timer {
		font-size: 1.1em;
		padding: 9px 16px;
		gap: 8px;
	}

	.pause-button {
		width: 36px;
		height: 36px;
		font-size: 0.95em;
	}

	.difficulty-buttons {
		flex-wrap: wrap;
		gap: 8px;
	}

	.difficulty-buttons button {
		padding: 9px 14px;
		font-size: 0.82em;
		flex: 1 1 calc(50% - 4px);
		min-width: 120px;
	}

	.modal {
		padding: 25px 18px;
		width: 92%;
	}

	.modal h2 {
		font-size: 1.7em;
	}

	.start-button {
		padding: 16px;
		font-size: 1.2em;
	}

	.give-up-button,
	.new-game-button {
		padding: 11px 20px;
		font-size: 0.92em;
	}

	.history-modal {
		width: 92%;
	}

	.history-header h2 {
		font-size: 1.5em;
	}

	.history-item {
		padding: 10px;
		gap: 8px;
	}

	.history-stats {
		flex-direction: column;
		gap: 8px;
	}
}

/* Mobile Medium (375px - 479px) */
@media (min-width: 375px) and (max-width: 479px) {
	.sudoku-container {
		padding: 20px 10px;
		gap: 15px;
	}

	h1 {
		font-size: 1.6em !important;
		letter-spacing: 1px;
	}

	.sudoku-cell {
		width: 36px;
		height: 36px;
	}

	.sudoku-cell input {
		font-size: 1.15em;
	}

	.paused-overlay {
		min-height: 330px;
		width: 330px;
		max-width: 92vw;
	}

	.pause-icon {
		font-size: 4em;
	}

	.paused-overlay p {
		font-size: 1.4em;
	}

	.controls {
		flex-direction: column;
		width: 100%;
		gap: 10px;
	}

	.hint-button {
		width: 100%;
		justify-content: center;
		padding: 10px 18px;
	}

	.history-button {
		width: 100%;
		height: 44px;
		border-radius: 50px;
		justify-content: center;
	}

	.timer {
		width: 100%;
		justify-content: center;
		font-size: 1.05em;
		padding: 10px 16px;
	}

	.pause-button {
		width: 36px;
		height: 36px;
	}

	.difficulty-buttons {
		flex-direction: column;
		width: 100%;
		gap: 8px;
	}

	.difficulty-buttons button {
		width: 100%;
		padding: 11px 18px;
		font-size: 0.9em;
	}

	.modal {
		padding: 22px 16px;
		width: 94%;
	}

	.modal h2 {
		font-size: 1.6em;
	}

	.instructions ul {
		padding-left: 18px;
	}

	.instructions li {
		font-size: 0.95em;
	}

	.start-button {
		padding: 15px;
		font-size: 1.15em;
	}

	.action-buttons {
		width: 100%;
		flex-direction: column;
		gap: 10px;
	}

	.give-up-button,
	.new-game-button {
		width: 100%;
		justify-content: center;
		padding: 12px 20px;
	}

	.history-modal {
		width: 94%;
	}

	.history-header h2 {
		font-size: 1.3em;
		gap: 6px;
	}

	.close-history {
		width: 36px;
		height: 36px;
		font-size: 1.3em;
	}

	.history-item {
		padding: 10px;
	}

	.difficulty-badge {
		font-size: 0.75em;
		padding: 3px 10px;
	}

	.history-stats {
		flex-direction: column;
		gap: 6px;
		font-size: 0.9em;
	}

	.history-date {
		font-size: 0.8em;
	}
}

/* Mobile Small (320px - 374px) */
@media (max-width: 374px) {
	.sudoku-container {
		padding: 15px 8px;
		gap: 12px;
	}

	h1 {
		font-size: 1.4em !important;
		letter-spacing: 0.5px;
	}

	.sudoku-cell {
		width: 32px;
		height: 32px;
	}

	.sudoku-cell input {
		font-size: 1.05em;
	}

	.paused-overlay {
		min-height: 292px;
		width: 292px;
		max-width: 94vw;
	}

	.pause-icon {
		font-size: 3.5em;
	}

	.paused-overlay p {
		font-size: 1.3em;
	}

	.controls {
		flex-direction: column;
		width: 100%;
		gap: 8px;
	}

	.hint-button {
		width: 100%;
		justify-content: center;
		padding: 9px 16px;
		font-size: 0.95em;
	}

	.history-button {
		width: 100%;
		height: 42px;
		border-radius: 50px;
	}

	.timer {
		width: 100%;
		justify-content: center;
		font-size: 1em;
		padding: 9px 14px;
		gap: 6px;
	}

	.pause-button {
		width: 34px;
		height: 34px;
		font-size: 0.9em;
	}

	.difficulty-buttons {
		flex-direction: column;
		width: 100%;
		gap: 6px;
	}

	.difficulty-buttons button {
		width: 100%;
		padding: 10px 16px;
		font-size: 0.85em;
	}

	.modal {
		padding: 20px 14px;
		width: 96%;
		border-radius: 20px;
	}

	.modal h2 {
		font-size: 1.5em;
	}

	.instructions {
		margin: 15px 0;
	}

	.instructions ul {
		padding-left: 16px;
	}

	.instructions li {
		font-size: 0.9em;
		line-height: 1.5;
	}

	.difficulty-selector {
		margin: 20px 0;
	}

	.start-button {
		padding: 14px;
		font-size: 1.1em;
	}

	.action-buttons {
		width: 100%;
		flex-direction: column;
		gap: 8px;
	}

	.give-up-button,
	.new-game-button {
		width: 100%;
		justify-content: center;
		padding: 11px 18px;
		font-size: 0.9em;
	}

	.history-modal {
		width: 96%;
	}

	.history-header h2 {
		font-size: 1.2em;
		gap: 5px;
	}

	.close-history {
		width: 34px;
		height: 34px;
		font-size: 1.2em;
	}

	.history-item {
		padding: 8px;
		gap: 6px;
	}

	.difficulty-badge {
		font-size: 0.7em;
		padding: 2px 8px;
	}

	.history-stats {
		flex-direction: column;
		gap: 5px;
		font-size: 0.85em;
	}

	.history-date {
		font-size: 0.75em;
	}

	.victory-icon {
		font-size: 4em;
	}

	.victory-message {
		font-size: 1em;
		line-height: 2;
	}

	.warning-icon {
		font-size: 3.5em;
	}

	.encouragement {
		font-size: 0.95em;
		line-height: 1.8;
	}
}
</style>
