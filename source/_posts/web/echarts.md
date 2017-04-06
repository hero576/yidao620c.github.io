---
title: "JavaScript图表库ECharts使用"
date: 2016-06-22 12:09:08 +0800
comments: true
toc: true
categories: web
tags: [echarts]
---

ECharts是一款由百度前端技术部开发的，基于Javascript的数据可视化图表库，提供直观，生动，
可交互，可个性化定制的数据可视化图表。

提供大量常用的数据可视化图表，底层基于ZRender（一个全新的轻量级canvas类库），创建了坐标系，图例，提示，
工具箱等基础组件，并在此上构建出折线图（区域图）、柱状图（条状图）、散点图（气泡图）、饼图（环形图）、K线图、地图、
力导向布局图以及和弦图，同时支持任意维度的堆积和多图表混合展现。<!--more-->

官方网站：<http://echarts.baidu.com/index.html>

通过自己一个实际项目例子来说明最基本用法，这里我画的是最基础的二维折线图。

首先引入最新js依赖`echarts.min.js`
``` html
<script type="text/javascript" src="${ctx}/assets/js/echarts.min.js"></script>
```

第二步，页面定义图表div，里面放的隐藏input是首页加载时候从后台传入的_隔开数据:
``` html
<!-- 为ECharts准备一个具备大小（宽高）的Dom -->
<div id="main1" style="width: 800px;height:400px;"></div>
<hr/>

<input id="xDataH" type="hidden" value="${dayVO.xaxis!}">
<input id="scanDataH" type="hidden" value="${dayVO.yaxisScan!}">
```

注意看js怎么写:
``` js
function details(channelId) {
    console.log('statistic detail...' + channelId);
    var opts = {
        type: "get",
        url: "${ctx}/statisticDetail.html?channelId=" + channelId,
        dataType: "html",
        success: function (data, strStatus) {
            $('#topDiv').empty().html(data);
            // 开始初始化echart数据
            var xData = $("#xDataH").val().split('_');
            var scanData = $("#scanDataH").val().split('_');
            // 基于准备好的dom，初始化echarts实例
            var myChart1 = echarts.init(document.getElementById('main1'));
            var option1 = {
                title: {
                    text: '扫描二维码次数'
                },
                tooltip: {
                    trigger: 'axis'
                },
                legend: {
                    data:['扫描二维码次数']
                },
                grid: {
                    left: '3%',
                    right: '4%',
                    bottom: '3%',
                    containLabel: true
                },
                toolbox: {
                },
                xAxis: {
                    type: 'category',
                    boundaryGap: false,
                    data: xData
                },
                yAxis: {
                    type: 'value'
                },
                series: [
                    {
                        name:'扫描二维码次数',
                        type:'line',
                        stack: '总量',
                        data: scanData
                    }
                ]
            };
            // 使用刚指定的配置项和数据显示图表。
            myChart1.setOption(option1);
        }
    };
    $.ajax(opts);
}
```

最后来一张效果图:

![](https://xnstatic-1253397658.file.myqcloud.com/echarts01.png)

