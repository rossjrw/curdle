<template>
  <Message :message="message"></Message>
  <form id="maker" @submit="copyLinkToClipboard">
    <p>Enter a 5-letter word:</p>
    <p><input type="text" v-model="word" ref="picker"/></p>
    <p class="indicator">{{ indication }}</p>
    <p><b>Optional:</b> Who created this Curdle?</p>
    <p><input type="text" v-model="sender" /></p>
    <div class="link">
      <p>Copy this link to share your Curdle:</p>
      <p>
        <a :href="link">{{ link }}</a>
      </p>
      <p>
        <input
          type="text"
          readonly
          :value="link"
          ref="copyLink"
          class="linkBox"
        />

        <input type="submit" value="COPY" />
      </p>
    </div>
  </form>
</template>

<script setup lang="ts">
import { onMounted } from "vue"
import Message from "./Message.vue"
import {
  indications,
  normaliseWord,
  recreateGameId,
  wordInDictionary,
} from "./words"

const copyLink = $ref<HTMLInputElement>()
let word = $ref("")
let indication = $ref("")
let sender = $ref("")

let message = $ref("")
let link = $computed(
  () => `${location.origin}${location.pathname}?game=${gameId}`
)

const picker = $ref<HTMLInputElement>()

let gameId = $computed(() => {
  word = normaliseWord(word)
  if (word.length < 5) {
    indication = indications.notEnoughLetters
    return ""
  }
  indication = indications[wordInDictionary(word)]
  return recreateGameId(word, sender)
})

function copyLinkToClipboard(event: Event) {
  copyLink.focus()
  copyLink.select()
  document.execCommand("copy")
  showMessage("Copied!")
  // Block default submit action (navigation)
  event.preventDefault()
}

function showMessage(msg: string, time = 1000) {
  message = msg
  if (time > 0) setTimeout(() => (message = ""), time)
}

onMounted(() => picker.focus())
</script>

<style scoped>
.link {
  padding: 1rem;
  background-color: #eee;
  border-radius: 1rem;
}

.indicator {
  margin-top: -1rem;
}

form#maker {
  flex: 1;
  margin-top: 10%;
}

.linkBox {
  width: 40ch;
  max-width: 100%;
  margin-right: 1ch;
}
</style>
