<template>
  <div ref="div" style="width: 500px; height: 500px"></div>
  <div>w: {{ w }}, b: {{ b }}</div>

  <div>
    <math-field>{{ latex }}</math-field>
  </div>
</template>

<script setup lang="ts">
import { ref, watchEffect, onMounted, nextTick, watch, computed } from 'vue';
import * as echarts from 'echarts';
import points from '@/assets/data/points.json';
import { MathfieldElement } from 'mathlive';
import { debounce, round } from 'lodash-es';

const mfe = new MathfieldElement();

const div = ref<HTMLElement>();

const isDark = ref(true);
let myChart;
const symbolSize = 16;

const w = ref(1);
const b = ref(0);

const latex = computed(() => {
  return `{{\\color{Blue} f_{w,b}} (x)} = {\\color{Blue} ${w.value}} x + {\\color{Blue} ${b.value}}`;
});

const pointsData = points.map((p) => {
  const { x, y } = p;
  return [x, y];
});

const pointSeries = {
  id: 'point',
  type: 'scatter',
  data: pointsData,
  symbolSize: 8,
};

const lineData = ref([
  [1, 1],
  [24, 24],
]);

watch(
  lineData,
  (newLineData) => {
    const [p1, p2] = newLineData;
    const { w: w1, b: b1 } = calcWB(p1, p2);

    w.value = round(w1, 2);
    b.value = round(b1, 2);
  },
  { deep: true, immediate: true },
);

const lineSeries = {
  id: 'line',
  type: 'line',
  symbolSize: symbolSize,
  data: lineData.value,
  lineStyle: {
    width: 4,
  },
};

function getOption() {
  const option = {
    title: {
      // text: 'Try Dragging these Points',
      // left: 'center',
    },
    tooltip: {
      // triggerOn: 'none',
      formatter: function (params) {
        return 'X: ' + params.data[0].toFixed(2) + '<br>Y: ' + params.data[1].toFixed(2);
      },
    },
    grid: {
      // top: '8%',
      // bottom: '12%',
    },
    xAxis: {
      min: 0,
      max: 25,
      type: 'value',
      // axisLine: { onZero: false },
    },
    yAxis: {
      min: 0,
      max: 25,
      type: 'value',
      // axisLine: { onZero: false },
    },
    series: [],
  };
  return option;
}

// onMounted(() => {
//   const mfe = new MathfieldElement();
//   mfe.setAttribute('read-only', 'true');
//   const latex = `{{\\color{Blue} f_{w,b}} (x)} = {\\color{Blue} w} x + {\\color{Blue} b}`;

//   mfe.value = latex;
//   mfe.style.display = 'inline-block';
//   mfe.style.fontSize = '24px';
//   formulaDiv.value.appendChild(mfe);
// });

function calcWB(p1, p2) {
  const deltaX = p1[0] - p2[0];
  const deltaY = p1[1] - p2[1];

  const w = deltaY / deltaX;

  const b = p1[1] - w * p1[0];

  return { w, b };
}

onMounted(() => {
  // 基于准备好的dom，初始化echarts实例
  myChart = echarts.init(div.value);

  const option = getOption();
  option.series.push(pointSeries);
  option.series.push(lineSeries);

  myChart.setOption(option);

  const setGraphic = () => {
    myChart.setOption({
      graphic: lineData.value.map((item, dataIndex) => {
        return {
          type: 'circle',
          position: myChart.convertToPixel('grid', item),
          shape: {
            cx: 0,
            cy: 0,
            r: symbolSize,
          },
          invisible: true,
          draggable: true,
          ondrag() {
            // @ts-ignore
            const pos = [this.x, this.y];
            const p = myChart.convertFromPixel('grid', pos);

            if (p[0] >= 26 || p[1] >= 26) {
              const p0 = p[0] >= 26 ? 25 : p[0];
              const p1 = p[1] >= 26 ? 25 : p[1];

              lineData.value[dataIndex] = [p0, p1];
              setGraphic();
              return;
            }

            if (p[0] <= -1 || p[1] <= -1) {
              const p0 = p[0] <= -1 ? 0 : p[0];
              const p1 = p[1] <= -1 ? 0 : p[1];

              lineData.value[dataIndex] = [p0, p1];
              setGraphic();
              return;
            }

            lineData.value[dataIndex] = p;

            const series = [pointSeries, lineSeries];

            myChart.setOption({
              series,
            });
          },
          z: 100,
        };
      }),
    });
  };

  setGraphic();
});
</script>

<style scoped></style>
