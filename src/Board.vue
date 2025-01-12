<template>
  <Message :message="message" :paste="sharePaste"></Message>
  <p>{{ sender ? sender : "Someone" }} has sent you a Curdle!</p>
  <p>{{ indications[indicator] }}</p>
  <div id="board" v-if="indicator !== 'invalid'">
    <div
      v-for="(row, index) in board"
      :class="[
        'row',
        shakeRowIndex === index && 'shake',
        success && currentRowIndex === index && 'jump',
      ]"
    >
      <div
        v-for="(tile, index) in row"
        :class="['tile', tile.letter && 'filled', tile.state && 'revealed']"
      >
        <div class="front" :style="{ transitionDelay: `${index * 300}ms` }">
          {{ tile.letter }}
        </div>
        <div
          :class="['back', tile.state]"
          :style="{
            transitionDelay: `${index * 300}ms`,
            animationDelay: `${index * 100}ms`,
          }"
        >
          {{ tile.letter }}
        </div>
      </div>
    </div>
  </div>
  <Keyboard @key="onKey" :letter-states="letterStates" />
</template>

<script setup lang="ts">
import { onUnmounted } from "vue"
import { LetterState } from "./types"
import Message from "./Message.vue"
import Keyboard from "./Keyboard.vue"
import {
  getWordOfTheDay,
  wordInDictionary,
  indications,
  recreateGameId,
} from "./words"

const props = defineProps({
  answer: {
    type: String,
    required: true,
  },
})

let { sender, indicator } = getWordOfTheDay()
if (indicator == null) indicator = "invalid"

const gameId = recreateGameId(props.answer, sender)

// Board state. Each tile is represented as { letter, state }
let board = $ref(
  Array.from({ length: 6 }, () =>
    Array.from({ length: 5 }, () => ({
      letter: "",
      state: LetterState.INITIAL,
    }))
  )
)

// Keep track of revealed letters for the virtual keyboard
const letterStates: Record<string, LetterState> = $ref({})

// Handle keyboard input.
let allowInput = true
const onKeyup = (e: KeyboardEvent) => onKey(e.key)
window.addEventListener("keyup", onKeyup)
onUnmounted(() => {
  window.removeEventListener("keyup", onKeyup)
})

// Recover game state from storage if present
const cachedGame = localStorage.getItem(gameId)
if (cachedGame) {
  board = JSON.parse(cachedGame)
  board.forEach((row) =>
    row.forEach((tile) => {
      // Update the letter state only if a) the letter does not already
      // have a state, or b) the state is a higher priority than the
      // current one
      const states = [
        LetterState.INITIAL,
        LetterState.ABSENT,
        LetterState.PRESENT,
        LetterState.CORRECT,
      ]
      if (
        !(tile.letter in letterStates) ||
        states.indexOf(tile.state) > states.indexOf(letterStates[tile.letter])
      )
        letterStates[tile.letter] = tile.state
    })
  )
  if (
    board.some((row) => row.every((tile) => tile.state === LetterState.CORRECT))
  )
    gameComplete()
}

// Feedback state: message and shake
let message = $ref("")
let shakeRowIndex = $ref(-1)
let success = $ref(false)

// Current active row.
let currentRowIndex = $ref(
  board.findIndex((row) => row[0].state === LetterState.INITIAL)
)
const currentRow = $computed(() => board[currentRowIndex])

function showMessage(msg: string, time = 1000) {
  message = msg
  if (time > 0) {
    setTimeout(() => {
      message = ""
    }, time)
  }
}

function shake() {
  shakeRowIndex = currentRowIndex
  setTimeout(() => {
    shakeRowIndex = -1
  }, 1000)
}

const icons = {
  [LetterState.CORRECT]: "🟩",
  [LetterState.PRESENT]: "🟨",
  [LetterState.ABSENT]: "⬜",
  [LetterState.INITIAL]: null,
}

let sharePaste = $ref("")
function makeSharePaste() {
  const boardString = board
    .slice(0, currentRowIndex + 1)
    .map((row) => {
      return row.map((tile) => icons[tile.state]).join("")
    })
    .join("  \n")
  const gameId = recreateGameId(props.answer, sender)
  const finalIndex = currentRowIndex >= 6 ? "X" : currentRowIndex + 1
  return [
    `Curdle ${gameId} ${finalIndex}/6`,
    ,
    boardString,
    ,
    `${location.origin}${location.pathname}?game=${gameId}`,
  ].join("\n")
}

function onKey(key: string) {
  if (!allowInput) return
  if (/^[a-zA-Z]$/.test(key)) {
    fillTile(key.toLowerCase())
  } else if (key === "Backspace") {
    clearTile()
  } else if (key === "Enter") {
    completeRow()
  }
}

function fillTile(letter: string) {
  for (const tile of currentRow) {
    if (!tile.letter) {
      tile.letter = letter
      break
    }
  }
}

function clearTile() {
  for (const tile of [...currentRow].reverse()) {
    if (tile.letter) {
      tile.letter = ""
      break
    }
  }
}

function gameComplete() {
  allowInput = false
  setTimeout(() => {
    sharePaste = makeSharePaste()
    showMessage("Nice", -1)
    success = true
  }, 1600)
}

function completeRow() {
  if (currentRow.every((tile) => tile.letter)) {
    const guess = currentRow.map((tile) => tile.letter).join("")
    if (
      // If the guess is the answer, it must be allowed
      guess !== props.answer &&
      // Disallow a word that's not in the dictionary
      wordInDictionary(guess) === "unknown" &&
      // ...unless the answer is also not in the dictionary
      wordInDictionary(props.answer) !== "unknown"
    ) {
      shake()
      showMessage("Word not in dictionary")
      return
    }

    const answerLetters: (string | null)[] = props.answer
      .toLowerCase()
      .split("")
    // first pass: mark correct ones
    currentRow.forEach((tile, i) => {
      if (answerLetters[i] === tile.letter) {
        tile.state = letterStates[tile.letter] = LetterState.CORRECT
        answerLetters[i] = null
      }
    })
    // second pass: mark the present
    currentRow.forEach((tile) => {
      if (!tile.state && answerLetters.includes(tile.letter)) {
        tile.state = LetterState.PRESENT
        answerLetters[answerLetters.indexOf(tile.letter)] = null
        if (!letterStates[tile.letter]) {
          letterStates[tile.letter] = LetterState.PRESENT
        }
      }
    })
    // 3rd pass: mark absent
    currentRow.forEach((tile) => {
      if (!tile.state) {
        tile.state = LetterState.ABSENT
        if (!letterStates[tile.letter]) {
          letterStates[tile.letter] = LetterState.ABSENT
        }
      }
    })

    try {
      // Cache game state
      localStorage.setItem(gameId, JSON.stringify(board))
    } catch (error) {
      // Could fail because e.g. private browsing - not important
      console.error(error)
    }

    allowInput = false
    if (currentRow.every((tile) => tile.state === LetterState.CORRECT)) {
      // yay!
      gameComplete()
    } else if (currentRowIndex < board.length - 1) {
      // go the next row
      currentRowIndex++
      setTimeout(() => {
        allowInput = true
      }, 1600)
    } else {
      // game over :(
      currentRowIndex++
      sharePaste = makeSharePaste()
      setTimeout(() => {
        showMessage(props.answer.toUpperCase(), -1)
      }, 1600)
    }
  } else {
    shake()
    showMessage("Not enough letters")
  }
}
</script>

<style scoped>
p {
  margin: 0;
}

#board {
  display: grid;
  grid-template-rows: repeat(6, 1fr);
  grid-gap: 5px;
  padding: min(1.5vw, 10px);
  box-sizing: border-box;
  width: min(95%, 350px);
  margin: 0px auto;
  flex: 1;
}

.row {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  grid-gap: 5px;
}
.tile {
  width: 100%;
  font-size: 2rem;
  line-height: 2rem;
  font-weight: bold;
  vertical-align: middle;
  text-transform: uppercase;
  user-select: none;
  position: relative;
}
.tile.filled {
  animation: zoom 0.2s;
}
.tile .front,
.tile .back {
  box-sizing: border-box;
  display: inline-flex;
  justify-content: center;
  align-items: center;
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  transition: transform 0.6s;
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;
}
.tile .front {
  border: 2px solid #d3d6da;
}
.tile.filled .front {
  border-color: #999;
}
.tile .back {
  transform: rotateX(180deg);
}
.tile.revealed .front {
  transform: rotateX(180deg);
}
.tile.revealed .back {
  transform: rotateX(0deg);
}

@keyframes zoom {
  0% {
    transform: scale(1.1);
  }
  100% {
    transform: scale(1);
  }
}

.shake {
  animation: shake 0.5s;
}

@keyframes shake {
  0% {
    transform: translate(1px);
  }
  10% {
    transform: translate(-2px);
  }
  20% {
    transform: translate(2px);
  }
  30% {
    transform: translate(-2px);
  }
  40% {
    transform: translate(2px);
  }
  50% {
    transform: translate(-2px);
  }
  60% {
    transform: translate(2px);
  }
  70% {
    transform: translate(-2px);
  }
  80% {
    transform: translate(2px);
  }
  90% {
    transform: translate(-2px);
  }
  100% {
    transform: translate(1px);
  }
}

.jump .tile .back {
  animation: jump 0.5s;
}

@keyframes jump {
  0% {
    transform: translateY(0px);
  }
  20% {
    transform: translateY(5px);
  }
  60% {
    transform: translateY(-25px);
  }
  90% {
    transform: translateY(3px);
  }
  100% {
    transform: translateY(0px);
  }
}

@media (max-height: 680px) {
  .tile {
    font-size: 3vh;
  }
}
</style>
