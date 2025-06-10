---
tags:
  - Home
---
足迹
```ActivityHistory
/
```


字数热力图
```dataviewjs
// 读取 vault-stats.json
const jsonString = await app.vault.adapter.read(".obsidian/vault-stats.json");

if (!jsonString || jsonString.trim() === "") {
    throw new Error("vault-stats.json is empty!");
}
let jsonObject;
try {
    jsonObject = JSON.parse(jsonString);
} catch (e) {
    throw new Error("vault-stats.json is not valid JSON!");
}
if (!jsonObject.history) {
    throw new Error("vault-stats.json missing 'history' field!");
}

// 获取历史天数
function getJsonLength(jsonData) {
    return Object.keys(jsonData).length;
}

// 获取指定天数前的日期字符串
function getYMD(n) {
    const today = new Date();
    today.setDate(today.getDate() + n);
    const tYear = today.getFullYear();
    const tMonth = ("0" + (today.getMonth() + 1)).slice(-2);
    const tDate = ("0" + today.getDate()).slice(-2);
    return `${tYear}-${tMonth}-${tDate}`;
}

const history = jsonObject.history;
const length = getJsonLength(history);
const YMD = [];
const ycount = [];
const data = [];

for (let i = 0; i < length; i++) {
    const dateStr = getYMD(0 - i);
    YMD.push(dateStr);
    if (history[dateStr]) {
        ycount.push(history[dateStr].words);
    } else {
        ycount.push(0);
    }
    data.push([dateStr, ycount[i]]);
}

const currentYear = new Date().getFullYear();
const option = {
    backgroundColor: "rgba(0, 0, 0, 0)",
    width: 800,
    height: 300,
    tooltip: {
        position: "top",
        trigger: "item",
        formatter: function (params) {
            let dataIndex = params.dataIndex;
            let date = data[dataIndex][0];
            let value = data[dataIndex][1];
            return `日期：${date}<br>字数：${value}`;
        }
    },

    visualMap: {
        type: "piecewise",
        splitNumber: 7,
        orient: "horizontal",
        left: "center",
        top: 0,
        textStyle: { color: "white" },
        pieces: [
            { gte: 0, lte: 500 },
            { gt: 500, lte: 1000 },
            { gt: 1000, lte: 3000 },
            { gt: 3000, lte: 5000 },
            { gt: 5000, lte: 8000 },
            { gt: 8000 }
        ],

        color: [
            "#FF0000",
            "#FF3319",
            "#FF8040",
            "#FF9933",
            "#FFB30F",
            "#c0a75c"
        ],
        calculable: true
    },
    calendar: {
        left: 30,
        right: 10,
        range: currentYear,
        itemStyle: {
            normal: {
                color: "rgba(0, 0, 0, 0)",
                borderWidth: 2
            }
        }
    },
    series: [
        {
            type: "scatter",
            coordinateSystem: "calendar",
            data: data
        }
    ]
};
app.plugins.plugins["obsidian-echarts"].render(option, this.container);
```

```dataview
CALENDAR file.ctime
```


今天是 `=date(today).year`年`=date(today).month`月`=date(today).day`日, 今年已经过去了`=(date(today)-date(date(today).year+"-01-01")).days` 天。


最近10天修改
```dataview
LIST WHERE file.mtime >= date(today) - dur(10 day) sort file.mtime desc limit (20)
```



最近3天修改
```dataview
LIST WHERE file.mtime >= date(today) - dur(3 day) sort file.mtime desc limit (10)
```
