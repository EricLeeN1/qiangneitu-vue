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
        format="HH:mm:ss"
        style="max-width: 300px; margin-right: 12px"
        @change="changeTime"
      >
      </el-time-picker>
      <span style="margin-right: 12px">导联设置</span>
      <el-select
        v-model="currentLeads"
        placeholder="请选择导联"
        multiple
        collapse-tags
        collapse-tags-tooltip
        style="width: 300px"
        @blur="changeLeads"
      >
        <el-option
          v-for="item in leadsArr"
          :key="item"
          :label="item"
          :value="item"
        />
      </el-select>
      <el-button
        style="margin-left: auto"
        type="primary"
        :icon="Edit"
        @click="getClickTime"
      >
        测量波形时间
      </el-button>
      <el-button type="primary" :icon="Edit">测量振幅高度</el-button>
    </el-row>
    <!-- <Echarts></Echarts> -->
    <CanvasDom
      ref="canvasDomRef"
      @updateClickResult="updateClickResult"
    ></CanvasDom>
    <el-alert
      class="bottom-box"
      type="success"
      effect="light"
      :closable="false"
    >
      <el-row align="middle" class="alert-contents">
        <span>波形测量结果：{{ clickResult || "暂无数据" }}</span>
        <div></div>
        <el-button
          style="margin-left: auto"
          type="success"
          :icon="Document"
          round
          @click="saveExcel"
        >
          保存测量记录
        </el-button>
      </el-row>
    </el-alert>
  </div>
</template>

<script setup lang="ts">
import dayjs from "dayjs";
import { Edit, Document } from "@element-plus/icons-vue";
import { leadsArr } from "./const";
// import Echarts from "./Echarts.vue";
import CanvasDom from "./Canvas.vue";
const canvasDomRef: any = ref();
const clickResult: any = ref("");
const currentLeads: any = ref([...leadsArr]);
const currentTime: any = ref([dayjs().startOf("day"), dayjs().endOf("day")]);
const currentTimeFormate = computed(() => [
  dayjs(currentTime.value[0]).format("YYYY-MM-DD HH:mm:ss"),
  dayjs(currentTime.value[1]).format("YYYY-MM-DD HH:mm:ss"),
]);
const timeArr = ref([]);

/**
 * 根据 [startTime, endTime] 生成每隔一小时的时间数组
 * @param {string} startTime - 开始时间（格式：YYYY-MM-DD HH:mm:ss）
 * @param {string} endTime - 结束时间（格式：YYYY-MM-DD HH:mm:ss）
 * @returns {string[]} - 每隔一小时的时间数组
 */
const generateHourlyTimeSlots = (startTime, endTime) => {
  const timeSlots: any = [];
  let currentTime = dayjs(startTime); // 初始化当前时间为 startTime

  // 循环直到当前时间超过 endTime
  while (currentTime.isBefore(endTime)) {
    // timeSlots.push(currentTime.format("YYYY-MM-DD HH:mm:ss")); // 将当前时间加入数组
    timeSlots.push(currentTime.format("HH:mm:ss")); // 将当前时间加入数组
    currentTime = currentTime.add(1, "minutes"); // 增加一分钟
  }

  return timeSlots;
};

const getClickTime = () => {
  canvasDomRef.value?.getClickTime();
};

const saveExcel = () => {
  canvasDomRef.value?.saveExcel("测量结果.xlsx");
};

const changeTime = () => {
  console.log(currentTimeFormate.value, "currentTime.value");
  drawCanvas();
};
const changeLeads = () => {
  console.log(currentLeads.value, "currentLeads.value");
  drawCanvas();
};

const updateClickResult = ({ channel, data }) => {
  console.log(channel, data);
  clickResult.value = `${channel}/${data}`;
};

const drawCanvas = () => {
  timeArr.value = generateHourlyTimeSlots(
    currentTime.value[0],
    currentTime.value[1]
  );
  console.log(timeArr.value);
  nextTick(() => {
    canvasDomRef.value?.init({
      times: [...timeArr.value],
      currentTime: [...currentTimeFormate.value],
      leads: [...currentLeads.value],
    });
  });
};

onMounted(() => {
  drawCanvas();
});
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
