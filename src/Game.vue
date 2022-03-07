<script setup lang="ts">
import { getWordOfTheDay } from "./words"
import Maker from "./Maker.vue"
import Board from "./Board.vue"
import About from "./About.vue"

const { word } = getWordOfTheDay()
const baseUrl = location.origin + location.pathname

let showingAbout = $ref(false)
function toggleAbout(event: Event) {
  showingAbout = !showingAbout
  event.preventDefault()
}
</script>

<template>
  <header>
    <a href="#about" id="about-link" @click="toggleAbout">
      {{ showingAbout ? "close" : "about" }}
    </a>
    <h1>CURDLE</h1>
    <h2><span>cu</span>stom wo<span>rdle</span></h2>
    <a v-if="word" id="source-link" :href="baseUrl">new</a>
  </header>
  <template v-if="showingAbout">
    <About></About>
  </template>
  <template v-else>
    <Board v-if="word" :answer="word"></Board>
    <Maker v-else></Maker>
  </template>
</template>
