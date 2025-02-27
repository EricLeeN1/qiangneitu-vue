<template>
  <div id="canvas-dom" class="echarts-wrapper"></div>
</template>

<script setup lang="ts">
import * as echarts from "echarts";
const props = defineProps({
  title: {
    type: String,
    default: () => "",
  },
});
const leadsArr = [
  "I",
  "II",
  "III",
  "aVR",
  "aVL",
  "aVF",
  "V1",
  "V2",
  "V3",
  "V4",
  "V5",
  "V6",
];

const generateECGWaveform = (points) => {
  const data = [];
  let value = 0;
  for (let i = 0; i < points; i++) {
    if (i % 50 === 0) {
      value = 1.5 + Math.random() * 0.5; // R波峰
    } else if (i % 50 === 5) {
      value = -0.5 - Math.random() * 0.2; // S波谷
    } else if (i % 50 === 10) {
      value = 0.8 + Math.random() * 0.2; // T波
    } else {
      value = Math.random() * 0.1; // 噪声基线
    }
    data.push(value);
  }
  return data;
};

const seriesData = leadsArr.map((lead, index) => ({
  name: lead,
  type: "line",
  showSymbol: false,
  color: "#000000",
  data: generateECGWaveform(500).map((val) => val + index * 3), // 垂直偏移以分离通道
  emphasis: { focus: "series" },
  lineStyle: { width: 1.5 },
}));
const init = () => {
  const dom = document.getElementById("canvas-dom");
  console.log(dom);

  const chart = echarts.init(dom);
  chart.setOption({
    title: { text: "like Lines", left: "center" },
    tooltip: { trigger: "axis", axisPointer: { type: "cross" } },
    legend: { data: leadsArr, top: 30 },
    dataZoom: [
      {
        show: true,
        realtime: true,
        start: 30,
        end: 70,
        xAxisIndex: [0, 1],
      },
      {
        type: "inside",
        realtime: true,
        start: 30,
        end: 70,
        xAxisIndex: [0, 1],
      },
    ],
    grid: {
      left: "5%",
      right: "5%",
      top: "15%",
      bottom: "10%",
      containLabel: true,
    },
    xAxis: {
      type: "category",
      data: Array.from({ length: 500 }, (_, i) => i.toString()),
      axisLabel: { show: false },
      axisTick: { show: false },
      splitLine: { show: false },
    },
    yAxis: {
      type: "value",
      axisLine: { show: false },
      axisTick: { show: false },
      splitLine: { show: false },
      min: -2,
      max: leadsArr.length * 3,
    },
    series: seriesData,
  });
};
onMounted(() => {
  init();
});
defineExpose({
  init,
});
</script>

<style scoped lang="scss">
.echarts-wrapper {
  height: calc(100% - 104px);
  margin: 12px 0;
  border: 1px solid #eee;
}
</style>
