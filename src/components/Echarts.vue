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
  const data: any = [];
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

const datas = leadsArr.map((item, index) => {
  return generateECGWaveform(500).map((val) => val + index * 3); // 垂直偏移以分离通道
});

// 存储点击的竖线信息
let clickLines: any = [];
let clickCount = 0;

const seriesData: any = leadsArr.map((lead, index) => ({
  name: lead,
  type: "line",
  showSymbol: false,
  color: "#000000",
  data: datas[index],
  emphasis: { focus: "series" },
  lineStyle: { width: 1.5 },
}));
const init = () => {
  const dom = document.getElementById("canvas-dom");
  console.log(dom);

  const chart = echarts.init(dom);
  let option: any = {
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
  };
  chart.setOption(option);
  // 添加点击事件监听器
  chart.on("click", (params) => {
    console.log(params);
    const xAxisValue: any = Math.floor(params.dataIndex); // 获取点击的 x 轴值
    const seriesIndex: any = params.seriesIndex; // 获取点击的 x 轴值
    console.log(xAxisValue);
    console.log(clickLines);

    // 添加竖线
    const markLine = {
      animation: false,
      label: {
        show: true,
        formatter: "Line " + (clickCount + 1),
      },
      lineStyle: {
        type: "solid",
        color: "#ff0000",
      },
      data: [
        {
          xAxis: xAxisValue,
        },
      ],
    };
    // 更新配置项，添加竖线
    option.series[seriesIndex].markLine = markLine;
    chart.setOption(option);

    // 获取竖线上所有系列的数据
    const seriesData = datas.map((series) => series[params.dataIndex]);

    // 存储点击的竖线信息
    clickLines.push({
      xAxisValue: xAxisValue,
      seriesData: seriesData,
    });

    // 点击次数加1
    clickCount++;

    // 如果点击了两次
    if (clickCount === 2) {
      // 计算两条竖线之间的数据
      const firstLine = clickLines[0];
      const secondLine = clickLines[1];
      console.log(option.xAxis.data);

      let startIndex = option.xAxis.data.indexOf(
        firstLine.xAxisValue.toString()
      );
      let endIndex = option.xAxis.data.indexOf(
        secondLine.xAxisValue.toString()
      );
      console.log("startIndex", startIndex);
      console.log("endIndex", endIndex);
      if (startIndex > endIndex) {
        [startIndex, endIndex] = [endIndex, startIndex];
      }

      const rangeData = datas.map((series) =>
        series.slice(startIndex, endIndex + 1)
      );

      console.log("两条竖线之间的数据:", rangeData);

      // 重置点击次数和竖线信息
      clickCount = 0;
      clickLines = [];
    }
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
