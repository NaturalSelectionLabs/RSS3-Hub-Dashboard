<template>
  <h1>RSS3 Dashboard</h1>
  <v-chart class="line-chart" :option="itemChartOptions" />
  <v-chart class="line-chart" :option="lineChartOptions" />
   <v-chart class="line-chart" :option="reqChartOptions" />
  <el-row  justify="center">
    <el-col :xs="24" :sm="12" >
      <v-chart class="pie-chart" :option="evmPieChartOptions" />
    </el-col>
    <el-col :xs="24" :sm="12">
      <v-chart class="pie-chart" :option="linksChartOptions" />
      <!-- <div class="data-card">
          <h2>
            Links
          </h2>
          <span>Current Total: {{overall.links.following}}</span>
        </div> -->
    </el-col>
  </el-row>
  <el-row justify="center">
    <el-button @click="handleClick">View Raw Statistics</el-button>
  </el-row>
  <!-- <el-row>
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
  </el-row> -->
  <!-- <el-row :gutter="20" v-for="field in ['links', 'assets']" :key="field">
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
    <el-col
      v-for="type in keys(overall[field])"
      :key="type"
      :span="24 / (keys(overall[field]).length + 1)"
    >
      <el-card class="box-card">
        <template #header>
          <div class="card-header">
            <span>{{ field }}: {{ type }}</span>
          </div>
        </template>
        <div class="text">
          {{
            type.split(".").length === 1
              ? overall[field][type]
              : overall[field][type.split(".")[0]][type.split(".")[1]]
          }}
        </div>
      </el-card>
    </el-col>
  </el-row> -->

  <!-- <div class="load-details">
    <el-button type="primary" @click="loadDetails">Load Details ↓</el-button>
  </div> -->
</template>

<script>
// import { nextTick } from 'vue'
import { ElRow, ElCol, ElLoading, ElButton } from "element-plus";
import axios from "axios";
import { use } from "echarts/core";
import { CanvasRenderer } from "echarts/renderers";
import { LineChart, PieChart,BarChart } from "echarts/charts";
import {
  TooltipComponent,
  GridComponent,
  PolarComponent,
  TitleComponent,
  LegendComponent,
} from "echarts/components";
import VChart from "vue-echarts";

use([
  CanvasRenderer,
  TooltipComponent,
  LineChart,
  PieChart,
  BarChart,
  GridComponent,
  PolarComponent,
  TitleComponent,
  LegendComponent,
]);

export default {
  name: "App",
  components: {
    ElRow,
    ElCol,
    VChart,
    ElButton
  },
  data() {
    return {
      overall: {
        count: 0,
        links: {},
        assets: {},
        items: {}
      },
      details: [],
      loading: null,
      lineChartOptions: null,
      evmPieChartOptions: null,
      linksChartOptions: null,
      itemChartOptions: null,
      reqChartOptions: null
    };
  },
  async mounted() {
    this.openLoading();
    this.overall = (
      await axios.get(
        "https://raw.githubusercontent.com/NaturalSelectionLabs/RSS3-PreNode-Data/main/statics/overall.json"
      )
    ).data;

    const history = (
      await axios.get(
        "https://raw.githubusercontent.com/NaturalSelectionLabs/RSS3-PreNode-Data/main/statics/history.json"
      )
    ).data;

    const reqData = (
      await axios.get(
        "https://dashboard-analytics.rss3.workers.dev/",     
      )
    ).data;

    const tspList = Object.keys(history);
    var dailyMax = {};

    tspList.forEach((tsp) => {
      const currDate = tsp.split("T")[0];
      if (Object.keys(dailyMax).includes(currDate)) {
        if (+new Date(tsp) > +new Date(dailyMax[currDate])) {
          dailyMax[currDate] = tsp;
        }
      } else {
        dailyMax[currDate] = tsp;
      }
    });

    var itemData = [];
    for (const [key, value] of Object.entries(this.overall.items)) {
      if (key!=="totalCount") itemData.push({ value: value, name: key });
    }


    const historyData = Object.values(dailyMax)
      .map((key) => [+new Date(key), history[key].count])
      .sort((a, b) => a[0] - b[0]);

    var evmData = [];
    for (const [key, value] of Object.entries(this.overall.assets["EVM+"])) {
      evmData.push({ value: value, name: key });
    }

    const linksData = Object.values(dailyMax)
      .map((key) => [+new Date(key), history[key].links.following])
      .sort((a, b) => a[0] - b[0]);

    this.itemChartOptions = {
      textStyle: {
        fontFamily: "RockwellStd",
      },
      title: {
        text: "RSS3 Items",
        subtext: "Current Total: " + this.overall.items.totalCount,
        left: "center",
      },
      tooltip: {
        trigger: "item",
      },
      series: [
        {
          name: "EVM+ Assets",
          type: "pie",
          // radius: [100, 250],
          radius: ["30%", "70%"],
          avoidLabelOverlap: true,
          itemStyle: {
            borderRadius: 5,
            borderColor: "#fff",
            borderWidth: 1,
          },
          label: {
            show: true,
            color: "rgba(0, 0, 0, 0.2)",
          },
          labelLine: {
            lineStyle: {
              color: "rgba(0, 0, 0, 0.1)",
            },
            smooth: 0.1,
            length: 10,
          },
          data: itemData,
        },
      ],
    };


    this.lineChartOptions = {
      textStyle: {
        fontFamily: "RockwellStd",
      },
      title: {
        text: "RSS3 Users",
        subtext: "Current Total: " + this.overall.count,
        left: "center",
      },
      tooltip: {
        trigger: "axis",
      },
      xAxis: {
        type: "time",
        min: historyData[0][0],
      },
      yAxis: {
        axisLabel:{
          inside:true,
          margin: 0,
          verticalAlign: "bottom"
        }
      },
      series: [
        {
          name: "Users",
          data: historyData,
          type: "line",
          symbol: "none",
          smooth: true,
          lineStyle: {
            color: "#0072ff",
          },
          showSymbol: false,
        },
      ],
    };

    this.evmPieChartOptions = {
      textStyle: {
        fontFamily: "RockwellStd",
      },
      title: {
        text: "Assets",
        subtext: "Current Total: " + this.overall.assets.totalCount,
        left: "center",
      },
      tooltip: {
        trigger: "item",
      },
      series: [
        {
          name: "EVM+ Assets",
          type: "pie",
          // radius: [100, 250],
          radius: ["30%", "70%"],
          avoidLabelOverlap: true,
          itemStyle: {
            borderRadius: 5,
            borderColor: "#fff",
            borderWidth: 1,
          },
          label: {
            show: true,
            color: "rgba(0, 0, 0, 0.2)",
          },
          labelLine: {
            lineStyle: {
              color: "rgba(0, 0, 0, 0.1)",
            },
            smooth: 0.1,
            length: 10,
          },
          data: evmData,
        },
      ],
    };

    this.linksChartOptions = {
      textStyle: {
        fontFamily: "RockwellStd",
      },
      title: {
        text: "Links",
        subtext: "Current Total: " + this.overall.links.following,
        left: "center",
      },
      tooltip: {
        trigger: "axis",
      },
      xAxis: {
        type: "time",
        min: historyData[0][0],
      },
      yAxis: {
        axisLabel:{
          inside:true,
          margin: 0,
          verticalAlign: "bottom",
        }
      },
      series: [
        {
          name: "Links",
          data: linksData,
          type: "line",
          symbol: "none",
          smooth: true,
          lineStyle: {
            color: "#0072ff",
          },
          showSymbol: false,
        },
      ],
    };

    this.reqChartOptions = {
            textStyle: {
        fontFamily: "RockwellStd",
      },
            title: {
        text: "RSS3 Network Requests",
        subtext: "Over Past 24 Hours",
        left: "center",
      },
  tooltip: {
    trigger: 'axis',
    axisPointer: {
      type: 'none'
    }
  },
    grid: [
    { left: '10%', top: '12%', width: '35%', height: '75%' },
    { right: '10%', top: '12%', width: '35%', height: '75%' },

  ],
  xAxis: [
    {
      gridIndex:0,
      type: 'category',
      data: ["Total Requests"],
    axisTick: {
      show: false
    },
    axisLine: {
      show: false
    }
    },
        {
          gridIndex:1,
      type: 'category',
      data: ["Unique Users"],
    axisTick: {
      show: false
    },
    axisLine: {
      show: false
    }
    },
  ],
  yAxis: [
    { 
      gridIndex:0,
      type: 'value',
              axisLabel:{
          inside:true,
          margin: 0,
          verticalAlign: "bottom",
          color: "rgba(0, 0, 0, 0.2)",
          showMaxLabel: false
        },
            splitLine: {
      show: false
    }
    },
        {
          gridIndex:1,
      type: 'value',
      position: 'right',
              axisLabel:{
          inside:true,
          margin: 0,
          verticalAlign: "bottom",
          color: "rgba(0, 0, 0, 0.2)",
          showMaxLabel: false
        },
            splitLine: {
      show: false
    }
    }
  ],
  series: [
    {
            xAxisIndex: 0,
      yAxisIndex: 0,
      type: 'bar',
      barWidth: '30%',
      data: [reqData.requests],
            label: {
        show: true,
        position: 'top'
      },
    },
    {
            xAxisIndex: 1,
      yAxisIndex: 1,
      type: 'bar',
      barWidth: '30%',
      data: [reqData.uniques],
            label: {
        show: true,
        position: 'top'
      },
    }
  ]
    };

    this.closeLoading();
  },
  methods: {
    sum(obj) {
      if (typeof obj !== "object") {
        return obj;
      }
      return Object.keys(obj).reduce(
        (p, c) => p + (typeof obj[c] === "object" ? this.sum(obj[c]) : obj[c]),
        0
      );
    },
    dateFormatter(row, column, cellValue) {
      return new Date(cellValue).toLocaleString();
    },
    keys(obj) {
      let result = [];
      for (const key in obj) {
        if (typeof obj[key] === "object") {
          result = result.concat(this.keys(obj[key]).map((k) => key + "." + k));
        } else {
          result.push(key);
        }
      }
      return result;
    },
    openLoading() {
      if (!this.loading) {
        this.loading = ElLoading.service();
      }
    },
    closeLoading() {
      this.loading.close();
      this.loading = null;
    },
    // async loadDetails() {
    //   this.openLoading()
    //   await nextTick()
    //   const detailsData = (await axios.get('https://raw.githubusercontent.com/NaturalSelectionLabs/RSS3-Hub-Next-Data/main/statistics/details.json')).data
    //   this.details = Object.keys(detailsData).map((key) => ({
    //     address: key,
    //     ...detailsData[key]
    //   }))
    //   .splice(0, 100)
    //   await nextTick()
    //   this.closeLoading()
    // }
    handleClick(e) {
      window.open('https://github.com/NaturalSelectionLabs/RSS3-Network-Data/tree/main/statics');
      this.$emit("onClick", e);
    },
  },
};
</script>

<style>
@font-face {
  font-family: "RockwellStd";
  src: local("RockwellStd"),
    url("./fonts/RockwellStd-Light.otf") format("OpenType");
}
* {
  font-family: "RockwellStd";
}
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

.line-chart {
  width: 100%;
  height: 600px;
}
.pie-chart {
  width: 100%;
  height: 500px;
}
.data-card {
  height: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}
</style>
