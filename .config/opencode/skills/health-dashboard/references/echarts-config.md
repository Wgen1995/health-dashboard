# ECharts 防截断配置规范

> 本文件定义 health-dashboard 看板中所有 ECharts 图表必须遵循的配置规范。
> AI 生成 HTML 看板时必须 Read 本文件并严格遵循。

## ECharts 引入

```html
<script src="https://cdn.jsdelivr.net/npm/echarts@5/dist/echarts.min.js"></script>
```

如用户要求离线，AI 下载 echarts.min.js 内嵌到 HTML 的 `<script>` 标签中。

## 通用防截断配置（所有图表必须应用）

| 配置项 | 值 | 作用 |
|--------|-----|------|
| `grid.containLabel` | `true` | 网格自动包含轴标签，不被裁剪 |
| `grid.left` | `"3%"` 或 `"8%"` | 左侧留白 |
| `grid.right` | `"4%"` 或 `"18%"`（横向条形图留更多） | 右侧留白 |
| `grid.top` | `"15%"` | 顶部留白给柱顶标签 |
| `grid.bottom` | `"8%"` 或 `"12%"` | 底部留白 |
| `yAxis.boundaryGap` | `["0", "20%"]` | Y轴顶部留20%空间给柱顶标签 |
| `series.label.show` | `true` | 显示数据标签 |
| `xAxis.axisLabel.interval` | `0` | 强制显示所有标签 |
| `xAxis.axisLabel.rotate` | `30`（柱状图）或 `0`（横向条形图） | X轴标签倾斜防重叠 |
| `xAxis.axisLabel.overflow` | `"truncate"` | 超长标签截断 |
| `xAxis.axisLabel.width` | `80` | 限制标签宽度 |
| `tooltip.confine` | `true` | 提示框不溢出容器 |
| 图表容器 CSS | `min-height: 400px` | 每图最小高度，防矮图挤压 |

## 图表类型 → 配置映射

### 柱状图（医生投入排行等对比场景）

```javascript
{
  grid: { containLabel: true, left: '3%', right: '4%', top: '15%', bottom: '8%' },
  xAxis: {
    type: 'category',
    data: [...],
    axisLabel: { interval: 0, rotate: 30, overflow: 'truncate', width: 80 }
  },
  yAxis: { type: 'value', boundaryGap: ['0', '20%'] },
  series: [{
    type: 'bar',
    label: { show: true, position: 'top', fontSize: 11 },
    itemStyle: { color: '#2E86C1' }
  }]
}
```

### 横向条形图（地区分布等长标签场景）

```javascript
{
  grid: { containLabel: true, left: '3%', right: '18%', top: '10%', bottom: '10%' },
  xAxis: { type: 'value' },
  yAxis: {
    type: 'category',
    data: [...],
    axisLabel: { width: 100, overflow: 'truncate' }
  },
  series: [{
    type: 'bar',
    label: { show: true, position: 'right', fontSize: 11 },
    itemStyle: { color: '#2E86C1' }
  }]
}
```

### 饼图（风险等级构成等占比场景）

```javascript
{
  series: [{
    type: 'pie',
    radius: ['35%', '65%'],
    data: [...],
    label: {
      show: true,
      formatter: '{b}: {c} ({d}%)',
      overflow: 'truncate'
    },
    itemStyle: { borderColor: '#fff', borderWidth: 2 }
  }]
}
```

### 折线图（月度趋势等时间序列场景）

```javascript
{
  grid: { containLabel: true, left: '3%', right: '4%', top: '15%', bottom: '12%' },
  xAxis: {
    type: 'category',
    data: [...],
    axisLabel: { interval: 0, rotate: 30 }
  },
  yAxis: { type: 'value', boundaryGap: ['0', '15%'] },
  series: [{
    type: 'line',
    symbolSize: 8,
    label: { show: true, position: 'top' },
    itemStyle: { color: '#2E86C1' },
    markPoint: { data: [{ type: 'max' }, { type: 'min' }] }
  }]
}
```

### 堆叠柱状图（风险等级月度趋势等分层场景）

同柱状图配置，series 中每个系列加 `stack: 'total'`。

### 热力图（医生×地区矩阵等交叉分析场景）

```javascript
{
  grid: { containLabel: true, left: '10%', right: '12%', top: '8%', bottom: '15%' },
  xAxis: { type: 'category', data: [...], axisLabel: { interval: 0, rotate: 30 } },
  yAxis: { type: 'category', data: [...], axisLabel: { width: 60, overflow: 'truncate' } },
  visualMap: { min: 0, max: N, calculable: true, orient: 'horizontal', left: 'center', bottom: '2%' },
  series: [{
    type: 'heatmap',
    data: [...],
    label: { show: true }
  }]
}
```

## 配色规范

| 元素 | 颜色 |
|------|------|
| 主色 | `#2E86C1`（医蓝） |
| 深色 | `#1A5276` |
| 标题栏背景 | `linear-gradient(135deg, #2E86C1, #1A5276)` |
| 卡片背景 | `#fff` |
| 卡片阴影 | `0 2px 12px rgba(0,0,0,0.08)` |
| 文字解读区背景 | `#F4F6F7` |
| 文字解读区左边框 | `4px solid #2E86C1` |
| 正文文字 | `#333` |
| 次要文字 | `#666` |

## 响应式规范

```css
@media (max-width: 768px) {
  .chart-grid { grid-template-columns: 1fr; }
  .kpi-grid { grid-template-columns: 1fr; }
  .chart-container { min-height: 400px; }
}
```

```javascript
window.addEventListener('resize', function() {
  Object.keys(chartInstances).forEach(function(id) {
    if (chartInstances[id]) chartInstances[id].resize();
  });
});
```
