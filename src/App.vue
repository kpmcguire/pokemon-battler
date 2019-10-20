<template lang="pug">
  #app
    .bg-blue-800.py-2.text-center(class="md:py-10")
      img.w-32.block.m-auto(src="./assets/images/digimon.svg" class="md:w-64")
    battler
</template>

<script>
import battler from './components/battler.vue'
export default {
  name: 'app',
  components: {
    battler
  }
}
</script>

