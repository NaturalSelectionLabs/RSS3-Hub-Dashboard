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
import { ElRow, ElCol, ElCard, ElTable, ElTableColumn, ElLoading } from 'element-plus'
import axios from 'axios'

export default {
  name: 'App',
  components: {
    ElRow,
    ElCol,
    ElCard,
    ElTable,
    ElTableColumn,
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
      loading: null
    }
  },
  async mounted() {
    this.openLoading()
    this.overall = (await axios.get('https://raw.githubusercontent.com/NaturalSelectionLabs/RSS3-Hub-Next-Data/main/statistics/overall.json')).data
    const detailsData = (await axios.get('https://raw.githubusercontent.com/NaturalSelectionLabs/RSS3-Hub-Next-Data/main/statistics/details.json')).data
    this.details = Object.keys(detailsData).map((key) => ({
      address: key,
      ...detailsData[key]
    }))
    // .splice(0, 100)
    await nextTick()
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
</style>
