<template>
  <div class="container">
    <div class="score-area score">
      {{ score >= 0 ? "Очки:" : "" }}
      <code v-if="score >= 0" class="score-value">{{ score }}</code>
    </div>
    <div class="game-area">
      <v-grid />
    </div>
    <div class="score-area scoreboard"></div>
  </div>
</template>

<script lang="ts">
import VGrid from "@/components/Grid.vue";

export default {
  name: "Playground",
  components: {
    VGrid,
  },
  props: {
    score: {
      type: Number,
      required: false,
    },
  },
};
</script>

