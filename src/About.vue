<template>
  <div>
    <p>
      Curdle is just like
      <a href="https://www.nytimes.com/games/wordle">Wordle</a>, except you get
      to pick the answer.
    </p>

    <p><a href="https://github.com/rossjrw/curdle">Source code</a></p>

    <h3>How to play</h3>
    <p>Pick a word and send it to someone else to solve. That's about it.</p>

    <h3>The dictionary</h3>
    <p>
      The answer can be any set of 5 letters, but the player will be informed
      whether or not it's in the dictionary:
    </p>
    <ul>
      <li>
        <b>{{ indications.common }}:</b><br />
        This word either <i>already has been</i> or <i>one day will be</i> the
        day's official Wordle answer (unless Wordle's answer list changes).
        There are {{ dictionary.common.length.toLocaleString() }} such words.
      </li>
      <li>
        <b>{{ indications.extended }}:</b><br />
        This word will never be the day's Wordle answer, but Wordle will accept
        it as a possible guess. There are
        {{ dictionary.extended.length.toLocaleString() }} such words.
      </li>
      <li>
        <b>{{ indications.unknown }}:</b><br />
        This word isn't in Wordle's dictionary. It can still be the Curdle
        answer, but <i>all</i> guesses will be allowed, making it significantly
        more difficult. Make sure to give your player a clue!
      </li>
    </ul>
  </div>
</template>

<script setup lang="ts">
import { indications } from "./words"
import dictionary from "./dictionary.json"
</script>

<style scoped>
div {
  text-align: left;
  position: relative;
  padding: 0.5em;
}

button {
  float: right;
}
</style>
