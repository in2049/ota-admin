<template>

  <div style="display: flex;flex-direction: column;">

    <el-form :inline="true" size="small" style="padding-left: 10%">
      <el-form-item label="数据采集时间">
        <el-input style="width: 300px"
                  class="filter-item"
                  v-model="dataTime"
                  disabled
        ></el-input>
      </el-form-item>

      <el-button type="primary" @click="handleRefresh" :loading="refreshLoading" icon="el-icon-refresh" size="small">
        刷新
      </el-button>
    </el-form>

    <div class="box-main">

      <div class="box-content">
        <div v-for="(item,index) in cpData" :key="index" class="box-item" :style="`width:${itemWidth}%;`">

          <el-checkbox :label="index+1" :key="index" class="check-list"></el-checkbox>
          <div class="info" v-if="warningIndex.indexOf((index+1)) > -1">
            <span class="info-item" v-for="(item,index) in warningType[(index +1)]">{{ item }}</span>
          </div>

        </div>

      </div>
      <!--      整柜告警-->
      <div class="deviceWarning" >
        <div class="device" v-for="(item,index) in deviceWarning">
          <div class="info-text">{{ item }}</div>
          <div class="red"></div>
        </div>
      </div>


    </div>
  </div>
</template>

<script>


import { commandResult, deviceData, getWarning, refreshDevice } from '@/api/operation'
import { renderTime, warningType, type } from '@/utils'

export default {
  name: 'Warning',
  data() {
    return {
      itemWidth: 100,
      deviceInfo: JSON.parse(sessionStorage.getItem('infoQuery')),
      dataTime: '',
      //格口数据
      cpData: undefined,
      infoSoc: [],
      warningsData: {},
      warningIndex: [],
      warningType: null,

      //柜子
      refreshLoading: false,
      deviceWarning: [],
      hasWarning: false
    }
  },
  created() {
    this.getList()
  },
  methods: {

    async getList() {
      const loading = this.$loading({
        lock: true,
        text: 'Loading',
        spinner: 'el-icon-loading',
        background: 'rgba(0, 0, 0, 0.7)'
      })
      let res = await deviceData(this.deviceInfo.deviceName)
      let warningRes = await getWarning(this.deviceInfo.deviceName)
      if (res.data.success && res.data.data !== null) {
        this.dataTime = renderTime(res.data.data.dataTime)

        //电池
        this.cpData = Object.values(res.data.data.cpAndBatteryData)
        this.itemWidth = 100 / (Math.ceil(this.cpData.length / 4))

        //warning
        if (warningRes.data.success) {
          let subData = warningRes.data.data.subDeviceWarningData
          let deviceData = warningRes.data.data.deviceWarningData
          if (subData !== {}) {
            this.warningIndex = Object.keys(subData).map(Number)
            this.warningType = warningType(subData)
          }
          if (deviceData !== null) {
            this.deviceWarning = []
            deviceData.forEach((item, index) => {
              this.deviceWarning.push(type(item.warningType))
            })
          }
        }
        loading.close()

      } else {
        loading.close()
        this.$message({
          showClose: true,
          message: '获取失败: ' + res.data.message,
          type: 'error'
        })
      }

    },

    async handleRefresh() {
      let that = this
      that.refreshLoading = true
      let messageId = ''
      let i = 0
      let interval = null
      let cmdRes = null
      let idRes = await refreshDevice(this.deviceInfo.deviceName)
      if (idRes.data !== null) {
        if (idRes.data.success) {
          messageId = idRes.data.data.messageId

          interval = setInterval(function() {
            if (i === 4) {
              that.refreshLoading = false
              clearInterval(interval)
              that.$message({
                showClose: true,
                message: '刷新失败，请刷新重试！',
                type: 'error'
              })
            } else {
              commandResult(messageId).then(res => {
                if (res.data.data !== null) {
                  if (res.data.data.cmdStatus === '2') {
                    console.log(res.data.data.cmdStatus)
                    that.refreshLoading = false
                    clearInterval(interval)
                    that.getList()
                    that.$message({
                      showClose: true,
                      message: '刷新成功！',
                      type: 'success'
                    })
                  } else {
                    cmdRes = res.data.message
                    i++
                  }
                } else {
                  i++
                }
              })
            }

          }, 1000)

        } else {
          this.refreshLoading = false
          this.$message({
            showClose: true,
            message: '刷新失败：' + idRes.data.message,
            type: 'error'
          })
        }
      }
    }

  }

}
</script>

<style scoped lang="scss">
.info {
  display: flex;
  flex-direction: column;
  justify-content: start;

  .info-item {
    font-size: 14px;
    padding: 10px;
  }

  .info-text {
    color: #1482f0;
  }
}


.deviceWarning {
  position: absolute;
  right: 10%;
  bottom: 6%;
  margin-right: -20px;
  display: flex;
  flex-direction: column;
  .device {
    display: flex;
  }
  .red {
    width: 16px;
    height: 16px;
    background-color: lightcoral;
  }

  .info-text {
    margin-right: 8px;
  }

}

</style>
