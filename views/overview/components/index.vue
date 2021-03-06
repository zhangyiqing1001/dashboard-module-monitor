<template>
  <div class="overview-index">
    <a-row v-if="scope !=='project'">
      <a-breadcrumb style="padding: 5px;">
        <span>{{ $t('cloudenv.text_374') + ": " }}</span>
        <a-breadcrumb-item v-for="(item, index) in navs" :key="item.id">
          <template v-if="index === navs.length - 1">
            <span class="monitor-overview-breadcrumb-span">{{ locationTitle(item.location) }}</span>
          </template>
          <template v-else>
            <a class="monitor-overview-breadcrumb-link" @click="changeNav(index)">{{ item.location }}</a>
          </template>
        </a-breadcrumb-item>
        <a-breadcrumb-item />
        <a-breadcrumb-item />
        <a-breadcrumb-item />
      </a-breadcrumb>
      <div
        v-if="navs.length > 1"
        class="col-3"
        :style="{ fontSize: '20px', padding: '5px'}">
        <a-icon type="arrow-left" class="anticon anticon-arrow-left monitor-overview-breadcrumb-link" :style="{ cursor: 'pointer', }" @click="changeNav(navs.length-2)" />
        <span :style="{ paddingLeft: '5px' }">{{ navs[navs.length-1].title }}</span>
      </div>
    </a-row>
    <a-row>
      <a-col style="padding-left: 6px; padding-right: 6px;" :span="8">
        <overview-pie class="monitor-overview-card mb-2" :header="{ title: $t('monitor.overview_alert_sum') }" :loading="charts.alert_sum.chartLoading" :chartEvents="pieChartEvent()" :chartData="charts.alert_sum.chartData" :showLegend="true" :legendData="charts.alert_sum.legendData" :pieTitle="charts.alert_sum.title" :pieSubtext="charts.alert_sum.subtitle" />
      </a-col>
      <a-col style="padding-left: 6px; padding-right: 6px;" :span="16">
        <overview-line :header="{ title: charts.res_num.title }" class="monitor-overview-card mb-2" height="300px" :loading="charts.res_num.chartLoading" :isHistogram="true" :chartData="charts.res_num.chartData" :numerify-format="charts.res_num.numerifyFormat" splitLineShow />
      </a-col>
    </a-row>
    <a-row>
      <a-col style="padding-left: 6px; padding-right: 6px;" :span="24">
        <div class="monitor-overview-card mb-2">
          <div class="float-left">
            <a-radio-group v-model="activeTab" @change="radioChange">
              <a-radio-button :value="item.key" v-for="item in tabs" :key="item.key">{{ item.label }}</a-radio-button>
            </a-radio-group>
          </div>
          <div :style="{'margin-top': '30px'}" v-if="currentNav.scope !=='project'">
            <overview-line :height="`${Math.max(200, charts[activeTab].chartData.rows.length * 45)}px`" :header="{ title: charts[activeTab].title }" :loading="charts[activeTab].chartLoading"  :isHistogram="false" :chartData="charts[activeTab].chartData" :unit="charts[activeTab].unit" :chartEvents="chartEvents()" :numerify-format="charts[activeTab].numerifyFormat" key="domainView" />
          </div>
          <div :style="{'margin-top': '30px'}" v-else>
            <overview-line :header="{ title: charts[activeTab].title }" :loading="charts[activeTab].chartLoading"  :isHistogram="true" height="320px" :chartData="charts[activeTab].chartData" :chartSettings="{ showLine: charts[activeTab].chartData.columns}" :unit="charts[activeTab].unit" :numerify-format="charts[activeTab].numerifyFormat" key="projectView" splitLineShow />
          </div>
        </div>
      </a-col>
    </a-row>
    <a-row  v-if="tableData.showtable">
      <a-col style="padding-left: 6px; padding-right: 6px;">
        <div class="table">
          <vxe-grid
            class="mt-4"
            size="mini"
            border
            :columns="tableData.columns"
            max-height="400"
            :data="tableData.rows" />
          <div class="vxe-grid--pager-wrapper">
            <div class="vxe-pager size--mini">
              <div class="vxe-pager--wrapper">
                <span class="vxe-pager--total">{{ total }}</span>
              </div>
            </div>
          </div>
        </div>
      </a-col>
    </a-row>
  </div>
</template>

<script>
import OverviewLine from '@Monitor/components/OverviewLine'
import OverviewPie from '@Monitor/components/OverviewPie'
import { getSignature } from '@/utils/crypto'
import { transformUnit } from '@/utils/utils'
import i18n from '@/locales'

const allTabs = {
  usage_active: {
    label: i18n.t('monitor.overview_tab_usage_active'),
    measurement: 'vm_cpu',
    key: 'usage_active',
    index: 1,
  },
  bps_recv: {
    label: i18n.t('monitor.overview_tab_bps_recv'),
    measurement: 'vm_netio',
    key: 'bps_recv',
    index: 2,
  },
  bps_sent: {
    label: i18n.t('monitor.overview_tab_bps_sent'),
    measurement: 'vm_netio',
    key: 'bps_sent',
    index: 3,
  },
  read_bps: {
    label: i18n.t('monitor.overview_tab_read_bps'),
    measurement: 'vm_diskio',
    key: 'read_bps',
    index: 4,
  },
  write_bps: {
    label: i18n.t('monitor.overview_tab_write_bps'),
    measurement: 'vm_diskio',
    key: 'write_bps',
    index: 5,
  },
}

function newChart (title, subtitle, label, numerifyFormat) {
  const chart = {
    label: label,
    title: '',
    subtitle: '',
    rawDatas: {},
    chartLoading: false,
    numerifyFormat: numerifyFormat || '0.00',
    chartUint: {},
    chartData: {
      columns: [],
      rows: [],
    },
  }
  if (title) {
    chart.title = i18n.t(title)
  }
  if (subtitle) {
    chart.subtitle = i18n.t(subtitle)
  }

  return chart
}

export default {
  name: 'OverviewIndex',
  components: {
    OverviewPie,
    OverviewLine,
  },
  data () {
    const tabs = Object.values(allTabs).sort((a, b) => {
      return allTabs[a.key].index - allTabs[b.key].index
    })

    const charts = {}
    for (const k in allTabs) {
      charts[k] = newChart(`monitor.overview_${k}`, '', allTabs[k].label)
    }

    charts.alert_sum = newChart('monitor.overview_alert_sum_pie', '', '', '0')
    charts.res_num = newChart('monitor.overview_alert_trend', '', '', '0')
    const scope = this.$store.getters.scope
    let navs = []
    if (scope === 'system') {
      navs = [{ id: 'system', location: this.$t('cloudenv.text_457'), title: this.$t('cloudenv.text_457'), scope: scope }]
    } else if (scope === 'domain') {
      navs = [{ id: this.$store.getters.userInfo.domain_id, location: this.$t('dictionary.domain'), title: this.$store.getters.userInfo.projectDomain, scope: scope }]
    }
    return {
      scope: scope,
      tabs: tabs,
      activeTab: tabs[0].key,
      charts: charts,
      tableData: {
        showtable: (scope !== 'project'),
        columns: [],
        rows: [],
      },
      units: {}, // 度量单位
      currentNav: { index: 0, scope: scope, status: 'loading' },
      navs: navs,
    }
  },
  computed: {
    total () {
      const total = this.tableData.rows.length || 0
      return this.$t('monitor_metric_78', [total])
    },
  },
  watch: {
    scope () {
      this.fetchAllData()
    },
    currentNav () {
      this.fetchAllCharts()
    },
  },
  created () {
    this.fetchAllData()
  },
  methods: {
    locationTitle: function (base) {
      return this.$t('monitor.overview.location', [base])
    },
    chartEvents: function () {
      const self = this
      return {
        click: function (e) {
          self.nextNav(e)
        },
      }
    },
    nextNav: function (e) {
      const segs = e.name.split('__::__')
      if (this.currentNav.status === 'loaded' && this.currentNav.scope !== 'project') {
        if (this.currentNav.scope === 'system') {
          this.navs.push({ id: segs[0], location: this.$t('dictionary.domain'), title: segs[1], scope: 'domain' })
        } else if (this.currentNav.scope === 'domain') {
          this.navs.push({ id: segs[0], location: this.$t('dictionary.project'), title: segs[1], scope: 'project' })
        }
        this.currentNav = { index: this.navs.length - 1, scope: this.navs[this.navs.length - 1].scope, status: 'loading' }
      }
    },
    changeNav: function (e) {
      this.navs = this.navs.slice(0, e + 1)
      this.navs[e].status = 'loading'
      this.currentNav = this.navs[e]
      this.currentNav.index = e
    },
    pieChartEvent: function () {
      const self = this
      return {
        click: function (e) {
          self.toHistory(e)
        },
      }
    },
    toHistory: function (e) {
      const rn = this.charts.alert_sum.chartData.rows[e.dataIndex].raw_name
      this.$router.push({ path: '/alertrecord', query: { res_type: rn } })
    },
    radioChange: function (e) {
      const tab = allTabs[this.activeTab]
      this.fetchTabChartData(tab.measurement, tab.key)
    },
    fetchAllData () {
      this.fetchMeasurementsData().then((res) => {
        this.fetchAllCharts()
      })
    },
    async fetchAllCharts () {
      await this.fetchPieChartData()
      // 近30日告警趋势图
      await this.fetchTabChartData('alert_record_history', 'res_num')
      // 近7日图
      const tab = allTabs[this.activeTab]
      await this.fetchTabChartData(tab.measurement, tab.key)
      // table
      await this.fetchTableData()
      this.currentNav.status = 'loaded'
    },
    fetchMeasurementsData () {
      try {
        const data = {
          from: '168h',
          scope: this.$store.getters.scope,
        }
        data.signature = getSignature(data)
        this.units = {}
        const self = this
        return new this.$Manager('unifiedmonitors', 'v1').get({ id: 'measurements', params: data }).then((res) => {
          if (!res.data.res_type_measurements) {
            return
          }
          const ms = res.data.res_type_measurements
          for (const k in ms) {
            const m = ms[k]
            for (const mk in m) {
              if (m[mk].field_descriptions) {
                const fds = m[mk].field_descriptions
                for (const fdk in fds) {
                  const v = fds[fdk]
                  self.units[m[mk].measurement + '/' + v.name] = { labal: i18n.t(v.display_name), unit: v.unit }
                }
              }
            }
          }
        })
      } catch (error) {
        throw error
      }
    },
    commonParams () {
      const extendParams = {
        scope: this.currentNav.scope,
      }

      if (this.currentNav.index >= 1) {
        if (this.currentNav.scope === 'domain') {
          extendParams.domain_id = this.navs[this.currentNav.index].id
        }

        if (this.currentNav.scope === 'project') {
          extendParams.domain_id = this.navs[this.currentNav.index - 1].id
          extendParams.project_id = this.navs[this.currentNav.index].id
        }
      }
      return extendParams
    },
    chartQueryData (measurement, field) {
      const extendParams = this.commonParams()
      if (measurement === 'alert_record_history' && field === 'res_num') {
        return {
          from: '720h',
          interval: '24h',
          metric_query: [
            {
              model: {
                database: 'monitor',
                measurement: measurement,
                select: [
                  [{ params: [field], type: 'field' }, { type: 'sum' }],
                ],
              },
            },
          ],
          unit: true,
          ...extendParams,
        }
      }

      let group = 'tenant'
      let groupId = 'tenant_id'
      let interval = '168h'
      if (this.currentNav.scope === 'system') {
        group = 'project_domain'
        groupId = 'domain_id'
      } else if (this.currentNav.scope === 'project') {
        interval = '1h'
      }

      return {
        from: '168h',
        interval: interval,
        metric_query: [
          {
            model: {
              database: 'telegraf',
              measurement: measurement,
              select: [
                [{ params: [field], type: 'field' }, { type: 'mean' }, { type: 'abs' }],
              ],
              tags: [
                {
                  key: group,
                  operator: '!=',
                  value: '',
                }],
              group_by: [{ type: 'field', params: [group] }, { type: 'field', params: [groupId] }],
            },
          },
        ],
        ...extendParams,
      }
    },
    tabChartData (field, rawDatas) {
      const chartData = {
        columns: [],
        rows: [],
      }

      if (rawDatas) {
        if (!allTabs[field]) {
          // 30天
          if (rawDatas[0] && rawDatas[0].points && rawDatas[0].points.length) {
            const points = rawDatas[0].points
            let series = points.map((item) => {
              return { name: item[1], value: item[0] }
            })
            series = series.sort((a, b) => {
              return a.name - b.name
            })
            chartData.columns = ['name', 'alerts']
            const _temp = {}
            for (const i in series) {
              const d = new Date(series[i].name)
              const rn = `${d.getMonth() + 1}/${d.getDate()}`
              _temp[rn] = { name: rn, alerts: series[i].value }
            }

            // fill data
            const rows = []
            if (Object.keys(_temp).length < 30) {
              const now = new Date()
              for (let i = 30; i > 0; i--) {
                const cur = new Date(now - i * 24 * 60 * 60 * 1000)
                const rn = `${cur.getMonth() + 1}/${cur.getDate()}`
                if (_temp.hasOwnProperty(rn)) {
                  rows.push(_temp[rn])
                } else {
                  rows.push({ name: rn, alerts: 0 })
                }
              }
            }
            chartData.rows = rows
          }
        } else if (this.currentNav.scope === 'project') {
          // vm
          if (rawDatas.length) {
            chartData.columns = ['time']
            const datas = {}
            rawDatas.map((item) => {
              // cloumn
              const cloumnName = item.tags.tenant
              chartData.columns.push(cloumnName)
              item.points.map((point) => {
                if (datas[point[1]]) {
                  datas[point[1]][cloumnName] = parseFloat(point[0].toFixed(2))
                } else {
                  datas[point[1]] = {}
                  datas[point[1]].time = point[1]
                  datas[point[1]][cloumnName] = parseFloat(point[0].toFixed(2))
                }
              })
            })

            const dataKeys = Object.keys(datas).sort((a, b) => { return a - b })
            for (const k of dataKeys) {
              const d = new Date(datas[k].time)
              const darr = [String(d.getMonth() + 1), String(d.getDate()), String(d.getHours()), String(d.getMinutes())]
              for (const i in darr) {
                if (darr[i].length < 2) {
                  darr[i] = '0' + darr[i]
                }
              }
              datas[k].time = `${darr[0]}/${darr[1]} ${darr[2]}:${darr[3]}`
              chartData.rows.push(datas[k])
            }
          }
        } else {
          // 域/项目
          const series = rawDatas.map((item) => {
            const lastPoint = item.points ? item.points[item.points.length - 1] : undefined
            if (lastPoint) {
              return { name: item.tags.tenant || item.tags.project_domain, id: item.tags.tenant_id || item.tags.domain_id, value: lastPoint[0], timestamp: lastPoint[1] }
            }
          })
          if (series.length) {
            const v = allTabs[field].label
            chartData.columns = ['name', v]
            chartData.rows = series.map((item) => {
              const _item = { raw_name: item.name, raw_id: item.id, name: item.id + '__::__' + item.name, timestamp: item.timestamp }
              _item[v] = parseFloat(item.value.toFixed(2))
              return _item
            }).sort((a, b) => {
              return a[v] - b[v]
            })
          }
        }
      }
      return chartData
    },
    async fetchTabChartData (measurement, field) {
      try {
        // if (Object.values(this.charts[field].chartData.rows).length) {
        //   return this.charts[field].chartData
        // }
        this.charts[field].chartLoading = true
        this.charts[field].rawDatas = []
        this.charts[field].chartData = {
          columns: [],
          rows: [],
        }
        var data = this.chartQueryData(measurement, field)
        data.signature = getSignature(data)
        const { data: { series = [] } } = await new this.$Manager('unifiedmonitors', 'v1').performAction({ id: 'query', action: '', data })

        const self = this
        this.$nextTick(_ => {
          this.charts[field].rawDatas = series
          this.charts[field].unit = self.units[measurement + '/' + field] || {}
          this.charts[field].chartData = self.tabChartData(field, series)
          this.charts[field].chartLoading = false
        })
      } catch (error) {
        this.charts[field].chartLoading = false
        throw error
      }
    },
    async fetchPieChartData () {
      try {
        this.charts.alert_sum.chartLoading = true
        this.charts.alert_sum.rawDatas = {}
        this.charts.alert_sum.chartData = {
          columns: [],
          rows: [],
        }

        const params = this.commonParams()
        const { data: series = { } } = await new this.$Manager('alertrecords', 'v1').get({ id: 'total-alert', params: params })

        this.$nextTick(_ => {
          const chartData = {
            columns: ['name', 'count', 'raw_name'],
            rows: [],
          }

          let count = 0
          if (Object.keys(series).length > 0) {
            for (const item in series) {
              count += series[item]
              chartData.rows.push({ raw_name: item, name: this.$t(`dictionary.${item}`), count: series[item] })
            }
          } else {
            chartData.rows.push({ raw_name: '', name: '', count: 0 })
          }
          this.charts.alert_sum.subtitle = String(count)
          this.charts.alert_sum.rawDatas = series
          this.charts.alert_sum.chartData = chartData
          this.charts.alert_sum.chartLoading = false
        })
      } catch (error) {
        this.charts.alert_sum.chartLoading = false
        throw error
      }
    },
    async fetchTableData () {
      this.tableData.showtable = false
      const self = this
      let scopeTitle = ''
      if (this.currentNav.scope === 'project') {
        return
      } else if (this.currentNav.scope === 'system') {
        scopeTitle = this.$t('dictionary.domain')
      } else if (this.currentNav.scope === 'domain') {
        scopeTitle = this.$t('dictionary.project')
      }
      for (const t in allTabs) {
        const tab = allTabs[t]
        await this.fetchTabChartData(tab.measurement, tab.key)
      }
      const namecolumn = {
        edit: false,
        field: 'scope',
        title: scopeTitle,
        slots: {
          default: ({ row }, h) => {
            return [h('a', {
              domProps: {
                innerHTML: row.scope,
              },
              style: {
                color: row.deleted ? 'red' : '',
                textDecoration: row.deleted ? 'line-through' : 'auto',
              },
              props: {
                value: row.scope,
              },
              on: {
                click: () => {
                  self.nextNav({ name: row.scope_id + '__::__' + row.scope })
                },
              },
            }), h('span', {
              domProps: {
                innerHTML: row.deleted ? `(${self.$t('monitor.text_121')})` : '',
              },
            })]
          },
        },
      }
      this.tableData.columns = [namecolumn]
      this.tableData.rows = []
      this.$nextTick(_ => {
        this.tableData.showtable = true
        var datas = {}
        for (const k in allTabs) {
          const chart = this.charts[k].chartData
          this.tableData.columns.push({ field: k, title: this.charts[k].label })
          for (let i = 0; i < chart.rows.length; i++) {
            const id = chart.rows[i].raw_id
            const name = chart.rows[i].raw_name
            const vkey = chart.columns[1] || 'value'
            if (datas[id + name]) {
              const val = transformUnit(chart.rows[i][vkey], this.charts[k].unit.unit, 1000, this.charts[k].numerifyFormat)
              datas[id + name][k] = val.text
            } else {
              const item = { scope: name, scope_id: id, timestamp: chart.rows[i].timestamp }
              const val = transformUnit(chart.rows[i][vkey], this.charts[k].unit.unit, 1000, this.charts[k].numerifyFormat)
              item[k] = val.text
              datas[id + name] = item
            }
          }
        }

        const ids = {}
        for (const k in datas) {
          if (!ids[datas[k].scope_id]) {
            ids[datas[k].scope_id] = { timestamp: datas[k].timestamp, current: k }
          } else {
            if (ids[datas[k].scope_id].timestamp >= datas[k].timestamp) {
              datas[k].deleted = true
            } else {
              datas[ids[datas[k].scope_id].current].deleted = true
              ids[datas[k].scope_id].timestamp = datas[k].timestamp
            }
          }
          this.tableData.rows.push(datas[k])
        }
      })
    },
  },
}
</script>

<style lang="less" scoped>
@import '../../../../../src/styles/less/theme';

.monitor-overview-card {
  border: 1px solid #F1F1F1;
  padding: 12px 24px;
  .header {
    font-weight: 500;
    .title-wrapper {
      .subtitle {
        font-size: 12px;
        color: #ccc;
      }
    }
  }
}

.monitor-overview-breadcrumb-span {
  color: @text-color-secondary !important;
}

.monitor-overview-breadcrumb-link {
  color: @primary-color !important;
}

</style>
