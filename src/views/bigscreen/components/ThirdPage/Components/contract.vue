<template>
  <div class="third-content">
    <div class="content-header">
      <div class="title-bg-container">
        <img class="title-bg" src="../../../img/tit.png" alt="" />
        <div class="header-left">{{ moduleName }}统计</div>
        <div class="header-right">
          <el-input
            v-model="inputValue"
            size="large"
            style="width: 250px"
            class="input-field"
            placeholder="输入名称搜索"
            clearable
          />
          <el-select
            v-model="status"
            size="large"
            style="width: 250px"
            class="input-field"
            placeholder="请选择合同状态"
            clearable
          >
            <el-option label="已完成" value="completed" />
            <el-option label="履行中" value="ongoing" />
          </el-select>
          <el-button type="primary" size="large" class="search-button">
            搜索
          </el-button>
          <el-button plain size="large" class="reset-button">重置</el-button>
        </div>
      </div>
    </div>
    <div class="content-middle">
      <div
        id="chart-contract-1"
        style="flex: 0.8; height: 300px; margin: auto"
      />
      <div
        id="chart-contract-2"
        style="flex: 1.2; height: 300px; margin: auto"
      />
    </div>
    <div class="content-form">
      <el-table
        class="g-table-2"
        stripe
        :data="tableDataDisplay"
        style="width: 100%"
        height="480"
        @current-change="handleClickRow"
      >
        <!-- <el-table-column label="签订时间" prop="签署日期" /> -->
        <el-table-column label="审批通过日期" prop="审批通过时间" />
        <el-table-column label="合同编号" prop="合同编号" />
        <el-table-column label="合同名称" prop="合同名称" />
        <el-table-column label="采购/销售类型" prop="购销类型" />
        <el-table-column label="合同类型" prop="合同类型" />
        <el-table-column label="业务类型" prop="业务类型" />
        <el-table-column label="合同金额(含税)(万元)" prop="含税金额">
          <template #default="{ row }">
            {{ row.内容?.合同总额 || "-" }}
          </template>
        </el-table-column>
        <el-table-column label="相对方" prop="相对方名称" />
        <el-table-column label="我方" prop="我方名称" />
      </el-table>
      <el-pagination
        v-model:current-page="pagination.currentPage"
        v-model:page-size="pagination.pageSize"
        :page-sizes="pagination.pageSizes"
        :size="size"
        background
        layout="total, sizes, prev, pager, next"
        style="margin-top: 10px; display: flex; justify-content: center"
        :total="pagination.total"
        @size-change="handleSizeChange"
        @current-change="handleCurrentChange"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import * as echarts from "echarts";
import { ref, onMounted, shallowRef } from "vue";
import { useRouter } from "vue-router";
import { ArrowRight } from "@element-plus/icons-vue";
import { ComponentSize } from "element-plus";
import BusinessStandbookAPI from "@/api/businessStandBook";
import { DingTalkFormApi } from "@/api/businessStandBook/dingding";

const router = useRouter();

// 页面动态显示内容，由参数决定
const title: Ref<string> = ref("广投石化驾驶舱");
const moduleName: Ref<string> = ref("合同台账");
const filters: Ref<any> = ref({
  /** 日期起始值 */
  日期晚于: "",
  /** 日期结束值 */
  日期早于: "",
});

const inputValue = ref("");
const status = ref();

const tableData: Ref<[]> = ref([]);
const tableDataDisplay = computed(() => {
  return tableData.value.slice(
    (pagination.value.currentPage - 1) * pagination.value.pageSize,
    pagination.value.currentPage * pagination.value.pageSize
  );
});
// 初始化Echarts图表
const chart = shallowRef<echarts.ECharts | null>(null);
const chart2 = shallowRef<echarts.ECharts | null>(null);

const initChart1 = () => {
  if (!chart.value) {
    chart.value = echarts.init(
      document.getElementById("chart-contract-1") as HTMLDivElement
    );
  }
  chart.value.clear();

  const { chartData, totalAmount, totalCount } = calculateContractTypeData(); // 获取合同类型数据

  // 饼状图
  const option = {
    title: {
      text: "合同类型分布",
      left: "center",
      top: 20,
      textStyle: {
        color: "#fff",
      },
    },
    tooltip: {
      trigger: "item",
      formatter: "{a} <br/>{b} : {c} ({d}%)",
    },
    legend: {
      orient: "vertical",
      left: "left",
      data: chartData.map((item) => item.name),
      textStyle: {
        color: "#fff",
      },
    },
    series: [
      {
        name: "合同类型",
        type: "pie",
        radius: "55%",
        center: ["50%", "60%"],
        data: chartData.map((item) => ({
          value: item.value,
          name: item.name,
        })),
        label: {
          show: true,
          // 颜色暗些
          color: "#eee",
          fontSize: 16,
          formatter: "{b} : {c}份 ({d}%)",
        },
        emphasis: {
          itemStyle: {
            shadowBlur: 10,
            shadowOffsetX: 0,
            shadowColor: "rgba(0, 0, 0, 0.5)",
          },
        },
      },
    ],
  };
  chart.value.setOption(option);
};

// 右侧图
const initChart2 = () => {
  if (!chart2.value) {
    chart2.value = echarts.init(
      document.getElementById("chart-contract-2") as HTMLDivElement
    );
  }
  chart2.value.clear();

  const profitDataArray = calculateProfitData()
    .filter((item) => item["企业名称"])
    .sort((a, b) => b.合同数量 - a.合同数量)
    .slice(0, 10); // 获取我方合同金额数据
  // 筛选前10
  // const top10ProfitDataArray = profitDataArray

  // 柱状图
  const option = {
    tooltip: {
      trigger: "axis",
      axisPointer: {
        type: "shadow",
      },
    },
    // 橙色与蓝色
    color: ["#FFA500", "#00CFFF"],
    legend: {
      data: ["合同金额", "合同数量"],
      textStyle: {
        color: "#fff",
      },
    },
    grid: {
      left: "3%",
      right: "4%",
      bottom: "3%",
      containLabel: true,
    },
    xAxis: {
      type: "category",
      data: profitDataArray.map((item) => item.企业名称), // X轴数据为我方名称
      axisLabel: {
        fontSize: 14,
        color: "#fff",
      },
    },
    yAxis: [
      {
        type: "value",
        name: "单位：万元",
        nameTextStyle: {
          color: "#fff",
          fontSize: 15,
        },
        axisLine: {
          show: true, // 显示坐标轴线
          lineStyle: {
            color: "#fff",
          },
        },
        splitLine: {
          show: true, // 显示分割线
          lineStyle: {
            type: "dashed", // 虚线
            color: "#fff",
          },
        },
        axisLabel: {
          fontSize: 14,
          color: "#fff",
        },
      },
      {
        type: "value",
        name: "单位：份",
        nameTextStyle: {
          color: "#fff",
          fontSize: 15,
        },
        axisLine: {
          show: true,
          lineStyle: {
            color: "#fff",
          },
        },
        splitLine: {
          show: true,
          lineStyle: {
            type: "dashed",
            color: "#fff",
          },
        },
        axisLabel: {
          fontSize: 14,
          color: "#fff",
        },
      },
    ],
    series: [
      {
        type: "bar",
        barWidth: "40%",
        barGap: "10%", // 柱体间距
        data: profitDataArray.map((item) => item.合同金额), // Y轴数据为合同金额
        name: "合同金额",
        // 显示标签
        label: {
          show: true,
          position: "top",
          color: "#fff",
          textStyle: {
            fontSize: "1rem",
          },
        },
      },
      {
        type: "bar",
        barWidth: "40%",
        barGap: "10%",
        data: profitDataArray.map((item) => item.合同数量), // Y轴数据为合同数量
        name: "合同数量",
        yAxisIndex: 1,
        label: {
          show: true,
          position: "top",
          color: "#fff",
          textStyle: {
            fontSize: "1rem",
          },
          formatter: (param: any) => {
            return `${param.value}份`;
          },
        },
      },
    ],
  };
  chart2.value.setOption(option);
};

const goBack = () => {
  router.go(-1);
};

const queryForm: Ref<any> = ref({
  状态集合: undefined,
  数据源集合: undefined,
  日期早于: undefined,
  日期晚于: undefined,
  页码: 1,
  页容量: 10,
});

const pagination: Ref<any> = ref({
  total: 0,
  pageSizes: [10, 20, 50, 100],
  pageSize: 20, // //每页大小
  currentPage: 1, // 当前页
});

async function initTableData() {
  DingTalkFormApi.queryDingTalkContractLedger({
    ...queryForm.value,
    // 页码: pagination.value.currentPage,
    // 页容量: pagination.value.pageSize,
    页码: 1,
    页容量: 999,
  })
    .then((res: any) => {
      tableData.value = res["当前记录"].map((item: any) => {
        return {
          ...item,
          审批通过时间: (item.审批通过时间 || "-").substring(0, 10),
          相对方名称: item["相对方名称"] || item["对方公司名称"],
          我方名称: item["我方名称"] || item["我方公司名称"],
          合同类型: item["合同类型"] || "未分类",
        };
      });
      pagination.value.total = +res["记录总数"];
      initChart1(); // 在数据加载后更新图表
      initChart2();
    })
    .catch((err: any) => {
      console.error(err);
    });
}

const calculateContractTypeData = () => {
  const typeMap: { [key: string]: number } = {};
  let totalAmount = 0;
  let totalCount = 0;

  // 计算每种合同类型的总金额
  tableData.value.forEach((item) => {
    const type = item["合同类型"] || "未归类";
    const amount = parseFloat(item["含税金额"] || "0");
    const count = 1;

    if (!typeMap[type]) {
      typeMap[type] = 0;
    }
    // typeMap[type] += amount;
    // totalAmount += amount;
    typeMap[type] += count;
    totalAmount += amount;
    totalCount += count;
  });

  // 转换为ECharts需要的数据格式
  // console.log("tableData.value", tableData.value);
  const chartData = Object.keys(typeMap).map((type) => {
    return {
      name: type,
      value: typeMap[type],
      percentage: ((typeMap[type] / totalCount) * 100).toFixed(2) + "%",
    };
  });
  // console.log("chartData", chartData, totalAmount);
  return { chartData, totalAmount, totalCount };
};

const calculateProfitData = () => {
  const profitDataMap: {
    [key: string]: { totalAmount: number; count: number };
  } = {};

  // 计算每个我方的合同金额总和和合同数量
  tableData.value.forEach((item) => {
    // const partyName = item["我方名称"] || item["我方公司名称"];
    const partyName = item["相对方名称"] || item["对方公司名称"];
    const amount = parseFloat(item["含税金额"]);
    const contractType = item["合同类型"];

    if (!profitDataMap[partyName]) {
      profitDataMap[partyName] = { totalAmount: 0, count: 0 };
    }

    profitDataMap[partyName].totalAmount += amount;
    profitDataMap[partyName].count += 1;
  });

  // 转换为ECharts需要的数据格式
  const profitDataArray = Object.keys(profitDataMap).map((partyName) => ({
    企业名称: partyName,
    合同金额: profitDataMap[partyName].totalAmount,
    合同数量: profitDataMap[partyName].count,
  }));

  return profitDataArray;
};

onMounted(async () => {
  const route = router.currentRoute.value;
  if (route.query.filters) {
    const params = JSON.parse(route.query.filters as string);
    filters.value = {
      ...filters.value,
      ...params,
    };
    if (params.status) {
      status.value = params.status;
    }
  }
  await initTableData();
  // setTimeout(() => {
  //   initChart1();
  //   initChart2();
  // }, 100);
  window.addEventListener("resize", () => {
    chart.value?.resize();
  });
});

const size = ref<ComponentSize>("default");

const handleSizeChange = (pageSize: number) => {
  pagination.value.pageSize = pageSize;
  // initTableData();
};
const handleCurrentChange = (currentPage: number) => {
  pagination.value.currentPage = currentPage;
  // initTableData();
};

const handleClickRow = (row: any) => {
  const route = router.resolve({
    name: "ReportForm",
    query: {
      // type: "contractDetail",
      type: "dingdingContractDetail",
      id: row.id,
    },
  });
  setTimeout(() => {
    window.open(route.href, "_blank");
  }, 100);
};
</script>

<style lang="scss" scoped>
.bg-view-img2 {
  @apply flex flex-col w-full h-full;
  background-color: #030542;
}
.bg-view1__header {
  height: 66px;
  @apply flex items-center relative w-full;
  background-image: url(../../img/tit_bg.png);
  background-repeat: no-repeat;
  background-size: 80% 100%;
  background-position: center;
  .back-btn {
    left: 30px;
    @apply flex justify-center items-center absolute;
    padding: 8px 15px;
    background-image: url(../../img/back.png);
    background-repeat: no-repeat;
    background-size: 100% 100%;
    background-position: center;
    &:active {
      transform: scale(0.95);
    }
    img {
      width: 15px;
    }
    span {
      font-size: 1rem;
      color: #40c7f4;
      margin-left: 6px;
      transition: transform 0.1s;
    }
  }
  .title {
    @apply flex flex-1 justify-center items-center;
    .__title--text {
      font-size: 2rem;
      letter-spacing: 6px;
      background: linear-gradient(
        to bottom,
        rgb(251, 254, 254),
        rgb(188, 247, 255)
      );
      -webkit-background-clip: text;
      -webkit-text-fill-color: transparent;
      @apply text-center font-bold relative;
    }
  }
}
.bg-view1__body {
  flex-grow: 1;
  margin: 20px 50px 50px 50px;
  ::-webkit-scrollbar {
    @apply hidden;
  }
  scrollbar-width: none;
}
.breadcrumb-box {
  @apply flex items-center;
  color: #d5e2fb;
  img {
    width: 20px;
  }
  .breadcrumb-title {
    font-size: 1rem;
    margin-left: 10px;
    transition: transform 0.1s;
    letter-spacing: 2px;
  }
}
.content-header {
  margin-top: 20px;
}
.title-bg-container {
  @apply relative w-full;
}
.title-bg {
  @apply w-full;
  height: 60px;
}
.header-left {
  @apply absolute z-1;
  font-size: 22px;
  top: 20%;
  left: 3%;
}
.header-right {
  @apply absolute z-1 flex flex-gap-2;
  top: -15px;
  right: 5px;
}
:deep(.input-field) {
  background-color: #1c1e57 !important;
  border: 1px solid #21246f !important;
  font-size: 16px;
  .el-input__wrapper {
    padding-top: 2px;
    padding-bottom: 2px;
    background-color: #1c1e57 !important;
  }
  .el-input__inner {
    background-color: #1c1e57 !important;
    color: #fefefe !important;
    .el-input__inner::placeholder {
      color: #7b9de0 !important;
    }
  }
}
:deep(.el-select) {
  .el-select__wrapper {
    background-color: #1c1e57 !important;
    color: #7b9de0 !important;
    font-size: 16px;
    .el-select__placeholder {
      color: #fefefe !important;
    }
  }
  .el-select__input-wrapper {
    background-color: #1c1e57 !important;
    color: #7b9de0 !important;
  }
}
.search-button {
  margin-left: 12px;
  background-color: #3550ab;
  color: #bbd4fe;
  font-size: 16px;
  letter-spacing: 1px;
  &:hover {
    @apply opacity-80;
  }
}
.reset-button {
  @apply bg-transparent;
  margin-left: 7px;
  color: #bbd4fe;
  font-size: 16px;
  border: 2px solid #3550ab;
  &:hover {
    background-color: #3550ab;
    color: #fff;
  }
}
.content-middle {
  margin-top: 20px;
  @apply flex justify-center flex-gap-2;
  canvas {
    @apply w-full;
  }
}
.content-form {
  margin-top: 20px;
}

:deep(.el-breadcrumb) {
  color: #d5e2fb;
  font-size: 16px;
  margin: 0 3px;
  letter-spacing: 1px;
}
:deep(.el-breadcrumb__separator.el-icon) {
  margin: 0 2px;
}
:deep(.el-breadcrumb__separator) {
  color: #d5e2fb;
}
:deep(.el-breadcrumb__inner) {
  color: #d5e2fb;
}
:deep(
    .el-breadcrumb__item:last-child .el-breadcrumb__inner,
    .el-breadcrumb__item:last-child .el-breadcrumb__inner a,
    .el-breadcrumb__item:last-child .el-breadcrumb__inner a:hover,
    .el-breadcrumb__item:last-child .el-breadcrumb__inner:hover
  ) {
  color: #519cec;
}

:deep(.el-table) {
  background-color: #11134e;
  color: #7b9de0;
}

:deep(.el-table th) {
  background-color: #1b1d67;
  color: #5289ee;
}

:deep(.el-table.is-scrolling-none th.el-table-fixed-column--left) {
  background-color: #1b1d67;
}

:deep(.el-table td) {
  color: #7b9de0;
}

:deep(.el-table tr) {
  @apply bg-transparent;
}

:deep(.el-table th.el-table__cell.is-leaf) {
  border-bottom: none;
}

:deep(
    .el-table--striped
      .el-table__body
      tr.el-table__row--striped
      td.el-table__cell
  ) {
  background: #1c1e57;
}

:deep(.el-table td.el-table__cell, .el-table th.el-table__cell.is-leaf) {
  border-bottom: none;
}

:deep(.el-pagination.is-background .el-pagination__sizes) {
  .el-select {
    .el-select__wrapper {
      @apply bg-transparent;
      .el-select__placeholder {
        color: #fefefe;
      }
    }
  }
}

:deep(.el-pagination.is-background .btn-prev, ) {
  background-color: #272a9b !important;
  color: white;
  &[aria-disabled="true"] {
    background-color: #272a9b !important;
  }
}

:deep(.el-pagination.is-background .btn-next) {
  background-color: #272a9b !important;
  color: white;
}

:deep(.el-pagination.is-background .el-pager li.is-active) {
  background-color: #272a9b;
  color: #fefefe;
}

:deep(.el-pagination__total) {
  color: #fefefe;
}

:deep(.el-pagination.is-background .el-pager li) {
  @apply bg-transparent;
  border: 1px solid #4e6ab2;
  color: #fefefe;
}

:deep(.el-pager li.is-active, .el-pager li:hover) {
  color: #272a9b;
}

:deep(.el-table__inner-wrapper:before) {
  @apply hidden;
}

:deep(.el-input__wrapper) {
  box-shadow: 0 0 0 1px #272a9b inset;
}

:deep(.el-select__wrapper) {
  box-shadow: 0 0 0 1px #272a9b inset;
}

:deep(.el-select__wrapper.is-hovering:not(.is-focused)) {
  box-shadow: 0 0 0 2px #272a9b inset;
}

:deep(
    .el-table--enable-row-hover .el-table__body tr:hover > td.el-table__cell
  ) {
  background-color: #272786 !important;
}

// :deep(.el-table) {
//   // 高亮当前行的背景色为#1c1e57
//   .el-table__body tr.current-row {
//     background-color: #1c1e57 !important;
//   }
// }

:deep(.g-table-2.el-table) {
  // 行距拉高
  .cell {
    line-height: 36px !important;
    font-size: 16px;
  }
}
</style>
