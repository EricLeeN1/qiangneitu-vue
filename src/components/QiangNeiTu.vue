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
        format="HH:mm"
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
      <el-button type="primary" :icon="Edit" @click="getAmplitude">
        测量振幅高度
      </el-button>
    </el-row>
    <!-- <Echarts></Echarts> -->
    <CanvasDom
      ref="canvasDomRef"
      @updateClickResult="updateClickResult"
    ></CanvasDom>
    <el-row class="slide-wrapper">
      <el-slider
        v-model="timeSlide"
        style="width: 100%"
        range
        :min="0"
        :max="1440"
        :step="1"
        :format-tooltip="formatTooltip"
        :marks="marks"
        @change="handleSlideChange"
      />
    </el-row>
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
const timeSlide = ref([0, 1439]);

const marks = reactive({
  0: "00:00",
  720: "12:00",
  1439: "23:59",
});

// 将分钟数转换为时间字符串（HH:mm）
const formatTime = (minutes) => {
  const hours = Math.floor(minutes / 60);
  const mins = minutes % 60;
  return `${String(hours).padStart(2, "0")}:${String(mins).padStart(2, "0")}`;
};

("12:00");
const getTimeInfos = (times, isStart = true) => {
  const timesArr = times.split(":");
  console.log(timesArr);

  const day = dayjs().format("YYYY-MM-DD");
  console.log(
    `${day} ${timesArr[0] - 0}:${timesArr[1] - 0}:${isStart ? 0 : 59}`
  );

  return dayjs(
    `${day} ${timesArr[0] - 0}:${timesArr[1] - 0}:${isStart ? 0 : 59}`
  );
};

// 处理时间范围变化
const handleSlideChange = (value) => {
  console.log("选择的时间范围:", value);
  console.log("开始时间:", formatTime(value[0]));
  console.log("结束时间:", formatTime(value[1]));
  const time1 = getTimeInfos(formatTime(value[0]));
  const time2 = getTimeInfos(formatTime(value[1]), false);
  // 同步选择器时间
  currentTime.value = [time1, time2];
  // 同时更新画布
  drawCanvas();
};
// 格式化滑块提示
const formatTooltip = (value) => {
  return formatTime(value);
};

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

const getAmplitude = () => {
  canvasDomRef.value?.getMaxAndMin();
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

  .slide-wrapper {
    width: 1002px;
    padding-left: 51px;
    padding-right: 51px;
    margin-bottom: 20px;
  }

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
