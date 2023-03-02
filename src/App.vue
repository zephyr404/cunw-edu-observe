<template>
  <ThreeView />
</template>

<script lang="ts">
import { Options, Vue } from "vue-class-component";
import ThreeView from "./components/ThreeView.vue";

@Options({
  components: {
    ThreeView,
  },
})
export default class App extends Vue {}
</script>

<style lang="scss">
@import "@/assets/style/config.scss";
@import "@/assets/style/mixins.scss";

html {
  @include respond-to(md) {
    font-size: 7px;
  }

  @include respond-to(lg) {
    font-size: 8px;
  }

  @include respond-to(mx) {
    font-size: 10px;
  }

  @include respond-to(mxx) {
    font-size: 20px;
  }

  @include respond-to(mxxx) {
    font-size: 20px;
  }
}

#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
  background-color: #ebf1f8;
}
</style>
