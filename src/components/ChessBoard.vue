<template>
  <div class="chess-container">
    <div class="controls">
      <button @click="generateNewSolution" class="control-btn">Generate New Solution</button>
    </div>
    <div class="chessboard">
      <div v-for="row in 9" :key="`row-${row}`" class="row">
        <div 
          v-for="col in 9" 
          :key="`${row}-${col}`"
          :class="[
            'cell',
            (row + col) % 2 === 0 ? 'white' : 'black',
            { 'king': hasKing(row-1, col-1) },
            { 'dragging': isDragging && dragPosition.row === row-1 && dragPosition.col === col-1 },
            { 'selected': !isDragging && selectedKing.row === row-1 && selectedKing.col === col-1 },
            { 'possible-move': !isDragging && selectedKing.row !== null && isPossibleMove(row-1, col-1) }
          ]"
          @click="selectKing(row-1, col-1)"
          @mousedown="startDrag(row-1, col-1, $event)"
          @mouseup="endDrag(row-1, col-1)"
          @mousemove="handleDrag($event)"
          @touchmove="handleDrag($event)"
        >
          <span v-if="hasKing(row-1, col-1)" class="king-symbol">
            <svg viewBox="0 0 45 45" class="king-svg">
              <g fill="none" fill-rule="evenodd" stroke="#000" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round">
                <path d="M22.5 11.63V6M20 8h5" stroke-linejoin="miter"/>
                <path d="M22.5 25s4.5-7.5 3-10.5c0 0-1-2.5-3-2.5s-3 2.5-3 2.5c-1.5 3 3 10.5 3 10.5" fill="#fff" stroke-linecap="butt" stroke-linejoin="miter"/>
                <path d="M11.5 37c5.5 3.5 15.5 3.5 21 0v-7s9-4.5 6-10.5c-4-6.5-13.5-3.5-16 4V27v-3.5c-3.5-7.5-13-10.5-16-4-3 6 5 10 5 10V37z" fill="#fff"/>
                <path d="M11.5 30c5.5-3 15.5-3 21 0M11.5 33.5c5.5-3 15.5-3 21 0M11.5 37c5.5-3 15.5-3 21 0"/>
              </g>
            </svg>
          </span>
        </div>
      </div>
    </div>
    <div v-if="moveError" class="move-error">
      {{ moveError }}
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue';

// Board state
const board = ref(Array(9).fill(null).map(() => Array(9).fill(false)));
const kingCount = computed(() => board.value.flat().filter(cell => cell).length);
const hasConflict = computed(() => checkConflicts());

// Helper function to get random integer between min and max (inclusive)
const getRandomInt = (min, max) => {
  return Math.floor(Math.random() * (max - min + 1)) + min;
};

// Check if position has a king
const hasKing = (row, col) => board.value[row][col];

// Check if a position is under attack
const isUnderAttack = (row, col, tempBoard = board.value) => {
  const directions = [
    [-1,-1], [-1,0], [-1,1],
    [0,-1],         [0,1],
    [1,-1],  [1,0],  [1,1]
  ];
  
  return directions.some(([dx, dy]) => {
    const newRow = row + dx;
    const newCol = col + dy;
    return newRow >= 0 && newRow < 9 && 
           newCol >= 0 && newCol < 9 && 
           tempBoard[newRow][newCol];
  });
};

// Check for conflicts between kings
const checkConflicts = () => {
  for (let row = 0; row < 9; row++) {
    for (let col = 0; col < 9; col++) {
      if (board.value[row][col]) {
        // Temporarily remove this king to not count it
        board.value[row][col] = false;
        if (isUnderAttack(row, col)) {
          board.value[row][col] = true;
          return true;
        }
        board.value[row][col] = true;
      }
    }
  }
  return false;
};

// Toggle king placement
const toggleKing = (row, col) => {
  if (board.value[row][col] || kingCount.value < 9) {
    board.value[row][col] = !board.value[row][col];
  }
};

// Generate a valid solution with random placement
const generateNewSolution = () => {
  // Clear selected king and error message
  selectedKing.value = { row: null, col: null };
  moveError.value = '';
  
  const maxAttempts = 1000; // Prevent infinite loops
  let attempts = 0;
  
  while (attempts < maxAttempts) {
    try {
      // Clear the board
      board.value = Array(9).fill(null).map(() => Array(9).fill(false));
      let kingsPlaced = 0;
      
      // Try to place 9 kings
      while (kingsPlaced < 9) {
        // Get random position
        const row = getRandomInt(0, 8);
        const col = getRandomInt(0, 8);
        
        // If position is empty and not under attack, place a king
        if (!board.value[row][col] && !isUnderAttack(row, col)) {
          board.value[row][col] = true;
          kingsPlaced++;
        }
        
        // If we've tried too many times at this king count, throw error
        if (attempts++ > maxAttempts) {
          throw new Error("Failed to place kings");
        }
      }
      
      // If we placed all 9 kings successfully, break the loop
      if (kingsPlaced === 9) {
        break;
      }
    } catch (error) {
      console.log("Retrying king placement...");
      attempts++;
      continue;
    }
  }
  
  // If we couldn't place all kings after max attempts, clear the board
  if (attempts >= maxAttempts) {
    console.log("Could not find solution after maximum attempts");
    board.value = Array(9).fill(null).map(() => Array(9).fill(false));
  }
};

// Generate initial solution on mount
onMounted(() => {
  generateNewSolution();
});

const isDragging = ref(false);
const dragPosition = ref({ row: null, col: null });

const startDrag = (row, col, event) => {
  if (hasKing(row, col)) {
    isDragging.value = true;
    dragPosition.value = { row, col };
    // Clear selected king when starting to drag
    selectedKing.value = { row: null, col: null };
    event.preventDefault();
  }
};

// Check if the move is valid (one square in any direction)
const isValidMove = (startRow, startCol, endRow, endCol) => {
  const rowDiff = Math.abs(startRow - endRow);
  const colDiff = Math.abs(startCol - endCol);
  return rowDiff <= 1 && colDiff <= 1;
};

// Check if the move is possible without conflict
const isPossibleMove = (row, col) => {
  if (selectedKing.value.row === null || selectedKing.value.col === null) return false;
  if (!isValidMove(selectedKing.value.row, selectedKing.value.col, row, col)) return false;
  return !isUnderAttack(row, col, board.value.map((r, i) => r.map((c, j) => (i === selectedKing.value.row && j === selectedKing.value.col) ? false : c)));
};

const selectedKing = ref({ row: null, col: null });

const moveError = ref('');

// Helper to validate move with specific error messages
const validateMove = (startRow, startCol, endRow, endCol) => {
  if (!isValidMove(startRow, startCol, endRow, endCol)) {
    return "Kings can only move one square in any direction";
  }
  
  if (isUnderAttack(endRow, endCol, board.value.map((r, i) => r.map((c, j) => 
    (i === startRow && j === startCol) ? false : c)))) {
    return "This move would put kings in conflict";
  }
  
  return null;
};

const selectKing = (row, col) => {
  moveError.value = ''; // Clear previous error
  
  if (hasKing(row, col)) {
    selectedKing.value = { row, col };
  } else if (selectedKing.value.row !== null) {
    const error = validateMove(selectedKing.value.row, selectedKing.value.col, row, col);
    if (error) {
      moveError.value = error;
      addInvalidMoveEffect(row, col);
      // Clear error after 2 seconds
      setTimeout(() => moveError.value = '', 2000);
    } else {
      board.value[selectedKing.value.row][selectedKing.value.col] = false;
      board.value[row][col] = true;
      addMoveEffect(row, col);
      saveMove();
      selectedKing.value = { row: null, col: null };
    }
  }
};

// Cache for possible moves
const possibleMovesCache = ref(new Map());

// Board state with move history
const moveHistory = ref([]);
const currentMoveIndex = ref(-1);

// Computed property for possible moves
const possibleMoves = computed(() => {
  if (!selectedKing.value.row) return [];
  const key = `${selectedKing.value.row},${selectedKing.value.col}`;
  
  if (!possibleMovesCache.value.has(key)) {
    const moves = calculatePossibleMoves(selectedKing.value.row, selectedKing.value.col);
    possibleMovesCache.value.set(key, moves);
  }
  
  return possibleMovesCache.value.get(key);
});

// Clear cache when board changes
watch(board, () => {
  possibleMovesCache.value.clear();
});

// Undo/Redo functions
const undo = () => {
  if (currentMoveIndex.value > 0) {
    currentMoveIndex.value--;
    board.value = JSON.parse(JSON.stringify(moveHistory.value[currentMoveIndex.value]));
  }
};

const redo = () => {
  if (currentMoveIndex.value < moveHistory.value.length - 1) {
    currentMoveIndex.value++;
    board.value = JSON.parse(JSON.stringify(moveHistory.value[currentMoveIndex.value]));
  }
};

// Save move to history
const saveMove = () => {
  // Remove any future moves if we're in the middle of history
  moveHistory.value = moveHistory.value.slice(0, currentMoveIndex.value + 1);
  moveHistory.value.push(JSON.parse(JSON.stringify(board.value)));
  currentMoveIndex.value = moveHistory.value.length - 1;
};

// Modified endDrag with move validation and history
const endDrag = (row, col) => {
  if (!isDragging.value) return;
  
  const error = validateMove(dragPosition.value.row, dragPosition.value.col, row, col);
  if (!error) {
    board.value[dragPosition.value.row][dragPosition.value.col] = false;
    board.value[row][col] = true;
    saveMove();
    addMoveEffect(row, col);
  } else {
    moveError.value = error;
    addInvalidMoveEffect(row, col);
    setTimeout(() => moveError.value = '', 2000);
  }
  
  isDragging.value = false;
  dragPosition.value = { row: null, col: null };
};

// Visual feedback functions
const addMoveEffect = (row, col) => {
  const cell = document.querySelector(`[data-pos="${row}-${col}"]`);
  if (cell) {
    cell.classList.add('move-success');
    setTimeout(() => cell.classList.remove('move-success'), 500);
  }
};

const addInvalidMoveEffect = (row, col) => {
  const cell = document.querySelector(`[data-pos="${row}-${col}"]`);
  if (cell) {
    cell.classList.add('move-invalid');
    setTimeout(() => cell.classList.remove('move-invalid'), 500);
  }
};

const handleDrag = (event) => {
  if (!isDragging.value) return;
  
  event.preventDefault();
  const touch = event.type === 'touchmove' ? event.touches[0] : event;
  
  // Get the element under the pointer/touch
  const element = document.elementFromPoint(
    touch.clientX,
    touch.clientY
  );
  
  if (element?.classList.contains('cell')) {
    const row = parseInt(element.getAttribute('data-row'));
    const col = parseInt(element.getAttribute('data-col'));
    
    // Update visual feedback
    if (isValidMove(dragPosition.value.row, dragPosition.value.col, row, col)) {
      element.classList.add('drag-over');
    }
  }
};
</script>
<style scoped>
.chess-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 1.5rem;
    padding: 1.5rem;
    background: darkturquoise;
    border-radius: 8px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    position: relative;
}

.controls {
  display: flex;
  justify-content: center;
}

.control-btn {
  padding: 0.75rem 1.5rem;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.3s ease;
  font-weight: 600;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
}

.control-btn:hover {
  background-color: #45a049;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.control-btn:active {
  transform: translateY(0);
}

.chessboard {
  display: flex;
  flex-direction: column;
  border: 12px solid #624731;
  border-radius: 4px;
  box-shadow: 
    0 8px 16px rgba(0, 0, 0, 0.2),
    inset 0 0 50px rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
  background: #624731;
  padding: 2px;
  position: relative;
}

.chessboard::before {
  content: '';
  position: absolute;
  top: -12px;
  left: -12px;
  right: -12px;
  bottom: -12px;
  background: linear-gradient(45deg, #8b5e34, #624731);
  z-index: -1;
  border-radius: 8px;
  box-shadow: 
    0 10px 20px rgba(0, 0, 0, 0.3),
    inset 0 0 100px rgba(0, 0, 0, 0.2);
}

.row {
    display: flex;
}

.cell {
    width: 80px;  /* Increased from 60px */
    height: 80px; /* Increased from 60px */
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    transition: all 0.3s ease;
    position: relative;
    overflow: hidden;
}

.cell:hover {
  /* Remove the general hover effect */
  z-index: 1;
}

/* Add hover effect only for cells with kings */
.cell.king:hover {
  transform: scale(1.05);
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.3);
}

.cell.white {
  background: linear-gradient(135deg, #f0d9b5, #e6cca3);
}

.cell.black {
  background: linear-gradient(135deg, #b58863, #9e7b5a);
}

.king-symbol {
    width: 65px;  /* Increased from 45px */
    height: 65px; /* Increased from 45px */
    display: flex;
    align-items: center;
    justify-content: center;
    filter: drop-shadow(0 3px 3px rgba(0, 0, 0, 0.3));
    transition: all 0.3s ease;
}

.king-svg {
  width: 100%;
  height: 100%;
}

.cell.selected .king-svg g {
  stroke: #4a90e2;
  filter: drop-shadow(0 0 5px #4a90e2);
}

.cell:hover .king-symbol {
  transform: translateY(-2px);
  filter: drop-shadow(0 4px 4px rgba(0, 0, 0, 0.3));
}

.cell.white.king:hover {
  background: linear-gradient(135deg, #f7e4c7, #ebd5b6);
  box-shadow: inset 0 0 20px rgba(255, 255, 255, 0.3);
}

.cell.black.king:hover {
  background: linear-gradient(135deg, #c69b7b, #af8c6c);
  box-shadow: inset 0 0 20px rgba(0, 0, 0, 0.1);
}

@keyframes place-king {
    0% {
        transform: scale(0) rotate(-180deg);
        opacity: 0;
    }
    100% {
        transform: scale(1) rotate(0);
        opacity: 1;
    }
}

/* Drag effect styles */
.cell.dragging {
    opacity: 0.5;
    background-color: #e0e0e0;
}

.cell.selected {
  position: relative;
  z-index: 3;
}

.cell.selected::after {
  content: '';
  position: absolute;
  top: -2px;
  left: -2px;
  right: -2px;
  bottom: -2px;
  border: 2px solid #4a90e2;
  border-radius: 4px;
  animation: glowing 1.5s infinite;
  pointer-events: none;
}

.cell.selected .king-symbol {
  color: #4a90e2;
  transform: scale(1.15);
  text-shadow: 0 0 10px rgba(74, 144, 226, 0.6);
}

@keyframes glowing {
  0% { box-shadow: 0 0 5px #4a90e2, 0 0 10px #4a90e2, 0 0 15px #4a90e2; }
  50% { box-shadow: 0 0 10px #4a90e2, 0 0 20px #4a90e2, 0 0 25px #4a90e2; }
  100% { box-shadow: 0 0 5px #4a90e2, 0 0 10px #4a90e2, 0 0 15px #4a90e2; }
}

.cell.possible-move {
  position: relative;
  overflow: hidden;
}

.cell.possible-move::before {
  content: '';
  position: absolute;
  width: 20px;
  height: 20px;
  background: rgb(54, 126, 57);
  border-radius: 50%;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  animation: pulse 1.5s infinite;
}

.move-success {
  animation: moveSuccess 0.5s ease-out;
}

.move-invalid {
  animation: moveInvalid 0.5s ease-out;
}

@keyframes pulse {
  0% { transform: translate(-50%, -50%) scale(0.8); opacity: 0.8; }
  50% { transform: translate(-50%, -50%) scale(1.2); opacity: 0.4; }
  100% { transform: translate(-50%, -50%) scale(0.8); opacity: 0.8; }
}

@keyframes moveSuccess {
  0% { background-color: rgba(76, 175, 80, 0.4); }
  100% { background-color: transparent; }
}

@keyframes moveInvalid {
  0%, 100% { transform: translateX(0); }
  20%, 60% { transform: translateX(-5px); }
  40%, 80% { transform: translateX(5px); }
}

@media (max-width: 800px) { /* Updated from 600px for better responsive behavior */
    .cell {
        width: 50px;  /* Increased from 40px */
        height: 50px; /* Increased from 40px */
    }
    
    .king-symbol {
        width: 40px;
        height: 40px;
    }
    
    .chessboard:hover {
        transform: none;
    }
}

@keyframes shake {
  0%, 100% { transform: translateX(-50%); }
  20%, 60% { transform: translateX(calc(-50% - 5px)); }
  40%, 80% { transform: translateX(calc(-50% + 5px)); }
}

.move-error {
  position: fixed;
  bottom: 30px;
  left: 50%;
  transform: translateX(-50%);
  background: #2c3e50;
  color: #ecf0f1;
  padding: 16px 28px;
  border-radius: 12px;
  box-shadow: 
    0 4px 6px rgba(0, 0, 0, 0.1),
    0 1px 3px rgba(0, 0, 0, 0.08);
  animation: fadeInUp 0.4s ease;
  z-index: 1000;
  text-align: center;
  max-width: 90vw;
  font-weight: 500;
  font-size: 1rem;
  display: flex;
  align-items: center;
  gap: 10px;
  border-left: 4px solid #e74c3c;
}

.move-error::before {
  content: '!';
  display: flex;
  align-items: center;
  justify-content: center;
  width: 20px;
  height: 20px;
  background: #e74c3c;
  border-radius: 50%;
  font-weight: bold;
  font-size: 14px;
}

@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translate(-50%, 20px);
  }
  to {
    opacity: 1;
    transform: translate(-50%, 0);
  }
}

@media (max-width: 480px) {
  .move-error {
    font-size: 0.9rem;
    padding: 12px 20px;
    bottom: 20px;
  }
}
</style>
