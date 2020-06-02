[toc]
## 简介 ##
- 实现上海二手房全量数据动态更新、统计、汇总展示。
- 效果：
  - ![200602.echarts.png](https://img-blog.csdnimg.cn/20200603002234511.png)
  - ![https://img-blog.csdnimg.cn/20200603003733132.png](https://img-blog.csdnimg.cn/20200603003733132.png "200602.form.png")
- [前端项目](https://github.com/hanjg/house-vue)
  - [关键流程](https://blog.csdn.net/qq_40369829/article/details/106089327)
- [后端项目](https://github.com/hanjg/house)

## echarts使用 ##
### 依赖 ###
- 安装
```txt
cnmp install echarts
```

- 依赖。路径：``` views/dashboard/index.vue```
```js
<script>
import { mapGetters } from 'vuex'
import { getDistrictSecondHandHouseSummary } from '@/api/table'
import { assembleOptions } from '@/utils/echartsutils'

var echarts = require('echarts');
</script>

```

### 框架 ###
- template中搭建框架。路径：``` views/dashboard/index.vue```
```html
    <el-row>
      <el-col :span="2"></el-col>
      <el-col :span="21">
        <div id="chart1" style="width: 100%;height:500px;"></div>
      </el-col>
      <el-col :span="1"></el-col>
    </el-row>

```

### 渲染 ###
- 根据返回值,组装option,渲染template中的视图框架。
- 渲染逻辑，路径：``` views/dashboard/index.vue```
```js
<script>
import { mapGetters } from 'vuex'
import { getDistrictSecondHandHouseSummary } from '@/api/table'
import { assembleOptions } from '@/utils/echartsutils'

var echarts = require('echarts');

export default {
  name: 'Dashboard',
  computed: {
    ...mapGetters([
      'name'
    ])
  },
  data() {
    return {
      avgUnitPriceOption: {},
      avgTotalPriceOption: {},
      avgTotalHouseOption: {},
      mychart1:{},
    }
  },
  methods: {
    fetchData() {
      getDistrictSecondHandHouseSummary().then(response => {
        let result = response.result;
        let options = assembleOptions(result);
        this.avgTotalHouseOption = options.avgTotalHouseOption;
        this.avgTotalPriceOption = options.avgTotalPriceOption;
        this.avgUnitPriceOption = options.avgUnitPriceOption;
        this.drawChart();
      });
    },
    drawChart() {
      this.mychart1 = echarts.init(document.getElementById("chart1")).setOption(this.avgUnitPriceOption);
      window.addEventListener("resize", () => {
        setTimeout(() => {
          this.mychart1.resize();
        }, 2000)
      });
    }
  },
  mounted() {
    this.fetchData();
  }
}
</script>
```

- 工具类，路径：``` utils/echartsutils.js```
```js
import { formatDate } from '@/utils/date'

export function assembleOptions(result) {
  let options = {};
  let dataX = [];
  for (let i = 0; i < result.timeList.length; i++) {
    dataX.push(formatDate(result.timeList[i]));
  }
  let legend = result.districts;

  let avgUnitPriceY = {};
  for (let key in result.sumMap) {
    avgUnitPriceY[key] = [];
    for (let i = 0; i < result.sumMap[key].length; i++) {
      let node = result.sumMap[key][i];
      avgUnitPriceY[key].push(node.avgUnitPrice);
    }
  }
  options.avgUnitPriceOption = assembleAvgUnitPriceOption(dataX, legend, avgUnitPriceY);

  let avgTotalPriceY = {};
  for (let key in result.sumMap) {
    avgTotalPriceY[key] = [];
    for (let i = 0; i < result.sumMap[key].length; i++) {
      let node = result.sumMap[key][i];
      avgTotalPriceY[key].push(node.avgTotalPrice/10000);
    }
  }
  options.avgTotalPriceOption = assembleAvgTotalPriceOption(dataX, legend, avgTotalPriceY);

  let totalHouseY = {};
  for (let key in result.sumMap) {
    totalHouseY[key] = [];
    for (let i = 0; i < result.sumMap[key].length; i++) {
      let node = result.sumMap[key][i];
      totalHouseY[key].push(node.totalHouseCount);
    }
  }
  options.avgTotalHouseOption = assembleTotalHouseOption(dataX, legend, totalHouseY);
  return options;
}

/**
 * @param dataX 轴
 * @param legend 图例
 * @param dataY  legend:[]
 * @returns echarts option
 */
export function assembleAvgUnitPriceOption(dataX, legend, dataY) {
  let option = JSON.parse(JSON.stringify(optionTemplate))
  option.title.text = '均价'
  option.yAxis.axisLabel.formatter = '{value}'
  option.xAxis.data = dataX;
  option.legend.data = legend;
  option.series = assembleDataY(dataY);
  return option;
}

export function assembleAvgTotalPriceOption(dataX, legend, dataY) {
  let option = JSON.parse(JSON.stringify(optionTemplate))
  option.title.text = '总价'
  option.yAxis.axisLabel.formatter = '{value}万'
  option.xAxis.data = dataX;
  option.legend.data = legend;
  option.series = assembleDataY(dataY);
  return option;
}

export function assembleTotalHouseOption(dataX, legend, dataY) {
  let option = JSON.parse(JSON.stringify(optionTemplate))
  option.title.text = '总数'
  option.yAxis.axisLabel.formatter = '{value}'
  option.xAxis.data = dataX;
  option.legend.data = legend;
  option.series = assembleDataY(dataY);
  return option;
}

export function assembleDataY(dataY) {
  let lineList = [];
  for (let key in dataY) {
    let line = {};
    line.name = key;
    line.type = 'line';
    line.markLine = {
      data: [
        {type: 'average', name: '平均值'},
      ]
    };
    line.data = [];
    for (let i = 0; i < dataY[key].length; i++) {
      let node = dataY[key][i];
      line.data.push(node)
    }
    lineList.push(line);
  }
  return lineList;
}

var optionTemplate = {
  title: {
    text: '',
  },
  tooltip: {
    trigger: 'axis'
  },
  legend: {
    data: []
  },
  toolbox: {
    show: true,
    feature: {
      dataZoom: {
        yAxisIndex: 'none'
      },
      dataView: {readOnly: false},
      magicType: {type: ['line', 'bar']},
      restore: {},
      saveAsImage: {}
    }
  },
  xAxis: {
    type: 'category',
    boundaryGap: false,
    data: []
  },
  yAxis: {
    type: 'value',
    axisLabel: {
      formatter: '{value}'
    }
  },
  series: [
    {
      name: '行政区1',
      type: 'line',
      data: [11, 44, 12],
      markLine: {
        data: [
          {type: 'average', name: '平均值'}
        ]
      }
    }
  ]
};


```
