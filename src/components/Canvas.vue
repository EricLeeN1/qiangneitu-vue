<template>
  <div class="canvas-wrapper">
    <canvas
      id="iegmCanvas"
      ref="canvasRef"
      width="1000"
      height="722"
      @click="canvasClick"
      @mousedown="canvasMouseDown"
      @mousemove="canvasMouseMove"
      @mouseup="canvasMouseUp"
    />
    <!-- Element Plus 的 Popover -->
    <el-popover
      v-if="state.showPopover"
      popper-class="popover-canvas"
      :visible="state.showPopover"
      :popper-style="{
        top: state.popoverPosition.top,
        left: state.popoverPosition.left,
      }"
      virtual-triggering
      width="300px"
      @close="state.showPopover = false"
    >
      <div>
        <p>点击信息: {{ state.info }}</p>
      </div>
    </el-popover>
  </div>
</template>

<script setup lang="ts">
import { ElMessage } from "element-plus";
import { leadsArr } from "./const";
import * as XLSX from "xlsx";
const canvasRef: any = ref(null);
let ctx: any = ref();
const infoDiv = document.getElementById("info");
const clickInfoDiv = document.getElementById("click-info");
const selectBox = document.getElementById("selectBox");
const popup: any = document.getElementById("popup");
const clearBtn: any = document.getElementById("clearBtn");
const emits = defineEmits(["updateClickResult"]);

const state: any = reactive({
  speed: 2, // 波形的移动速度
  padding: 50, // 画布边距
  times: [], // 时间数组
  totalTime: 0, // 时间数组长度（ms）
  initHeight: 722, // canvas初始化高度（px）
  timeScale: "", // 时间缩放比例,
  channelCount: 5, // 要绘制的腔内电图数量
  channelSpacings: "", // 每个导联的间距比例
  channelSpacing: "", // 每个腔内电图之间的垂直间距
  offset: 0, // 波形偏移量
  iegmData: [], // 存储每个腔内电图的数据
  iegmNames: leadsArr, // 存储每个腔内电图的名称
  isDraggingChannel: false, // 是否正在拖拽调整导联间距
  draggedChannelIndex: -1, // 当前拖拽的导联索引
  verticalLines: [], // 存储竖线的位置
  isFirstLine: true, // 标记当前是第一条还是第二条竖线
  initialMouseY: 0, // 拖拽起始Y坐标
  initialSpacing: 0, // 拖拽起始间距
  lastClick: {
    channel: null,
    time: null,
  },
  longPressTimer: null,
  showPopover: false, // 是否显示弹出框
  popoverPosition: {
    x: 0,
    y: 0,
  }, // 弹框位置
  info: "", // 点位信息
});

const currentChannel = computed(() => {
  return state.lastClick.channel;
});

const finallyVerticalLines = computed(() => {
  if (state.verticalLines.length == 2) {
    const num1 = Math.round(
      (state.verticalLines[0] - state.padding) / state.timeScale
    );
    const num2 = Math.round(
      (state.verticalLines[1] - state.padding) / state.timeScale
    );
    return num2 > num1 ? [num1, num2] : [num2, num1];
  } else if (state.verticalLines.length == 1) {
    return [
      Math.round((state.verticalLines[0] - state.padding) / state.timeScale),
    ];
  } else {
    return [];
  }
});

// 生成随机噪声，用于模拟腔内电图的不规则性
const generateNoise = (scale) => {
  return (Math.random() * 2 - 1) * scale;
};

// 初始化数据
const initDatas = () => {
  for (let i = 0; i < state.channelCount; i++) {
    for (let t = 0; t < state.totalTime; t++) {
      // 基础波形（正弦波）
      const baseY = Math.sin(((t + state.offset) * Math.PI) / 40) * 20;
      // 添加随机噪声
      const noise = generateNoise(5);
      const y = baseY + noise;
      // 存储当前点的数据
      state.iegmData[i][t] = y;
    }
  }
  state.offset += state.speed;
};

// 绘制X轴（时间轴）
const drawXAxis = () => {
  ctx.value.strokeStyle = "#000";
  ctx.value.lineWidth = 1;
  ctx.value.beginPath();
  ctx.value.moveTo(state.padding, canvasRef.value.height - state.padding); // X轴起点
  ctx.value.lineTo(
    canvasRef.value.width - state.padding,
    canvasRef.value.height - state.padding
  ); // X轴终点
  ctx.value.stroke();

  // 添加时间刻度
  const timeInterval = 100; // 时间间隔
  for (let t = 0; t <= state.totalTime; t += timeInterval) {
    const x = state.padding + t * state.timeScale;
    ctx.value.beginPath();
    ctx.value.moveTo(x, canvasRef.value.height - state.padding);
    ctx.value.lineTo(x, canvasRef.value.height - state.padding + 5);
    ctx.value.stroke();

    let time = state.times[t];
    // 显示时间刻度值
    if (time) {
      ctx.value.fillText(
        `${time}`,
        x - 10,
        canvasRef.value.height - state.padding + 20
      );
    }
  }
};

// 绘制Y轴（因素名称）
const drawYAxis = () => {
  ctx.value.strokeStyle = "#000";
  ctx.value.lineWidth = 1;
  ctx.value.beginPath();
  ctx.value.moveTo(state.padding, state.padding); // Y轴起点
  ctx.value.lineTo(state.padding, canvasRef.value.height - state.padding); // Y轴终点
  ctx.value.stroke();
  ctx.value.font = "10px Arial";

  for (let i = 0; i < state.channelCount; i++) {
    const y = getChannelY(i) + getChannelHeight() / 2;
    ctx.value.fillText(state.iegmNames[i], 2, y);
  }
};

// 获取导联的Y坐标
const getChannelY = (index) => {
  let y = state.padding;
  for (let i = 0; i < index; i++) {
    y += getChannelHeight() + state.channelSpacings[i] * 20; // 20 是基础间距
  }
  return y;
};

// 获取导联的高度
const getChannelHeight = () => {
  return state.channelSpacing - 20; // 固定高度
};

// 绘制导联波形
const drawChannels = () => {
  ctx.value.strokeStyle = "#00ff00";
  ctx.value.lineWidth = 2;

  for (let i = 0; i < state.channelCount; i++) {
    // console.log(isDraggingChannel, draggedChannelIndex);

    if (state.isDraggingChannel && state.draggedChannelIndex !== -1) {
      // 正在拖拽时
      if (i === state.draggedChannelIndex) {
        ctx.value.strokeStyle = "#f0f000";
      } else {
        ctx.value.strokeStyle = "#00ff00";
      }
    }
    ctx.value.beginPath();
    ctx.value.moveTo(state.padding, getChannelY(i) + getChannelHeight() / 2); // 从左侧中间开始绘制

    for (let t = 0; t < state.totalTime; t++) {
      const x = state.padding + t * state.timeScale;
      const y = state.iegmData[i][t];
      ctx.value.lineTo(x, getChannelY(i) + getChannelHeight() / 2 + y);
    }

    ctx.value.stroke();
  }
};

// 绘制竖线
const drawVerticalLines = () => {
  // 绘制竖线
  ctx.value.strokeStyle = "#ff0000";
  ctx.value.lineWidth = 1;
  state.verticalLines.forEach((line) => {
    ctx.value.beginPath();
    ctx.value.moveTo(line, state.padding);
    ctx.value.lineTo(line, canvasRef.value.height - state.padding);
    ctx.value.stroke();
  });

  // 如果存在两条竖线，绘制带箭头的横线并显示差异
  if (state.verticalLines.length === 2) {
    const x1 = state.verticalLines[0];
    const x2 = state.verticalLines[1];
    const index = state.iegmNames.findIndex(
      (name) => name === state.lastClick.channel
    );
    const y = getChannelY(index) + getChannelHeight() / 2;

    const midX = (x1 + x2) / 2;

    // 绘制带箭头的横线
    ctx.value.strokeStyle = "#0000ff";
    ctx.value.lineWidth = 2;
    ctx.value.beginPath();
    ctx.value.moveTo(x1, y);
    ctx.value.lineTo(x2, y);
    ctx.value.stroke();

    // 绘制左箭头
    ctx.value.beginPath();
    ctx.value.moveTo(x1, y);
    ctx.value.lineTo(x1 + 10, y - 5);
    ctx.value.lineTo(x1 + 10, y + 5);
    ctx.value.closePath();
    ctx.value.fillStyle = "#0000ff";
    ctx.value.fill();

    // 绘制右箭头
    ctx.value.beginPath();
    ctx.value.moveTo(x2, y);
    ctx.value.lineTo(x2 - 10, y - 5);
    ctx.value.lineTo(x2 - 10, y + 5);
    ctx.value.closePath();
    ctx.value.fillStyle = "#0000ff";
    ctx.value.fill();

    // 计算并显示差异
    const t1 = finallyVerticalLines.value[0];
    const t2 = finallyVerticalLines.value[1];
    const y1 =
      Math.sin(((t1 + state.offset) * Math.PI) / 40) * 20 + generateNoise(5);
    const y2 =
      Math.sin(((t2 + state.offset) * Math.PI) / 40) * 20 + generateNoise(5);
    const diff = (y2 - y1).toFixed(2);

    ctx.value.fillStyle = "#0000ff";
    ctx.value.fillText(`Δ: ${diff}`, midX - 20, y - 10);
  }
};

// 绘制Canvas内容
const drawCanvas = () => {
  ctx.value.clearRect(0, 0, canvasRef.value.width, canvasRef.value.height);

  // 绘制背景
  // ctx.value.fillStyle = '#0000ff';
  // ctx.value.fillRect(0, 0, canvasRef.value.width, canvasRef.value.height);

  // 绘制坐标轴
  drawXAxis();
  drawYAxis();

  // 绘制导联波形
  drawChannels();

  // requestAnimationFrame(drawChannels);

  // 绘制竖线
  drawVerticalLines();
};

const getClickTime = () => {
  console.log("点击了吗");

  let message = "";
  let type = "success";
  const length = state.verticalLines.length;

  if (!length) {
    message = "请先点击获取时间";
    type = "warning";
  } else if (length === 1) {
    const time = state.times[finallyVerticalLines.value[0]];
    message = `测量波形的时间是${time}`;
  } else if (length === 2) {
    const time1 = state.times[finallyVerticalLines.value[0]];
    const time2 = state.times[finallyVerticalLines.value[1]];
    message = `测量波形的时间是${time1}-${time2}`;
  }

  console.log(message, type);
  ElMessage({
    message,
    type,
  });
};

// 隐藏弹窗
const hidePopup = () => {
  popup.style.display = "none";
};

const initCloseButton = () => {
  const dom = document.createElement("div"); // 添加dom
  dom.setAttribute("class", "close-btn");
  dom.innerText = "X";
  dom.addEventListener("click", () => {
    hidePopup();
  });
  return dom;
};

// 显示弹窗
const showPopup = (content, x, y) => {
  const closeBtn = initCloseButton();

  popup.innerHTML = content;
  popup.style.display = "block";
  const width = popup.offsetWidth;
  popup.style.left = `${x + width + 10}px`;
  popup.style.top = `${y}px`;
  popup.appendChild(closeBtn);
};

// 显示竖线上的数据
const showData = (mouseX, mouseY) => {
  if (state.verticalLines.length === 0) return;

  let info = "";

  if (state.verticalLines.length === 1) {
    const x = finallyVerticalLines.value[0];
    console.log(x);
    const time = state.times[x]; // 计算时间点

    info += `时间: ${time}\n`;
    for (let i = 0; i < state.channelCount; i++) {
      info += `${state.iegmNames[i]}: ${state.iegmData[i][x].toFixed(2)}\n`;
    }
  } else if (state.verticalLines.length === 2) {
    const x1 = finallyVerticalLines.value[0];
    const x2 = finallyVerticalLines.value[1];
    const time1 = state.times[x1];
    const time2 = state.times[x2];
    console.log(x1, x2);
    info += `时间范围: ${time1} 到 ${time2}\n`;
    for (let i = 0; i < state.channelCount; i++) {
      const values = state.iegmData[i].slice(x1, x2 + 1);
      const avg = (values.reduce((a, b) => a + b, 0) / values.length).toFixed(
        2
      );
      info += `${state.iegmNames[i]}: 平均数 = ${avg}\n`;
    }
  }

  state.info = info;

  console.log(mouseX, mouseY);

  // 设置 Popover 的位置
  state.popoverPosition = {
    top: `${mouseY}px`,
    left: `${mouseX}px`,
  };

  // 显示 Popover
  state.showPopover = true;

  // showPopup(info, mouseX, mouseY);

  // state.infoDiv.textContent = info;
};

// 获取点击位置对应的导联和时间
const getClickInfo = (x, y) => {
  const channelIndex = Math.floor((y - 20) / state.channelSpacing);
  const time = ((x - 50) / (canvasRef.value.width - 100)) * state.totalTime;
  return { channel: state.iegmNames[channelIndex], time };
};

const showClickInfos = (x, y) => {
  const { channel, time } = getClickInfo(x, y);
  // console.log(`点击导联: ${channel}, 时间: ${time}`);

  if (state.lastClick.channel === channel) {
    let info = "";

    const minTime = Math.min(state.lastClick.time, time);
    const maxTime = Math.max(state.lastClick.time, time);

    // console.log(`时间范围是: ${minTime.toFixed(2)}ms - ${maxTime.toFixed(2)}ms`);

    const channelIndex = state.iegmNames.indexOf(channel);
    const channelData = state.iegmData[channelIndex];
    // console.log(`导联 Index: ${channelIndex}`);

    const startIndex = Math.floor(minTime);
    const endIndex = Math.floor(maxTime);
    // console.log(`导联开始与结束时间取整: ${startIndex} ${endIndex}`);

    const dataSlice = channelData.slice(startIndex, endIndex + 1);
    const maxValue = Math.max(...dataSlice);
    const minValue = Math.min(...dataSlice);

    info += `当前导联: ${channel}, 时间范围: ${minTime.toFixed(
      2
    )}ms - ${maxTime.toFixed(2)}ms\n`;
    info += `最大值: ${maxValue}, 最小值: ${minValue}\n`;

    // state.clickInfoDiv.textContent = info;
  }

  state.lastClick = { channel, time };
};

// 点击事件处理
const canvasClick = (event) => {
  if (state.isDraggingChannel) {
    return false;
  }
  const rect = canvasRef.value.getBoundingClientRect();
  const mouseX = event.clientX - rect.left; // 点击的X坐标
  const mouseY = event.clientY - rect.top;

  // 检查点击是否在绘图区域内
  if (mouseX < state.padding || mouseX > canvasRef.value.width - state.padding)
    return;

  showClickInfos(mouseX, mouseY);
  const channelIndex = state.iegmNames.indexOf(state.lastClick.channel);
  const channelData = state.iegmData[channelIndex][mouseX];
  const clickDatas = {
    channel: state.lastClick.channel,
    data: channelData,
  };
  emits("updateClickResult", clickDatas);

  // 添加或替换竖线
  if (state.isFirstLine) {
    state.verticalLines[0] = mouseX;
    state.isFirstLine = false;
  } else {
    state.verticalLines[1] = mouseX;
    state.isFirstLine = true;
  }

  // 如果超过两条竖线，替换第一条
  if (state.verticalLines.length > 2) {
    state.verticalLines.shift(); // 移除第一条竖线
  }

  // 重新绘制Canvas
  drawCanvas();

  // 显示数据
  showData(mouseX, mouseY);
};

// 长按拖拽调整导联间距
const canvasMouseDown = (event) => {
  console.log(event);

  const rect = canvasRef.value.getBoundingClientRect();
  console.log(rect, "rect");
  console.log(event.clientY, "event.clientY");

  const mouseY = event.clientY - rect.top; // 点击的Y坐标
  console.log(mouseY, "mouseY");
  // 检查点击是否在某个导联范围内
  for (let i = 0; i < state.channelCount; i++) {
    const y = getChannelY(i);
    const height = getChannelHeight();
    if (mouseY >= y && mouseY <= y + height) {
      state.draggedChannelIndex = i;
      state.initialMouseY = mouseY;
      state.initialSpacing = state.channelSpacings[i];
      // 长按时初始化其他导联缩放比例为1
      for (let j = 0; j < state.channelCount; j++) {
        if (j !== state.draggedChannelIndex) {
          state.channelSpacings[j] = 1;
        }
      }
      state.longPressTimer = setTimeout(() => {
        state.isDraggingChannel = true;
      }, 500); // 长按500ms后开始拖拽
      break;
    }
  }
};
const canvasMouseMove = (event) => {
  if (state.isDraggingChannel && state.draggedChannelIndex !== -1) {
    const rect = canvasRef.value.getBoundingClientRect();
    const mouseY = event.clientY - rect.top; // 拖拽的Y坐标

    // 计算新的导联间距
    const deltaY = Math.abs(mouseY - state.initialMouseY);
    state.channelSpacings[state.draggedChannelIndex] =
      state.initialSpacing + deltaY / 50; // 50 是缩放因子

    //   限制间距范围;
    //   channelSpacings[draggedChannelIndex] = Math.max(
    //     2,
    //     Math.min(2, channelSpacings[draggedChannelIndex])
    //   );
    console.log(state.channelSpacings);

    console.log(deltaY, state.channelSpacing);
    console.log(canvasRef.value.height);
    // 动态调整Canvas高度
    const totalSpacing = state.channelSpacings.reduce((a, b) => a + b, 0);
    state.canvasHeight = Math.max(
      state.initHeight,
      state.padding * 2 +
        getChannelHeight() * state.channelCount +
        totalSpacing * 20
    );
    canvasRef.value.height = state.canvasHeight;
    drawCanvas();
  }
};

const canvasMouseUp = () => {
  clearTimeout(state.longPressTimer);
  setTimeout(() => {
    state.isDraggingChannel = false;
    state.draggedChannelIndex = -1;
    // canvasRef.value.height = initHeight;
    // channelSpacings = new Array(channelCount).fill(1); // 每个导联的间距比例
    // 重新绘制Canvas
    drawCanvas();
  }, 200);
};

clearBtn?.addEventListener("click", () => {
  canvasRef.value.height = state.initHeight;
  state.channelSpacings = new Array(state.channelCount).fill(1); // 每个导联的间距比例
  // 重新绘制Canvas
  drawCanvas();
});

// 生成模拟数据
const generateData = () => {
  let data: any = [];
  const channels = state.iegmNames; // 12 个通道
  const minutes = state.times; // 时间数组
  for (let i = 0; i < channels.length; i++) {
    const row = {
      导联: channels[i],
    };
    for (let j = 0; j < minutes.length; j++) {
      row[minutes[j]] = state.iegmData[i][j]; // 生成随机数据
    }
    console.log(row, "row");
    data.push(row);
  }

  return data;
};

// 添加表头
const addHeader = (worksheet, header) => {
  const headerRange = XLSX.utils.decode_range(worksheet["!ref"]); // 获取当前工作表范围
  XLSX.utils.sheet_add_aoa(worksheet, [header], { origin: -1 }); // 在顶部插入表头
  worksheet["!ref"] = XLSX.utils.encode_range({
    s: { r: headerRange.s.r - 1, c: headerRange.s.c }, // 调整范围
    e: headerRange.e,
  });
};

// 将数据保存为 Excel 文件
const saveExcel = (filename) => {
  const data = generateData();

  console.log(data);
  // 将数据转换为工作表
  const worksheet = XLSX.utils.json_to_sheet(data);
  // 添加表头
  // try {
  //   addHeader(worksheet, state.iegmNames);
  // } catch (error) {
  //   console.log(error);
  // }
  // 创建工作簿并添加工作表
  const workbook = XLSX.utils.book_new();
  XLSX.utils.book_append_sheet(workbook, worksheet, "Sheet1");
  XLSX.writeFile(workbook, filename);
};

// 获取某一段数据的最高值和最低值
const getMinMaxInRange = (arr, startIndex, endIndex) => {
  // 检查输入是否有效
  if (startIndex < 0 || endIndex >= arr.length || startIndex > endIndex) {
    throw new Error("Invalid range");
  }

  // 提取指定范围内的数据
  const rangeData = arr.slice(startIndex, endIndex + 1);

  let max = rangeData[0];
  let min = rangeData[0];
  for (let i = 1; i < rangeData.length; i++) {
    if (rangeData[i] > max) max = rangeData[i];
    if (rangeData[i] < min) min = rangeData[i];
  }

  return { max, min };
};

const getMaxAndMin = () => {
  let message = "";
  let type = "success";
  if (state.verticalLines.length === 2) {
    const x1 = finallyVerticalLines.value[0];
    const x2 = finallyVerticalLines.value[1];
    const time1 = state.times[x1];
    const time2 = state.times[x2];
    const index = state.iegmNames.findIndex(
      (name) => name === state.lastClick.channel
    );
    const arr = state.iegmData[index]; // 最后一次点击的导联
    const { max, min } = getMinMaxInRange(arr, x1, x2);
    console.log(max, min);
    message = `导联 ${state.lastClick.channel} 在 ${time1} - ${time2} 范围内的最大值是 ${max}，最小值是 ${min}`;
  } else {
    message = "请先选择导联及时间范围";
    type = "error";
  }
  ElMessage({
    message,
    type,
  });
};

const init = ({ times, currentTime, leads }) => {
  console.log(times, currentTime, leads);
  ctx.value = canvasRef.value?.getContext("2d");
  state.iegmNames = leads;
  state.channelCount = leads.length;
  state.times = times;
  // 计算两个时间点之间的分钟差
  state.totalTime = times.length;
  console.log(canvasRef.value.width - 2 * state.padding);

  state.timeScale =
    (canvasRef.value.width - 2 * state.padding) / state.totalTime;
  state.channelSpacing =
    (canvasRef.value.height - 2 * state.padding) / state.channelCount; // 每个腔内电图之间的垂直间距

  state.channelSpacings = new Array(state.channelCount).fill(1); // 每个导联的间距比例
  // 初始化数据存储和名称
  state.iegmData = []; // 初始化数据
  for (let i = 1; i <= state.channelCount; i++) {
    state.iegmData.push([]);
  }
  initDatas();
  // 初始化Canvas
  drawCanvas();
};

// selectBox.addEventListener("change", (e) => {
//   state.channelCount = e.target.value - 0;
//   init();
// });
defineExpose({
  init,
  getClickTime,
  saveExcel,
  getMaxAndMin,
});
</script>

<style scoped lang="scss">
.canvas-wrapper {
  position: relative;
}

#iegmCanvas {
  border: 1px solid #ccc;
  margin: 10px 0;
}
</style>

<style lang="scss">
.popover-canvas {
  p {
    white-space: break-spaces;
  }
}
</style>
