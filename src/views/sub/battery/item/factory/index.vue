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
          <div class="info">
            <div class="info-item"><span class="info-text">厂家：</span>{{ infoFactory[index] || '----' }}</div>
            <div class="info-item"><span class="info-text">厂家编码：</span>{{ infoProductModel[index] || '----' }}</div>
            <div class="info-item"><span class="info-text">硬件版本：</span>{{ infoHard[index] || '----' }}</div>
            <div class="info-item"><span class="info-text">软件版本：</span>{{ infoSoft[index] || '----' }}</div>
          </div>

        </div>

      </div>

    </div>
  </div>
</template>

<script>


import {commandResult, deviceData, refreshDevice} from "@/api/operation";
import {batteryError, iconBattery, isLocked, renderTime} from "@/utils";

export default {
  name: 'FactoryVersion',
  data() {
    return {
      itemWidth: 100,
      deviceInfo: JSON.parse(sessionStorage.getItem('infoQuery')),
      dataTime: '',
      //格口数据
      cpData: undefined,
      infoFactory: [],
      infoProductModel: [],
      infoHard: [],
      infoSoft: [],
      //柜子
      refreshLoading: false,
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
      });
      let res = await deviceData(this.deviceInfo.deviceName)
      if (res.data.success && res.data.data !== null) {
        //电池
        this.cpData = Object.values(res.data.data.cpAndBatteryData)
        this.itemWidth = 100 / (Math.ceil(this.cpData.length / 4))
        this.cpData.forEach((item, index, arr) => {
          if (item.deviceInfoVo !== null) {
            this.infoFactory[index] = item.deviceInfoVo.factoryName
            this.infoProductModel[index] = item.deviceInfoVo.productModel
            this.infoHard[index] = item.deviceInfoVo.hardVersion
            this.infoSoft[index] = item.deviceInfoVo.softVersion
          }
        })
        loading.close();

      } else {
        loading.close();
        this.$message({
          showClose: true,
          message: '获取失败，请刷新重试: ' + res.data.message,
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

        interval = setInterval(function () {
          if (i === 4) {
            that.refreshLoading = false
            clearInterval(interval)
            that.$message({
              showClose: true,
              message: '刷新失败，请重试！' ,
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
  },

}

}
</script>

<style scoped>
.info {
  display: flex;
  flex-direction: column;
  justify-content: start;
  padding: 10px;
}

.info-item {
  font-size: 14px;
  padding: 4px;
}

.info-text {
  color: #1482f0;
}
</style>
