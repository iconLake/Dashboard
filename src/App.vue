<script lang="ts" setup>
import { ref } from "vue";
import CreateValidator from "./components/CreateValidator.vue";
import Unjail from "./components/Unjail.vue";

const nav = ref("cv");

const setNav = (navType: string) => {
  nav.value = navType;
};
</script>

<template>
  <div class="app">
    <div class="nav">
      <div class="item" @click="setNav('cv')">Create Validator</div>
      <div class="item" @click="setNav('uj')">Unjail</div>
    </div>
    <div class="main">
      <div class="content">
        <CreateValidator v-if="nav === 'cv'" />
        <Unjail v-if="nav === 'uj'" />
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
.app {
  height: 100vh;
  width: 100vw;
  display: flex;
}
.nav {
  width: 180px;
  text-align: center;
  background: #f1f1f1;
  .item {
    padding: 16px;
    cursor: pointer;
    &:hover {
      background: #000;
      color: #fff;
    }
  }
}
.main {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  height: 100%;
  overflow: auto;
  text-align: center;
  .content {
    padding: 20px;
  }
}
</style>
