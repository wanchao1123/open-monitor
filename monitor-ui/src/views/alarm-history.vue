<template>
  <div>
    <Title :title="$t('alarmHistory')"></Title>
    <div>
      <ul>
        <li class="filter-li">
          <DatePicker 
            type="date" 
            :value="startDate" 
            @on-change="changeStartDate"
            format="yyyy-MM-dd HH:mm:ss" 
            placement="bottom-start" 
            :placeholder="$t('startDatePlaceholder')" 
            style="width: 220px">
          </DatePicker>
        </li>
        <li class="filter-li">
          <DatePicker 
            type="date" 
            :value="endDate" 
            @on-change="changeEndDate"
            format="yyyy-MM-dd HH:mm:ss" 
            placement="bottom-start" 
            :placeholder="$t('endDatePlaceholder')" 
            style="width: 220px">
          </DatePicker>
        </li>
        <li class="filter-li">
          <Select v-model="filter" style="width:80px">
            <Option v-for="item in filterList" :value="item.value" :key="item.value">{{ item.label }}</Option>
          </Select>
        </li>
        <li class="filter-li">
          <button class="btn btn-sm btn-confirm-f" @click="getAlarm">
            <i class="fa fa-search"></i>
            {{$t('button.search')}}
          </button>
        </li>
        <li class="filter-li">
          <button class="btn btn-sm btn-cancel-f" @click="realTimeAlarm">
            {{$t('realTimeAlarm')}}
          </button>
        </li>
      </ul>
    </div>
    <div class="flex-container">
      <template v-if="!resultData.length">
        <Tag color="primary">{{$t('table.noDataTip')}}！</Tag>
      </template>
      <transition name="slide-fade">
        <div class="flex-item" v-show="showGraph && resultData.length">
          <div>
            <Tag color="success"><span style="font-size:14px">Low:{{this.low}}</span></Tag>
            <Tag color="warning"><span style="font-size:14px">Medium:{{this.mid}}</span></Tag>
            <Tag color="error"><span style="font-size:14px">High:{{this.high}}</span></Tag>
            <div v-show="alramEmpty" style="display:none" id="elId" class="echart"></div>
            <div v-if="!alramEmpty"  class="alarm-empty">
              <span style="font-size:14px"></span>
            </div>
          </div>
        </div>
      </transition>
      <div class="flex-item" style="width: 100%">
        <section style="margin-left:8px" class="c-dark-exclude-color">
          <template v-for="(filterItem, filterIndex) in filtersForShow">
            <Tag color="success" type="border" closable @on-close="exclude(filterItem.key)" :key="filterIndex">{{filterItem.key}}：{{filterItem.value}}</Tag>
          </template>
          <button v-if="filtersForShow.length" @click="clearAll" class="btn btn-small btn-cancel-f">{{$t('clearAll')}}</button>
        </section>
        <div class="alarm-list">
          <template v-for="(alarmItem, alarmIndex) in resultData">
            <section :key="alarmIndex" class="alarm-item c-dark-exclude-color" :class="'alarm-item-border-'+ alarmItem.s_priority">
              <ul>
                <li>
                  <label class="col-md-2">{{$t('field.endpoint')}}:</label>
                  <Tag type="border" closable @on-close="addParams('endpoint',alarmItem.endpoint)" color="primary">{{alarmItem.endpoint}}</Tag>
                </li>
                <li v-if="!alarmItem.is_custom">
                  <label class="col-md-2">{{$t('field.metric')}}:</label>
                  <Tag type="border" closable @on-close="addParams('metric',alarmItem.s_metric)" color="primary">{{alarmItem.s_metric}}</Tag>
                </li>
                <li>
                  <label class="col-md-2">{{$t('tableKey.s_priority')}}:</label>
                  <Tag type="border" closable @on-close="addParams('priority',alarmItem.s_priority)" color="primary">{{alarmItem.s_priority}}</Tag>
                </li>
                <li v-if="!alarmItem.is_custom && alarmItem.tags">
                  <label class="col-md-2">Tags:</label>
                  <Tag type="border" v-for="(t,tIndex) in alarmItem.tags.split('^')" :key="tIndex" color="cyan">{{t}}</Tag>
                </li>
                <li>
                  <label class="col-md-2">{{$t('tableKey.start')}}:</label><span>{{alarmItem.start_string}}</span>
                </li>
                <li>
                  <label class="col-md-2">{{$t('details')}}:</label>
                  <span>
                    <Tag color="default">{{$t('tableKey.start_value')}}:{{alarmItem.start_value}}</Tag>
                    <Tag color="default" v-if="alarmItem.s_cond">{{$t('tableKey.threshold')}}:{{alarmItem.s_cond}}</Tag>
                    <Tag color="default" v-if="alarmItem.s_last">{{$t('tableKey.s_last')}}:{{alarmItem.s_last}}</Tag>
                    <Tag color="default" v-if="alarmItem.path">{{$t('tableKey.path')}}:{{alarmItem.path}}</Tag>
                    <Tag color="default" v-if="alarmItem.keyword">{{$t('tableKey.keyword')}}:{{alarmItem.keyword}}</Tag>
                  </span>
                </li>
                <li>
                  <label class="col-md-2" style="vertical-align: top;">{{$t('alarmContent')}}:</label>
                  <div class="col-md-9" style="display: inline-block;padding:0" v-html="alarmItem.content"></div>
                </li>
              </ul>
            </section>
          </template>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import echarts from 'echarts'
export default {
  name: '',
  data() {
    return {
      startDate: '',
      endDate: '',
      filter:'start',
      filterList: [
        {label: 'all', value: 'all'},
        {label: 'start', value: 'start'}
      ],

      showGraph: true,
      alramEmpty: true,
      myChart: null,
      interval: null,
      timeForDataAchieve: null,
      filters: {},
      filtersForShow: [],
      actveAlarmIndex: null,
      resultData: [],
      selectedData: '', // 存放选中数据

      low: 0,
      mid: 0,
      high: 0
    }
  },
  mounted(){
    this.myChart = echarts.init(document.getElementById('elId'))
  },
  methods: {
    changeStartDate (data) {
      this.startDate = data
    },
    changeEndDate (data) {
      this.endDate = data
    },
    getAlarm() {
      if (!this.startDate || !this.endDate || Date.parse(new Date(this.startDate)) > Date.parse(new Date(this.endDate))) {
        this.$Message.error(this.$t('timeIntervalWarn'))
        return
      }
      if (this.startDate === this.endDate) {
        this.endDate = this.endDate.replace('00:00:00', '23:59:59')
      }
      const start = Date.parse(this.startDate)/1000
      const end = Date.parse(this.endDate)/1000
      let params = {
        start,
        end,
        filter: this.filter
      }
      let keys = Object.keys(this.filters)
      this.filtersForShow = []
      for (let i = 0; i< keys.length ;i++) {
        params[keys[i]] = this.filters[keys[i]]
        this.filtersForShow.push({key:keys[i], value:this.filters[keys[i]]})
      }
      this.$root.$httpRequestEntrance.httpRequestEntrance('POST', '/monitor/api/v1/alarm/problem/history', params, (responseData) => {
        this.resultData = responseData.data
        this.low = responseData.low
        this.mid = responseData.mid
        this.high = responseData.high
        this.alramEmpty = !!this.low || !!this.mid ||!!this.high
        this.showSunburst(responseData)
      })
    },
    compare (prop) {
      return function (obj1, obj2) {
        var val1 = obj1[prop];
        var val2 = obj2[prop];
        if (!isNaN(Number(val1)) && !isNaN(Number(val2))) {
          val1 = Number(val1);
          val2 = Number(val2);
        }
        if (val1 < val2) {
          return -1;
        } else if (val1 > val2) {
          return 1;
        } else {
          return 0;
        }  
      } 
    },
    showSunburst (originData) {
      this.myChart.off()
      let alramData = originData.data
      let legendData = []
      let pieInner = []
      if (originData.high) {
        let high = {
          name: 'high',
          value: originData.high,
          filterType: 'priority',
          itemStyle: {
            color: '#ed4014'
          }
        }
        legendData.push('high')
        pieInner.push(high)
      }
      if (originData.low) {
        let low = {
          name: 'low',
          value: originData.low,
          filterType: 'priority',
          itemStyle: {
            color: '#19be6b'
          }
        }
        legendData.push('low')
        pieInner.push(low)
      }
      if (originData.mid) {
        let mid = {
          name: 'medium',
          value: originData.mid,
          filterType: 'priority',
          itemStyle: {
            color: '#2d8cf0'
          }
        }
        legendData.push('medium')
        pieInner.push(mid)
      }

      const colorX = ['#33CCCC','#666699','#66CC66','#996633','#9999CC','#339933','#339966','#663333','#6666CC','#336699','#3399CC','#33CC66','#CC3333','#CC6666','#996699','#CC9933']
      let index = 0
      let pieOuter = []
      alramData.forEach(alarm => {
        const has = pieOuter.find(item => item.name === alarm.s_metric && item.type === alarm.s_priority)
        if (has) {
          has.value++
        } else {
          const color = colorX[index]
          index++
          legendData.push(alarm.s_metric)
          pieOuter.push({
            name: alarm.s_metric,
            value: 1,
            type: alarm.s_priority,
            filterType: 'metric',
            itemStyle: {
              color: color
            }
          })
        }
      })
      pieOuter = pieOuter.sort(this.compare('type'))
      let option = {
        backgroundColor: '#ffffff',
          tooltip: {
              trigger: 'item',
              formatter: '{b}: {c}'
          },
          legend: {
            bottom: '15%',
            selectedMode: false,
            data: legendData
          },
          series: [
              {
                  type: 'pie',
                  selectedMode: 'single',
                  radius: [0, '30%'],
                  center: ['50%', '40%'],
                  label: {
                    formatter: '{b}:{c}',
                    position: 'inner',
                    rich: {
                      b: {
                        fontSize: 16,
                        lineHeight: 33
                      }
                    }
                  },
                  labelLine: {
                      show: false
                  },
                  data: pieInner
              },
              {
                  type: 'pie',
                  radius: ['40%', '55%'],
                  center: ['50%', '40%'],
                  label: {
                      formatter: ' {b|{b}:}{c} ',
                      backgroundColor: '#ffffff',
                      borderColor: '#2d8cf0',
                      borderWidth: 1,
                      borderRadius: 4,
                      position: 'outer',
                      alignTo: 'edge',
                      margin: 8,
                      rich: {
                        b: {
                          fontSize: 12,
                          lineHeight: 28
                        }
                      }
                  },
                  data: pieOuter
              }
          ]
      }
      this.myChart.setOption(option)
      this.myChart.on('click', params => {
        this.addParams(params.data.filterType, params.data.name)
      })
    },
    realTimeAlarm () {
      this.$router.push('/alarmManagement')
    },
    addParams (key, value) {
      this.filters[key] = value
      this.getAlarm()
    },
    clearAll () {
      this.filters = []
      this.getAlarm()
    },
    exclude (key) {
      delete this.filters[key]
      this.getAlarm()
    },
  },
  components: {},
}
</script>

<style scoped lang="less">
 .filter-li {
   display: inline-block;
   margin-left: 8px;
 }
.echart {
  height: ~"calc(100vh - 180px)";
  width: ~"calc(100vw * 0.4)";
  background:#ffffff;
}
.alarm-empty {
  height: ~"calc(100vh - 180px)";
  width: ~"calc(100vw * 0.4)";
  text-align: center;
  padding:50px;
  color: #2d8cf0;
}
.flex-container {
  margin: 8px;
  display: flex;
}
li {
  list-style: none;
} 

label {
  margin-bottom: 0;
  text-align: right;
}
.alarm-total {
  float: right;
  font-size: 18px;
}
.alarm-list {
  height: ~"calc(100vh - 180px)";
  width: 100%;
  overflow-y: auto;
}
.alarm-item {
  border: 1px solid @gray-d;
  margin: 8px;
  padding: 4px;
  border-radius: 4px;
  li {
    padding: 2px;
  }
}
.alarm-item-border-high {
  // border: 1px solid @color-orange-F;
  color: @color-orange-F;
}
.alarm-item-border-medium {
  // border: 1px solid @blue-2;
  color: @blue-2;
}
// .alarm-item-border-low {
//   // border: 1px solid @gray-d;
// }

.alarm-item:hover {
  box-shadow: 0 0 12px @gray-d;
}

.alarm-item /deep/.ivu-icon-ios-close:before {
  content: "\F102";
}

.fa-operate {
  margin: 8px;
  float: right;
  font-size: 16px;
  cursor: pointer;
}

/* 可以设置不同的进入和离开动画 */
/* 设置持续时间和动画函数 */
.slide-fade-enter-active {
  transition: all .3s ease;
}
.slide-fade-leave-active {
  transition: all .8s cubic-bezier(1.0, 0.5, 0.8, 1.0);
}
.slide-fade-enter, .slide-fade-leave-to
/* .slide-fade-leave-active for below version 2.1.8 */ {
  transform: translateX(10px);
  opacity: 0;
}
</style>
