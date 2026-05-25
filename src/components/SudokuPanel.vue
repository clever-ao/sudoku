<template>
  <div class="sudoku-board">
    <div v-for="(cell, index) in cells" :key="index" class="cell" :class="getCellClass(cell.row, cell.col)"
      @click="selectCell(cell.row, cell.col)">
      <input v-model="board[cell.row][cell.col]" class="cell-input tracking-tight" maxlength="1"
        @input="handleInput(cell.row, cell.col)" :readonly="fixedCells[cell.row][cell.col]" :class="{
          'fixedInput': fixedCells[cell.row][cell.col],
          'userInput': !fixedCells[cell.row][cell.col],
          'error': isErrorCell(cell.row, cell.col)
        }" />
      <div class="candidates" v-if="!board[cell.row][cell.col]">
        <span v-for="n in 9" :key="n" class="candidate" :class="{ active: isCandidate(cell.row, cell.col, n) }">
          {{ n }}
        </span>
      </div>
      <!-- {{ isHighlighted(cell.row, cell.col) }} -->
    </div>
  <div class="controls">
    <button @click="startSolve">Solve</button>
    <button @click="stopSolve">Pause</button>
  </div>
  </div>

</template>

<script setup lang="ts">
import { computed, reactive, ref} from 'vue'

interface Cell {
  row: number
  col: number
}



const board = reactive<(string | number)[][]>([
  ['', '', '', 8, 3, '', 4, '', 6],
  ['', 3, '', '', '', '', 2, '', 8],
  [8, '', '', '', '', '', 7, 5, 3],

  [2, '', 8, '', '', 1, '', 7, 4],
  ['', '', '', '', '', '', 1, 8, 5],
  [1, '', '', '', 7, 8, '', '', 2],

  ['', '', 9, '', '', '', 8, '', ''],
  ['', 8, '', 2, 5, '', '', 4, ''],
  ['', 6, '', '', 8, 4, 5, '', '']
])

const fixedCells = board.map(row =>
  row.map(cell => cell !== '')
)
const selected = reactive({
  row: 0,
  col: 2
})

const highlightConfig = reactive({
  error: true,
  selected: true,
  sameNumber: true,
  sameRow: true,
  sameCol: true,
  sameBox: true,
  conflict: true,
  preFill: true,
})


const steps: any[] = []

const isValid = (
  board: (string | number)[][],
  row: number,
  col: number,
  num: number
) => {
  // 行
  for (let c = 0; c < 9; c++) {
    if (board[row][c] == num) return false
  }

  // 列
  for (let r = 0; r < 9; r++) {
    if (board[r][col] == num) return false
  }

  // 宫
  const br = Math.floor(row / 3) * 3
  const bc = Math.floor(col / 3) * 3

  for (let r = br; r < br + 3; r++) {
    for (let c = bc; c < bc + 3; c++) {
      if (board[r][c] == num) return false
    }
  }

  return true
}

const solve = (board: (string | number)[][]): boolean => {
  for (let row = 0; row < 9; row++) {
    for (let col = 0; col < 9; col++) {

      if (board[row][col] === '' || board[row][col] === 0) {

        for (let num = 1; num <= 9; num++) {

          if (isValid(board, row, col, num)) {
            board[row][col] = num

            steps.push({
              type: 'fill',
              row,
              col,
              value: num
            })

            if (solve(board)) return true

            board[row][col] = ''

            steps.push({
              type: 'clear',
              row,
              col
            })
          }
        }

        return false
      }
    }
  }

  return true
}

const delay = (ms: number) => new Promise(res => setTimeout(res, ms))

const isPlaying = ref(false)
const speed = ref(50)

const playSteps = async () => {
  isPlaying.value = true

  for (const step of steps) {
    if (!isPlaying.value) break

    if (step.type === 'fill') {
      board[step.row][step.col] = step.value
    }

    if (step.type === 'clear') {
      board[step.row][step.col] = ''
    }

    await delay(speed.value)
  }

  isPlaying.value = false
}

const startSolve = () => {
  steps.length = 0

  // ⚠️ 必须 deep copy，不然会污染 UI board
  const copyBoard = board.map(row => [...row])

  solve(copyBoard)
  playSteps()
}

const stopSolve = () => {
  isPlaying.value = false
}
/* 错误 -> 选中 -> 同数字 -> 同行同列 -> 同九宫格 -> 候选数字冲突 -> 预填 -> 普通 */
const getCellState = (row: number, col: number) => {
  const value = board[row][col]
  const selectedValue = board[selected.row][selected.col]

  const state = {
    isError: false,
    isSelected: false,
    isSameNumber: false,
    isSameRow: false,
    isSameCol: false,
    isSameBox: false,
    isConflict: false,
    isPreFill: false
  }

  // 1. error（最高优先级）
  if (highlightConfig.error && isErrorCell(row, col)) {
    state.isError = true
  }

  // 2. selected
  if (row === selected.row && col === selected.col) {
    state.isSelected = true
  }

  // 3. 同数字
  if (
    highlightConfig.sameNumber &&
    selectedValue &&
    value == selectedValue &&
    !(row === selected.row && col === selected.col)
  ) {
    state.isSameNumber = true
  }

  // 4. 同行
  if (highlightConfig.sameRow && row === selected.row) {
    state.isSameRow = true
  }

  // 5. 同列
  if (highlightConfig.sameCol && col === selected.col) {
    state.isSameCol = true
  }

  // 6. 同宫
  if (
    highlightConfig.sameBox &&
    Math.floor(row / 3) === Math.floor(selected.row / 3) &&
    Math.floor(col / 3) === Math.floor(selected.col / 3)
  ) {
    state.isSameBox = true
  }

  // 7. 候选数字冲突
  if (highlightConfig.conflict && isErrorCell(row, col, selectedValue)) {
    state.isConflict = true
  }

  // 8. 预填
  // if (highlightConfig.preFill && fixedCells[row][col]) {
  //   state.isPreFill = true
  // }

  return state
}

const getCellClass = (row: number, col: number) => {
  const s = getCellState(row, col)

  return {
    error: s.isError,
    selected: s.isSelected,

    sameNumber: s.isSameNumber,
    sameRow: s.isSameRow,
    sameCol: s.isSameCol,
    sameBox: s.isSameBox,

    conflict: s.isConflict,
    preFill: s.isPreFill,

    fixed: fixedCells[row][col],
    thickRight: (col + 1) % 3 === 0 && col !== 8,
    thickBottom: (row + 1) % 3 === 0 && row !== 8,
  }
}


const cells = computed<Cell[]>(() => {
  const result: Cell[] = []

  for (let row = 0; row < 9; row++) {
    for (let col = 0; col < 9; col++) {
      result.push({
        row,
        col
      })
    }
  }

  return result
})

const handleInput = (row: number, col: number) => {
  let value = String(board[row][col])

  // 只能输入 1~9
  value = value.replace(/[^1-9]/g, '')

  // 只保留一个字符
  value = value.slice(0, 1)

  board[row][col] = value
}

const selectCell = (row: number, col: number) => { selected.row = row; selected.col = col }

const isErrorCell = (row: number, col: number, value: string | number = board[row][col]) => {
  // const value = board[row][col]
  if (!value) return false

  // 同行
  for (let c = 0; c < 9; c++) {
    if (c !== col && board[row][c] == value) {
      return true
    }
  }

  // 同列
  for (let r = 0; r < 9; r++) {
    if (r !== row && board[r][col] == value) {
      return true
    }
  }

  // 同宫
  const br = Math.floor(row / 3) * 3
  const bc = Math.floor(col / 3) * 3

  for (let r = br; r < br + 3; r++) {
    for (let c = bc; c < bc + 3; c++) {
      if ((r !== row || c !== col) && board[r][c] == value) {
        return true
      }
    }
  }

  return false
}
const isCandidate = (row: number, col: number, num: number) => {
  // 如果格子已有值，不显示
  if (board[row][col]) return false

  // 行冲突
  for (let c = 0; c < 9; c++) {
    if (board[row][c] == num) return false
  }

  // 列冲突
  for (let r = 0; r < 9; r++) {
    if (board[r][col] == num) return false
  }

  // 宫冲突
  const br = Math.floor(row / 3) * 3
  const bc = Math.floor(col / 3) * 3

  for (let r = br; r < br + 3; r++) {
    for (let c = bc; c < bc + 3; c++) {
      if (board[r][c] == num) return false
    }
  }

  return true
}
</script>

<style scoped>

.cell {
  transition: all 0.15s ease;
}

.cell input {
  transition: transform 0.1s;
}

.cell input:focus {
  transform: scale(1.1);
}

.sudoku-board {
  @apply w-[540px] h-[540px];
  @apply grid grid-cols-9 grid-rows-9;

  @apply border-[4px] border-[#3b4b63];
  @apply bg-[#f5f5f5];

  @apply select-none;
}

.cell {
  @apply flex items-center justify-center relative;
  @apply text-[42px] font-normal text-[#394b63];
  @apply border-r border-b border-[#b8c0cc];
  @apply bg-[#f7f7f7];
  @apply transition-all duration-150;
  @apply cursor-pointer;
  @apply relative;
}

/* hover */
.cell:hover {
  @apply bg-[#c8d9ec];
}

/* 输入框 */
.cell-input {
  @apply w-full h-full;

  @apply border-0 outline-none bg-transparent;

  @apply text-center text-4xl font-satoshi;

  @apply cursor-pointer caret-transparent;
}

/* 错误 -> 选中 -> 同数字 -> 同行同列 -> 同九宫格 -> 候选数字冲突 -> 预填 -> 普通 */
.cell.error {
  @apply !bg-red-200;
}

.error.userInput {
  @apply !bg-red-200;
  @apply !text-red-500;
}

/* 选中格 */
.cell.selected {
  @apply !bg-[#a9c7e5];
}

/* 同数字 */
.cell.sameNumber {
  @apply !bg-[#ffe7a3];
}

/* 同行 */
.cell.sameRow {
  @apply bg-[#dfe7f1];
}

/* 同列 */
.cell.sameCol {
  @apply bg-[#dfe7f1];
}

/* 同九宫格 */
.cell.sameBox {
  @apply bg-[#d8e2ee];
}

/* 冲突 */
.cell.conflict {
  @apply bg-[#d8e2ee];
}

/* 粗边框 */
.cell.thickRight {
  @apply border-r-[4px] border-r-[#3b4b63];
}

.cell.thickBottom {
  @apply border-b-[4px] border-b-[#3b4b63];
}

/* 原始题目数字 */
.fixedInput {
  @apply text-[#394b63];
  @apply font-medium;
}

/* 用户输入数字 */
.userInput {
  @apply text-[#4a78d1];
  @apply font-normal;
}

/* 固定数字 */
.cell.fixed {
  @apply text-[#394b63];
}

/* 候选数字 */
.candidates {
  position: absolute;
  inset: 0;
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  font-size: 10px;
  pointer-events: none;
  padding: 6px;
}

.candidate {
  display: flex;
  align-items: center;
  justify-content: center;
  color: #cbd5e1;
  /* 默认灰色 */
  transition: all 0.15s;
}

.candidate.active {
  color: #3b82f6;
  /* 可选数字蓝色 */
  font-weight: 600;
}
</style>