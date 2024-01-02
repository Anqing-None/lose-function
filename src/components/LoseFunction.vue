<template>
  <div style="width: 100%; height: 100%; display: flex; align-items: center; justify-content: center; flex-direction: column;">
    <div style="display: flex">
      <div ref="div" style="width: 500px; height: 500px"></div>
      <div ref="div3D" style="width: 500px; height: 500px"></div>
    </div>

    <div>
      <div>w: {{ w }}, b: {{ b }}</div>
      <div>error: {{ error }}</div>
      <div>
        <math-field readonly>{{ latex }}</math-field>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, watchEffect, onMounted, nextTick, watch, computed } from 'vue';
import * as echarts from 'echarts';
import 'echarts-gl';
import points from '@/assets/data/points.json';
import { MathfieldElement } from 'mathlive';
import { debounce, round } from 'lodash-es';

const mfe = new MathfieldElement();

const div = ref<HTMLElement>();
const div3D = ref<HTMLElement>();

const isDark = ref(true);
let pointChart;
let threeDChart;
const symbolSize = 16;

const w = ref(1);
const b = ref(0);

const latex = computed(() => {
  return `{{\\color{Blue} f_{w,b}} (x)} = {\\color{Blue} ${w.value}} x + {\\color{Blue} ${b.value}}`;
});

const error = computed(() => {
  const w1 = w.value;
  const b1 = b.value;
  let error = 0;
  points.forEach((p) => {
    const { x, y } = p;

    const yhat = w1 * x + b1;

    error += Math.pow(yhat - y, 2);
  });

  const ret = error / points.length;
  return round(ret, 2);
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

function getPointChartOption() {
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
  pointChart = echarts.init(div.value);
  threeDChart = echarts.init(div3D.value);

  const pointChartOption = getPointChartOption();
  pointChartOption.series.push(pointSeries);
  pointChartOption.series.push(lineSeries);

  pointChart.setOption(pointChartOption);

  const setPointGraphic = () => {
    pointChart.setOption({
      graphic: lineData.value.map((item, dataIndex) => {
        return {
          type: 'circle',
          position: pointChart.convertToPixel('grid', item),
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
            const p = pointChart.convertFromPixel('grid', pos);

            if (p[0] >= 26 || p[1] >= 26) {
              const p0 = p[0] >= 26 ? 25 : p[0];
              const p1 = p[1] >= 26 ? 25 : p[1];

              lineData.value[dataIndex] = [p0, p1];
              setPointGraphic();
              return;
            }

            if (p[0] <= -1 || p[1] <= -1) {
              const p0 = p[0] <= -1 ? 0 : p[0];
              const p1 = p[1] <= -1 ? 0 : p[1];

              lineData.value[dataIndex] = [p0, p1];
              setPointGraphic();
              return;
            }

            lineData.value[dataIndex] = p;

            const series = [pointSeries, lineSeries];

            pointChart.setOption({
              series,
            });
          },
          z: 100,
        };
      }),
    });
  };

  setPointGraphic();

  const threeDChartOption = {
    tooltip: {},
    backgroundColor: '#fff',
    visualMap: {
      show: false,
      dimension: 2,
      min: -1,
      max: 1,
      inRange: {
        color: ['#313695', '#4575b4', '#74add1', '#abd9e9', '#e0f3f8', '#ffffbf', '#fee090', '#fdae61', '#f46d43', '#d73027', '#a50026'],
      },
    },
    xAxis3D: {
      type: 'value',
    },
    yAxis3D: {
      type: 'value',
    },
    zAxis3D: {
      type: 'value',
    },
    grid3D: {
      viewControl: {
        // projection: 'orthographic'
      },
    },
    series: [
      {
        type: 'surface',
        wireframe: {
          // show: false
        },
        equation: {
          x: {
            step: 0.05,
          },
          y: {
            step: 0.05,
          },
          z: function (x, y) {
            if (Math.abs(x) < 0.1 && Math.abs(y) < 0.1) {
              return '-';
            }
            return Math.sin(x * Math.PI) * Math.sin(y * Math.PI);
          },
        },
      },
    ],
  };

  threeDChart.setOption(threeDChartOption);
});
</script>

<style scoped></style>
