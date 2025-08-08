<script setup>
defineProps({
  routeName: String,
  dest: String,
  depTime: String,
  timer: String,
});
</script>

<template>
  <article>
    <header>
      <mark v-if="routeName">{{ routeName }}</mark>
      {{ dest }}
    </header>
    <h3>{{ depTime }}</h3>
    <footer v-if="timer">{{ timer }}</footer>
    <footer v-else>当日の運行が終了しています。</footer>
  </article>
</template>
