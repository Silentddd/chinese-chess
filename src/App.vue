<script setup>
import { ref, computed } from 'vue'

// 棋盘状态
const board = ref([
  ['俥', '傌', '象', '士', '將', '士', '象', '傌', '俥'],
  ['　', '　', '　', '　', '　', '　', '　', '　', '　'],
  ['　', '砲', '　', '　', '　', '　', '　', '砲', '　'],
  ['卒', '　', '卒', '　', '卒', '　', '卒', '　', '卒'],
  ['　', '　', '　', '　', '　', '　', '　', '　', '　'],
  ['　', '　', '　', '　', '　', '　', '　', '　', '　'],
  ['兵', '　', '兵', '　', '兵', '　', '兵', '　', '兵'],
  ['　', '炮', '　', '　', '　', '　', '　', '炮', '　'],
  ['　', '　', '　', '　', '　', '　', '　', '　', '　'],
  ['車', '馬', '相', '仕', '帥', '仕', '相', '馬', '車']
])

// 当前选中的棋子位置
const selectedPiece = ref(null)
// 当前回合（true为红方，false为黑方）
const isRedTurn = ref(true)

// 判断是否是红方棋子
const isRedPiece = (piece) => {
  return ['車', '馬', '相', '仕', '帥', '炮', '兵'].includes(piece)
}

// 判断是否是黑方棋子
const isBlackPiece = (piece) => {
  return ['俥', '傌', '象', '士', '將', '砲', '卒'].includes(piece)
}

// 检查马腿位置是否有棋子
const hasHorseLeg = (fromRow, fromCol, toRow, toCol) => {
  const rowDiff = toRow - fromRow
  const colDiff = toCol - fromCol
  
  // 检查马腿位置
  if (Math.abs(rowDiff) === 2) {
    const legRow = fromRow + (rowDiff > 0 ? 1 : -1)
    return board.value[legRow][fromCol] !== '　'
  } else {
    const legCol = fromCol + (colDiff > 0 ? 1 : -1)
    return board.value[fromRow][legCol] !== '　'
  }
}

// 检查象眼位置是否有棋子
const hasElephantEye = (fromRow, fromCol, toRow, toCol) => {
  const eyeRow = (fromRow + toRow) / 2
  const eyeCol = (fromCol + toCol) / 2
  return board.value[eyeRow][eyeCol] !== '　'
}

// 检查移动是否合法
const isValidMove = (fromRow, fromCol, toRow, toCol) => {
  const piece = board.value[fromRow][fromCol]
  const targetPiece = board.value[toRow][toCol]
  
  // 不能吃自己的子
  if (targetPiece !== '　' && 
      ((isRedPiece(piece) && isRedPiece(targetPiece)) || 
       (isBlackPiece(piece) && isBlackPiece(targetPiece)))) {
    return false
  }

  // 计算移动距离
  const rowDiff = toRow - fromRow
  const colDiff = toCol - fromCol

  // 根据不同棋子类型检查移动是否合法
  switch (piece) {
    case '帥':
    case '將':
      // 将帅只能在九宫格内移动，每次只能走一步
      if (isRedPiece(piece)) {
        if (toRow < 7 || toRow > 9 || toCol < 3 || toCol > 5) return false
      } else {
        if (toRow > 2 || toRow < 0 || toCol < 3 || toCol > 5) return false
      }
      return Math.abs(rowDiff) + Math.abs(colDiff) === 1

    case '仕':
    case '士':
      // 士只能在九宫格内斜着走一步
      if (isRedPiece(piece)) {
        if (toRow < 7 || toRow > 9 || toCol < 3 || toCol > 5) return false
      } else {
        if (toRow > 2 || toRow < 0 || toCol < 3 || toCol > 5) return false
      }
      return Math.abs(rowDiff) === 1 && Math.abs(colDiff) === 1

    case '相':
    case '象':
      // 相/象走田字，不能过河，不能塞象眼
      if (isRedPiece(piece)) {
        if (toRow < 5) return false // 红相不能过河
      } else {
        if (toRow > 4) return false // 黑象不能过河
      }
      if (Math.abs(rowDiff) !== 2 || Math.abs(colDiff) !== 2) return false
      return !hasElephantEye(fromRow, fromCol, toRow, toCol)

    case '馬':
    case '傌':
      // 马走日，不能蹩马腿
      if ((Math.abs(rowDiff) === 2 && Math.abs(colDiff) === 1) || 
          (Math.abs(rowDiff) === 1 && Math.abs(colDiff) === 2)) {
        return !hasHorseLeg(fromRow, fromCol, toRow, toCol)
      }
      return false

    case '車':
    case '俥':
      // 车走直线
      if (rowDiff !== 0 && colDiff !== 0) return false
      // 检查路径上是否有其他棋子
      if (rowDiff !== 0) {
        const step = rowDiff > 0 ? 1 : -1
        for (let i = fromRow + step; i !== toRow; i += step) {
          if (board.value[i][fromCol] !== '　') return false
        }
      } else {
        const step = colDiff > 0 ? 1 : -1
        for (let i = fromCol + step; i !== toCol; i += step) {
          if (board.value[fromRow][i] !== '　') return false
        }
      }
      return true

    case '炮':
    case '砲':
      // 炮走直线，吃子时需要翻山
      if (rowDiff !== 0 && colDiff !== 0) return false
      let count = 0
      if (rowDiff !== 0) {
        const step = rowDiff > 0 ? 1 : -1
        for (let i = fromRow + step; i !== toRow; i += step) {
          if (board.value[i][fromCol] !== '　') count++
        }
      } else {
        const step = colDiff > 0 ? 1 : -1
        for (let i = fromCol + step; i !== toCol; i += step) {
          if (board.value[fromRow][i] !== '　') count++
        }
      }
      return targetPiece === '　' ? count === 0 : count === 1

    case '兵':
      // 红兵
      if (fromRow > 4) { // 未过河
        return rowDiff === -1 && colDiff === 0
      } else { // 已过河
        return (rowDiff === -1 && colDiff === 0) || 
               (rowDiff === 0 && Math.abs(colDiff) === 1)
      }

    case '卒':
      // 黑卒
      if (fromRow < 5) { // 未过河
        return rowDiff === 1 && colDiff === 0
      } else { // 已过河
        return (rowDiff === 1 && colDiff === 0) || 
               (rowDiff === 0 && Math.abs(colDiff) === 1)
      }

    default:
      return false
  }
}

// 处理棋子点击
const handlePieceClick = (row, col) => {
  const piece = board.value[row][col]
  
  // 如果已选中棋子，尝试移动或选择新棋子
  if (selectedPiece.value) {
    const [fromRow, fromCol] = [selectedPiece.value.row, selectedPiece.value.col]
    const selectedPieceName = board.value[fromRow][fromCol]
    
    // 如果点击的是自己的棋子，更新选中状态
    if (piece !== '　') {
      if ((isRedTurn.value && isRedPiece(piece)) || 
          (!isRedTurn.value && isBlackPiece(piece))) {
        selectedPiece.value = { row, col }
        return
      }
    }
    
    // 检查是否是当前回合方的棋子
    if ((isRedTurn.value && !isRedPiece(selectedPieceName)) || 
        (!isRedTurn.value && !isBlackPiece(selectedPieceName))) {
      return
    }
    
    // 尝试移动棋子
    if (isValidMove(fromRow, fromCol, row, col)) {
      board.value[row][col] = selectedPieceName
      board.value[fromRow][fromCol] = '　'
      selectedPiece.value = null
      isRedTurn.value = !isRedTurn.value // 切换回合
    }
    return
  }

  // 选择新棋子
  if (piece !== '　') {
    if ((isRedTurn.value && isRedPiece(piece)) || 
        (!isRedTurn.value && isBlackPiece(piece))) {
      selectedPiece.value = { row, col }
    }
  }
}

// 计算当前回合提示文字
const turnText = computed(() => {
  return isRedTurn.value ? '红方回合' : '黑方回合'
})

// 获取棋子的样式类
const getPieceClass = (piece) => {
  const classes = []
  if (piece !== '　') {
    classes.push('piece')
    if (isRedPiece(piece)) {
      classes.push('red-piece')
    } else if (isBlackPiece(piece)) {
      classes.push('black-piece')
    }
  }
  return classes
}
</script>

<template>
  <div class="chess-game">
    <div class="game-container">
      <div class="game-info">
        <h1>中国象棋</h1>
        <div class="turn-indicator" :class="{ 'red-turn': isRedTurn, 'black-turn': !isRedTurn }">
          {{ turnText }}
        </div>
      </div>
      <div class="chess-board">
        <div class="board-border">
          <div class="board-background">
            <div class="river">楚河　　　　汉界</div>
            <!-- 绘制九宫格 -->
            <div class="palace top-palace"></div>
            <div class="palace bottom-palace"></div>
          </div>
          <div class="board-grid">
            <div v-for="(row, rowIndex) in board" :key="rowIndex" class="board-row">
              <div
                v-for="(piece, colIndex) in row"
                :key="colIndex"
                class="board-cell"
                :class="{
                  selected: selectedPiece && selectedPiece.row === rowIndex && selectedPiece.col === colIndex
                }"
                @click="handlePieceClick(rowIndex, colIndex)"
              >
                <div v-if="piece !== '　'" :class="getPieceClass(piece)">
                  {{ piece }}
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.chess-game {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  background: linear-gradient(135deg, #f5f7fa 0%, #c3cfe2 100%);
  padding: 20px;
}

.game-container {
  background: white;
  padding: 30px;
  border-radius: 15px;
  box-shadow: 0 10px 20px rgba(0, 0, 0, 0.1);
}

.game-info {
  text-align: center;
  margin-bottom: 30px;
}

.game-info h1 {
  font-size: 2.5em;
  color: #2c3e50;
  margin-bottom: 20px;
  font-family: "Microsoft YaHei", sans-serif;
}

.turn-indicator {
  font-size: 1.2em;
  padding: 10px 20px;
  border-radius: 8px;
  font-weight: bold;
  transition: all 0.3s ease;
}

.red-turn {
  background-color: #ffebee;
  color: #c62828;
  box-shadow: 0 2px 4px rgba(198, 40, 40, 0.2);
}

.black-turn {
  background-color: #263238;
  color: #ffffff;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
}

.chess-board {
  position: relative;
  padding: 20px;
  background: linear-gradient(135deg, #8B4513, #A0522D);
  border-radius: 8px;
}

.board-border {
  position: relative;
  width: 500px;
  height: 550px;
  background-color: #DEB887;
  border: 12px solid #8B4513;
  border-radius: 4px;
  box-shadow: 
    0 4px 8px rgba(0, 0, 0, 0.2),
    inset 0 0 0 2px #A0522D,
    inset 0 0 0 4px #8B4513;
}

.board-border::before {
  content: '';
  position: absolute;
  top: -12px;
  left: -12px;
  right: -12px;
  bottom: -12px;
  background: 
    linear-gradient(45deg, transparent 10px, #8B4513 10px) top left,
    linear-gradient(-45deg, transparent 10px, #8B4513 10px) top right,
    linear-gradient(135deg, transparent 10px, #8B4513 10px) bottom left,
    linear-gradient(-135deg, transparent 10px, #8B4513 10px) bottom right;
  background-size: 51% 51%;
  background-repeat: no-repeat;
  border-radius: 12px;
  z-index: -1;
}

.board-background {
  position: absolute;
  top: 20px;
  left: 20px;
  right: 20px;
  bottom: 20px;
  pointer-events: none;
}

.river {
  position: absolute;
  top: 45%;
  width: 100%;
  text-align: center;
  color: #8B4513;
  font-size: 24px;
  font-family: "KaiTi", "STKaiti", serif;
  opacity: 0.8;
}

.palace {
  position: absolute;
  width: 125px;
  height: 125px;
  border: 2px solid #8B4513;
}

.top-palace {
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
}

.bottom-palace {
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
}

.board-grid {
  position: relative;
  z-index: 1;
}

.board-row {
  display: flex;
  justify-content: center;
}

.board-cell {
  width: 50px;
  height: 50px;
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
}

.board-cell::before {
  content: '';
  position: absolute;
  top: 50%;
  left: 0;
  right: 0;
  height: 1px;
  background-color: #8B4513;
}

.board-cell::after {
  content: '';
  position: absolute;
  left: 50%;
  top: 0;
  bottom: 0;
  width: 1px;
  background-color: #8B4513;
}

.piece {
  width: 44px;
  height: 44px;
  border-radius: 50%;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 24px;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.3s ease;
  font-family: "KaiTi", "STKaiti", serif;
  background: radial-gradient(circle at 30% 30%, #fff 0%, #f0f0f0 100%);
  box-shadow: 
    0 2px 4px rgba(0, 0, 0, 0.2),
    inset 0 -2px 2px rgba(0, 0, 0, 0.1);
  border: none;
  position: relative;
}

.piece::after {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  border: 2px solid;
  border-radius: 50%;
}

.red-piece {
  color: #c62828;
}

.red-piece::after {
  border-color: #c62828;
}

.black-piece {
  color: #000000;
}

.black-piece::after {
  border-color: #000000;
}

.board-cell.selected .piece {
  transform: scale(1.1);
  box-shadow: 
    0 0 10px rgba(255, 215, 0, 0.6),
    inset 0 -2px 2px rgba(0, 0, 0, 0.1);
}

.board-cell.selected .piece::after {
  border-color: #FFD700;
  border-width: 3px;
}

.piece:hover {
  transform: scale(1.05);
  box-shadow: 
    0 4px 8px rgba(0, 0, 0, 0.3),
    inset 0 -2px 2px rgba(0, 0, 0, 0.1);
}

@media (max-width: 600px) {
  .chess-board {
    width: 100%;
    height: auto;
    aspect-ratio: 1;
  }

  .piece {
    width: 36px;
    height: 36px;
    font-size: 20px;
  }
}
</style>
