<script lang="ts" setup>
import axios from "axios"
import { CandlestickChart } from "echarts/charts"
import { DataZoomComponent, GridComponent, TitleComponent, TooltipComponent } from "echarts/components"
import * as echarts from "echarts/core"
import { CanvasRenderer } from "echarts/renderers"
import { onBeforeUnmount, onMounted, ref } from "vue"

echarts.use([
  CandlestickChart,
  GridComponent,
  TooltipComponent,
  TitleComponent,
  DataZoomComponent,
  CanvasRenderer
])

const chartRef = ref<HTMLDivElement | null>(null)
let chartInstance: echarts.ECharts | null = null

const symbol = ref("sz000001") // 初始默认股票代码

// 数据结构定义
const data: [number, number, number, number][] = []
const categoryData: string[] = []

const option: echarts.EChartsCoreOption = {
  tooltip: { trigger: "axis", axisPointer: { type: "cross" } },
  xAxis: {
    type: "category",
    data: categoryData,
    scale: true,
    boundaryGap: false,
    axisLine: { onZero: false },
    splitLine: { show: false },
    min: "dataMin",
    max: "dataMax"
  },
  yAxis: {
    scale: true,
    splitArea: { show: true }
  },
  dataZoom: [
    { type: "inside", start: 50, end: 100 },
    { show: true, type: "slider", top: "90%", start: 50, end: 100 }
  ],
  series: [
    {
      type: "candlestick",
      name: "K线",
      data,
      itemStyle: {
        color: "#ec0000",
        color0: "#00da3c",
        borderColor: "#8A0000",
        borderColor0: "#008F28"
      }
    }
  ]
}

// 请求数据函数，传入symbol参数
async function fetchKLineData(sym: string) {
  if (!sym) return
  try {
    const res = await axios.get(
      "api/sina/quotes_service/api/json_v2.php/CN_MarketData.getKLineData",
      {
        params: {
          symbol: sym,
          scale: 60,
          datalen: 200
        }
      }
    )
    const klineArr = res.data
    if (Array.isArray(klineArr)) {
      const newCategoryData: string[] = []
      const newData: [number, number, number, number][] = []
      klineArr.forEach((item: any) => {
        const timeLabel = item.day.length > 10 ? item.day : `${item.day} 15:00`
        newCategoryData.push(timeLabel)
        newData.push([
          Number.parseFloat(item.open),
          Number.parseFloat(item.close),
          Number.parseFloat(item.low),
          Number.parseFloat(item.high)
        ])
      })
      // 更新图表数据
      option.xAxis = { ...(option.xAxis || {}), data: newCategoryData }
      option.series = [{ ...((option.series as any[])?.[0] || {}), data: newData }]

      chartInstance?.setOption(option, true)
    }
  } catch (e) {
    console.error("获取K线数据失败", e)
  }
}

// 初始化图表和监听
onMounted(() => {
  if (chartRef.value) {
    chartInstance = echarts.init(chartRef.value)
    chartInstance.setOption(option)
    window.addEventListener("resize", resizeChart)
  }
  fetchKLineData(symbol.value) // 初次加载
})

function resizeChart() {
  chartInstance?.resize()
}

onBeforeUnmount(() => {
  window.removeEventListener("resize", resizeChart)
  chartInstance?.dispose()
})

// 输入框变化时触发
function onSymbolChange() {
  console.log("当前 symbol 是：", symbol.value)
  fetchKLineData(symbol.value)
}
</script>

<template>
  <div style="display: flex;">
    <!-- 左侧股票列表 -->
    <div style="width: 10%; height: 100%; margin-right: 24px;">
      <el-menu
        default-active="sz000002"
        @select="(key) => { symbol = key; onSymbolChange(); }"
        style="height: 500px;"
      >
        <el-menu-item index="sz000002">
          平安银行 (sz000002)
        </el-menu-item>
        <el-menu-item index="sh600519">
          贵州茅台 (sh600519)
        </el-menu-item>
        <el-menu-item index="sz002475">
          立讯精密 (sz002475)
        </el-menu-item>
        <el-menu-item index="sh601318">
          中国平安 (sh601318)
        </el-menu-item>
        <el-menu-item index="sh600036">
          招商银行 (sh600036)
        </el-menu-item>
      </el-menu>
    </div>
    <!-- 右侧输入和K线图 -->
    <div style="flex: 1;">
      <div style="display: flex; justify-content: center; margin-bottom: 16px;">
        <el-input
          v-model="symbol"
          placeholder="请输入股票代码（如 sz000001）"
          clearable
          style="width: 300px;"
          prefix-icon="el-icon-search"
          @change="onSymbolChange"
        />
      </div>
      <div ref="chartRef" style="width: 100%; height: 500px;" />
    </div>
  </div>
</template>
