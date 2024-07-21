<template>
  <v-app>
    <v-main>
      <ChatMain/>
    </v-main>
  </v-app>
</template>

<script>
import ChatMain from './components/ChatMain.vue'

export default {
  name: 'App',

  components: {
    ChatMain,
  },

  data: () => ({
    //
  }),
}
</script>
