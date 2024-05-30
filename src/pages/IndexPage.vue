<template>
  <q-page class="flex flex-center">
    <img alt="Quasar logo" src="~assets/sat.png" style="width: 200px; height: 200px">
  </q-page>
</template>

<script>
import { defineComponent } from 'vue'

export default defineComponent({
  name: 'IndexPage',
  setup() {
    return {

    }
  },

  mounted() {
    this.$router.push('/home')
    console.log("App mounted")
  }
})
</script>
