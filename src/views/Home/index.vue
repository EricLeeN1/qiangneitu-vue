<template>
  <el-config-provider :locale="zhCn">
    <QiangNeiTu />
  </el-config-provider>
</template>

<script setup lang="ts">
import zhCn from "element-plus/dist/locale/zh-cn.mjs";
// This starter template is using Vue 3 <script setup> SFCs
// Check out https://v3.vuejs.org/api/sfc-script-setup.html#sfc-script-setup
import QiangNeiTu from "@/components/QiangNeiTu.vue";
</script>

<style></style>
