<template>
  <div class="wrapper">
    <el-row class="top-box" align="middle">
      <span style="margin-right: 12px">时间删选</span>
      <el-time-picker
        v-model="currentTime"
        is-range
        range-separator="至"
        start-placeholder="开始时间"
        end-placeholder="结束时间"
        style="max-width: 300px; margin-right: 12px"
      >
      </el-time-picker>
      <span style="margin-right: 12px">通道设置</span>
      <el-select
        v-model="currentLead"
        placeholder="请选择导联"
        style="width: 300px"
      >
        <el-option
          v-for="item in seleteLists"
          :key="item.value"
          :label="item.label"
          :value="item.value"
        />
      </el-select>
      <el-button style="margin-left: auto" type="primary" :icon="Edit">
        测量波形时间
      </el-button>
      <el-button type="primary" :icon="Edit">测量振幅高度</el-button>
    </el-row>
    <Echarts></Echarts>
    <el-alert
      class="bottom-box"
      type="success"
      effect="light"
      :closable="false"
    >
      <el-row align="middle" class="alert-contents">
        <span>波形测量结果：</span>
        <div>Rva/69ms</div>
        <el-button
          style="margin-left: auto"
          type="success"
          :icon="Document"
          round
        >
          保存测量记录
        </el-button>
      </el-row>
    </el-alert>
  </div>
</template>

<script setup lang="ts">
import { Edit, Document } from "@element-plus/icons-vue";
import Echarts from "./Echarts.vue";
const currentLead = ref(12);
const currentTime = ref([]);
const seleteLists = ref([
  {
    value: 5,
    label: 5,
  },
  {
    value: 10,
    label: 10,
  },
  {
    value: 12,
    label: 12,
  },
]);
</script>

<style scoped lang="scss">
.wrapper {
  padding: 20px;
  background-color: #fff;
  height: 100vh;

  .top-box {
  }

  .bottom-box {
    ::v-deep(.el-alert__content) {
      width: 100%;
    }

    .alert-contents {
      span {
        font-weight: 600;
      }
      div {
        font-weight: 600;
        color: #111;
      }
    }
  }
}
</style>
