<template>
  <h1>RSS3 Dashboard</h1>
  <el-row>
    <el-col :span="24">
      <el-card class="box-card">
        <template #header>
          <div class="card-header">
            <span>file: sum</span>
          </div>
        </template>
        <div class="text">{{ overall.count }}</div>
      </el-card>
    </el-col>
  </el-row>
  <el-row :gutter="20" v-for="field in (['links', 'assets', 'accounts'])" :key="field">
    <el-col :span="24 / (keys(overall[field]).length + 1)">
      <el-card class="box-card">
        <template #header>
          <div class="card-header">
            <span>{{ field }}: sum</span>
          </div>
        </template>
        <div class="text">{{ sum(overall[field]) }}</div>
      </el-card>
    </el-col>
    <el-col v-for="type in keys(overall[field])" :key="type" :span="24 / (keys(overall[field]).length + 1)">
      <el-card class="box-card">
        <template #header>
          <div class="card-header">
            <span>{{ field }}: {{ type }}</span>
          </div>
        </template>
        <div class="text">{{ type.split('.').length === 1 ? overall[field][type] : overall[field][type.split('.')[0]][type.split('.')[1]] }}</div>
      </el-card>
    </el-col>
  </el-row>
  <v-chart class="chart" :option="echartsOptions" />
  <div class="load-details">
    <el-button type="primary" @click="loadDetails">Load Details ↓</el-button>
  </div>
  <el-table
    stripe
    show-summary
    :data="details"
    style="width: 100%">
    <el-table-column prop="address" label="Address" sortable>
      <template #default="scope">
        <span><a target="_black" :href="'https://rss3.io/' + scope.row.address">{{ scope.row.address }}</a></span>
      </template>
    </el-table-column>
    <el-table-column prop="name" label="Name" />
    <el-table-column prop="date_created" label="Date Created" sortable :formatter="dateFormatter" />
    <el-table-column prop="date_updated" label="Date Updated" sortable :formatter="dateFormatter" />
    <el-table-column v-for="field in (['links', 'assets', 'accounts'])" :key="field" :label="field">
      <el-table-column
        v-for="type in Object.keys(overall[field])"
        :key="type"
        :label="type"
        sortable
        :sort-method="(a, b) => (sum(a[field][type]) || 0) - (sum(b[field][type]) || 0)"
        :sort-orders="['descending', 'ascending', null]">
        <template
          v-if="field !== 'assets'"
          #default="scope">
          <span>{{ scope.row[field][type] || 0 }}</span>
        </template>
        <!-- eslint-disable -->
        <el-table-column
          v-if="field === 'assets'"
          v-for="assetsType in Object.keys(overall[field][type])"
          :key="assetsType"
          :label="assetsType"
          sortable
          :sort-method="(a, b) => (a[field][type]?.[assetsType] || 0) - (b[field][type]?.[assetsType] || 0)"
          :sort-orders="['descending', 'ascending', null]">
          <template #default="scope">
            <span>{{ scope.row[field][type]?.[assetsType] || 0 }}</span>
          </template>
        </el-table-column>
        <!-- eslint-enable -->
      </el-table-column>
    </el-table-column>
  </el-table>
</template>

<script>
import { nextTick } from 'vue'
import { ElRow, ElCol, ElCard, ElTable, ElTableColumn, ElLoading, ElButton } from 'element-plus'
import axios from 'axios'
import { use } from "echarts/core";
import { CanvasRenderer } from "echarts/renderers";
import { LineChart } from "echarts/charts";
import {
  TooltipComponent,
  GridComponent,
} from "echarts/components";
import VChart from "vue-echarts";

use([
  CanvasRenderer,
  TooltipComponent,
  LineChart,
  GridComponent
]);

export default {
  name: 'App',
  components: {
    ElRow,
    ElCol,
    ElCard,
    ElTable,
    ElTableColumn,
    ElButton,
    VChart,
  },
  data () {
    return {
      overall: {
        count: 0,
        links: {},
        assets: {},
        accounts: {}
      },
      details: [],
      loading: null,
      echartsOptions: null,
    }
  },
  async mounted() {
    this.openLoading()
    this.overall = (await axios.get('https://raw.githubusercontent.com/NaturalSelectionLabs/RSS3-Hub-Next-Data/main/statistics/overall.json')).data

    const history = (await axios.get('https://raw.githubusercontent.com/NaturalSelectionLabs/RSS3-Hub-Next-Data/main/statistics/history.json')).data
    const historyData = Object.keys(history).map((key) => ([+new Date(key), history[key].count])).sort((a, b) => a[0] - b[0]);
    this.echartsOptions = {
      tooltip: {
        trigger: 'axis',
        axisPointer: {
          type: 'line',
        }
      },
      xAxis: {
        name: 'Time',
        type: 'time',
        min: historyData[0][0],
      },
      yAxis: {
        name: 'Count',
        min: historyData[0][1],
      },
      series: [
        {
          data: historyData,
          type: 'line',
          smooth: true
        }
      ]
    };
  
    this.closeLoading()
  },
  methods: {
    sum(obj) {
      if (typeof obj !== 'object') {
        return obj
      }
      return Object.keys(obj).reduce((p, c) => p + (typeof obj[c] === 'object' ? this.sum(obj[c]) : obj[c]), 0)
    },
    dateFormatter(row, column, cellValue) {
      return new Date(cellValue).toLocaleString()
    },
    keys(obj) {
      let result = [];
      for (const key in obj) {
        if (typeof obj[key] === 'object') {
          result = result.concat(this.keys(obj[key]).map(k => key + '.' + k))
        } else {
          result.push(key)
        }
      }
      return result
    },
    openLoading() {
      if (!this.loading) {
        this.loading = ElLoading.service()
      }
    },
    closeLoading() {
      this.loading.close()
      this.loading = null
    },
    async loadDetails() {
      this.openLoading()
      await nextTick()
      const detailsData = (await axios.get('https://raw.githubusercontent.com/NaturalSelectionLabs/RSS3-Hub-Next-Data/main/statistics/details.json')).data
      this.details = Object.keys(detailsData).map((key) => ({
        address: key,
        ...detailsData[key]
      }))
      .splice(0, 100)
      await nextTick()
      this.closeLoading()
    }
  }
}
</script>

<style>
h1 {
  text-align: center;
  color: rgb(0, 114, 255);
}
.el-row {
  margin-bottom: 20px;
}
.el-row:last-child {
  margin-bottom: 0;
}
.el-col {
  border-radius: 4px;
}
.grid-content {
  border-radius: 4px;
  min-height: 36px;
}
.row-bg {
  padding: 10px 0;
}

.el-card {
  text-align: center;
  font-weight: bold;
}
.card-header {
  text-transform: capitalize;
}

.load-details {
  margin-bottom: 20px;
  text-align: center;
}

.chart {
  width: 100%;
  height: 600px;
}
</style>
