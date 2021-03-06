<template>
  <div>
    <div>
      <header v-if="header" class="header clearfix" style="padding: 10px">
        <div class="title-wrapper float-left" v-if="header.title">
          <div class="title">{{ header.title }}</div>
          <div class="subtitle" v-if="header.subtitle">{{ header.subtitle }}</div>
        </div>
      </header>
      <base-chart :chartType="chartType" :chartData="chartData" :chartConfig="chartConfig" :chartSettings="chartSettings" :chartExtend="chartExtend" :loading="loading" :emptyContent="emptyContent" :chartEvents="chartEvents" />
    </div>
  </div>
</template>

<script>
// import { completionDate } from '@/utils/utils'
import { transformUnit } from '@/utils/utils'

export default {
  name: 'OverviewLine',
  props: {
    unit: {
      type: Object,
      default: () => ({}),
    },
    header: {
      type: Object,
      default: () => ({}),
    },
    chartData: {
      type: Object,
      required: true,
    },
    chartEvents: {
      type: Object,
      default: () => ({}),
    },
    isHistogram: { // 默认表示 Y 轴是数据轴，竖向
      type: Boolean,
      default: false,
    },
    loading: {
      type: Boolean,
      default: false,
    },
    height: {
      type: String,
      default: '100%',
    },
    chartSettings: {
      type: Object,
      default: () => ({}),
    },
    splitLineShow: { // 是否显示分隔线。默认数值轴显示，类目轴不显示。
      type: Boolean,
      default: false,
    },
    emptyContent: {},
    showLegend: { // 图例显示
      type: Boolean,
      default: false,
    },
    dateMode: {},
    numerifyFormat: {
      type: String,
      default: '0.00',
    },
  },
  data () {
    return {
      chartType: this.isHistogram ? 've-histogram' : 've-bar',
    }
  },
  computed: {
    chartExtend () {
      const unit = this.unit.unit
      const numberFormat = this.numerifyFormat
      // const chartType = this.chartType

      const commonSerie = {
        barWidth: '12px',
        barMaxWidth: '24px',
        barCategoryGap: '60%',
      }
      if (!this.isHistogram) {
        const unit = this.unit.unit
        commonSerie.label = {
          show: !this.isHistogram,
          normal: {
            position: 'right',
            show: true,
            color: '#939EAB',
            formatter: function (params) {
              if (unit) {
                const val = transformUnit(params.value, unit, 1000, numberFormat)
                return val.text
              } else {
                return params.value
              }
            },
          },
          emphasis: {
            color: '#4DA1FF',
          },
        }
      }
      const ce = {
        series (v) {
          if (v) return v.map(i => ({ ...i, ...commonSerie }))
          return []
        },
        xAxis: {
          boundaryGap: [0, 0.01],
          splitLine: {
            show: false,
          },
          axisTick: {
            show: false,
          },
          axisLine: {
            show: false,
          },
          axisLabel: {
            show: this.isHistogram,
          },
        },
        yAxis: {
          nameGap: 30,
          splitLine: {
            show: this.splitLineShow,
          },
          axisTick: {
            show: false,
          },
          axisLine: {
            show: false,
          },
          axisLabel: {
            formatter: (value, index) => {
              if (unit) {
                const val = transformUnit(value, unit, 1000, numberFormat)
                return val.text
              } else {
                return value
              }
            },
          },
          max: function (value) { // 坐标轴刻度最大值
            // if (chartType === 've-histogram') {
            //   return Math.ceil(value.max * 1.2)
            // }
            return value.max
          },
        },
        toolbox: {
          showTitle: false,
        },
      }
      if (!this.isHistogram) {
        ce.yAxis.axisLabel = {
          formatter (value, index) {
            if (!value) {
              return value
            }

            value = value.split('__::__')[1]
            if (value.length < 15) {
              return value
            }

            const prefix = value.slice(0, 6)
            const surfix = value.slice(value.length - 6)
            return `${prefix}...${surfix}`
          },
        }
      }
      return ce
    },
    chartConfig () {
      const unit = this.unit.unit
      const numberFormat = this.numberFormat
      const config = {
        // title,
        height: this.height,
        width: '100%',
        legend: {
          show: this.showLegend,
        },
        tooltip: {
          trigger: 'axis',
          axisPointer: {
            type: 'shadow',
            shadowStyle: {
              color: 'rgb(77, 161, 255)',
              opacity: 0.1,
            },
          },
          formatter: function (params, ticket, callback) {
            let label = ''
            if (params.length > 0 && params[0].name) {
              if (params[0].name.indexOf('__::__') >= 0) {
                label = `${params[0].name.split('__::__')[1]}<br />`
              } else {
                label = `${params[0].name}<br />`
              }
            }
            for (const i in params) {
              if (!params[i].data) {
                continue
              }

              if (params[i].seriesName) {
                label += `${params[i].seriesName}:`
              }
              if (unit) {
                const val = transformUnit(params[i].data, unit, 1000, numberFormat)
                label += `${val.text}<br />`
              } else {
                label += `${params[i].data}<br />`
              }
            }
            return label
          },
        },
        grid: {},
      }
      if (this.isHistogram) {
        config.toolbox = {
          show: true,
          feature: {
            magicType: { type: ['line', 'bar'] },
          },
          right: 20,
        }
        config.grid.top = '30px'
        // config.grid.height = '50%'
        // config.grid.containLabel = true
      }
      return config
    },
  },
}
</script>

<style lang="less" scoped>
::v-deep {
  .ve-bar {
    max-height: 1400px; // 30条数据的高度，超过30条数据滚动
    overflow: auto;
  }
}
</style>
