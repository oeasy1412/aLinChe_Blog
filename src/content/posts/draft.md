---
title: Draft Example
published: 2024-08-25
tags: [Demo]
category: Examples
draft: true
---

# This Article is a Draft

This article is currently in a draft state and is not published. Therefore, it will not be visible to the general audience. The content is still a work in progress and may require further editing and review.

When the article is ready for publication, you can update the "draft" field to "false" in the Frontmatter:

```markdown
---
title: Draft Example
published: 2024-01-11T04:40:26.381Z
tags: [Markdown, Blogging, Demo]
category: Examples
draft: false
---
```

#### 样例代码


```dataviewjs

// --- 最终版代码 ---

const { DateTime } = dv.luxon;

const now = DateTime.now();

  

const startOfDay = now.startOf('day');

const passedMinutesInDay = now.diff(startOfDay, 'minutes').minutes;

const dayPercent = (passedMinutesInDay / (24 * 60)) * 100;

  

const startOfWeek = now.startOf('week');

const passedHoursInWeek = now.diff(startOfWeek, 'hours').hours;

const weekPercent = (passedHoursInWeek / (7 * 24)) * 100;

  

const monthPercent = (now.day / now.daysInMonth) * 100;

  

const passedDaysInYear = now.ordinal;

const totalDaysInYear = now.isInLeapYear ? 366 : 365;

const yearPercent = (passedDaysInYear / totalDaysInYear) * 100;

  

const progressBars = [

    { label: "今日进度", value: passedMinutesInDay, max: 24 * 60, percent: dayPercent },

    { label: "本周进度", value: passedHoursInWeek, max: 7 * 24, percent: weekPercent },

    { label: "本月进度", value: now.day, max: now.daysInMonth, percent: monthPercent },

    { label: "本年进度", value: passedDaysInYear, max: totalDaysInYear, percent: yearPercent }

];

  

function createProgressBar(data) {

    const container = dv.el("div", "");

    container.style.display = "flex";

    container.style.alignItems = "center";

    container.style.marginBottom = "8px";

  

    const label = dv.el("span", `${data.label}：`);

    label.style.minWidth = "75px";

    label.style.flexShrink = "0";

  

    const progress = dv.el("progress", "");

    progress.setAttribute("value", data.value);

    progress.setAttribute("max", data.max);

    progress.style.flexGrow = "1";

    progress.style.width = "100%";

    progress.style.height = "14px";

    const percentage = dv.el("span", ` ${data.percent.toFixed(2)}%`);

    percentage.style.marginLeft = "10px";

    container.append(label, progress, percentage);

    dv.paragraph(container);

}

  

progressBars.forEach(bar => createProgressBar(bar));

```


```dataviewjs
// --- 1. 定义时钟的 HTML 结构 ---
const clockHTML = `
<div id="clock" class="progress-clock">
    <button class="progress-clock__time-date" data-group="d" type="button">
        <small data-unit="w">星期四</small><br>
        <span data-unit="mo">六月</span>
        <span data-unit="d">20</span>
    </button>
    <button class="progress-clock__time-digit" data-unit="h" data-group="h" type="button">12</button>
    <span class="progress-clock__time-colon">:</span>
    <button class="progress-clock__time-digit" data-unit="m" data-group="m" type="button">30</button>
    <span class="progress-clock__time-colon">:</span>
    <button class="progress-clock__time-digit" data-unit="s" data-group="s" type="button">55</button>
    <span class="progress-clock__time-ampm" data-unit="ap">pm</span>
    <svg class="progress-clock__rings" width="256" height="256" viewBox="0 0 256 256">
        <g data-units="d">
            <circle class="progress-clock__ring" cx="128" cy="128" r="74" fill="none" opacity="0.1" stroke="#e13e78" stroke-width="12"></circle>
            <circle class="progress-clock__ring-fill" data-ring="mo" cx="128" cy="128" r="74" fill="none" stroke="#e13e78" stroke-width="12" stroke-dasharray="465 465" stroke-dashoffset="0" stroke-linecap="round" transform="rotate(-90,128,128)"></circle>
        </g>
        <g data-units="h">
            <circle class="progress-clock__ring" cx="128" cy="128" r="90" fill="none" opacity="0.1" stroke="#e79742" stroke-width="12"></circle>
            <circle class="progress-clock__ring-fill" data-ring="d" cx="128" cy="128" r="90" fill="none" stroke="#e79742" stroke-width="12" stroke-dasharray="565.5 565.5" stroke-dashoffset="0" stroke-linecap="round" transform="rotate(-90,128,128)"></circle>
        </g>
        <g data-units="m">
            <circle class="progress-clock__ring" cx="128" cy="128" r="106" fill="none" opacity="0.1" stroke="#4483ec" stroke-width="12"></circle>
            <circle class="progress-clock__ring-fill" data-ring="h" cx="128" cy="128" r="106" fill="none" stroke="#4483ec" stroke-width="12" stroke-dasharray="666 666" stroke-dashoffset="0" stroke-linecap="round" transform="rotate(-90,128,128)"></circle>
        </g>
        <g data-units="s">
            <circle class="progress-clock__ring" cx="128" cy="128" r="122" fill="none" opacity="0.1" stroke="#8f30eb" stroke-width="12"></circle>
            <circle class="progress-clock__ring-fill" data-ring="m" cx="128" cy="128" r="122" fill="none" stroke="#8f30eb" stroke-width="12" stroke-dasharray="766.5 766.5" stroke-dashoffset="0" stroke-linecap="round" transform="rotate(-90,128,128)"></circle>
        </g>
    </svg>
</div>
`;

// 将 HTML 渲染到 Dataview 容器中
dv.container.innerHTML = clockHTML;

// --- 2. JavaScript 逻辑 ---
// DataviewJS 内置了 moment.js
const dateElements = {
    week: dv.container.querySelector('[data-unit="w"]'),
    month: dv.container.querySelector('[data-unit="mo"]'),
    day: dv.container.querySelector('[data-unit="d"]'),
};
const timeElements = {
    hour: dv.container.querySelector('[data-unit="h"]'),
    minute: dv.container.querySelector('[data-unit="m"]'),
    second: dv.container.querySelector('[data-unit="s"]'),
    ampm: dv.container.querySelector('[data-unit="ap"]'),
};
const ringFills = {
    day: dv.container.querySelector('[data-ring="mo"]'),
    hour: dv.container.querySelector('[data-ring="d"]'),
    minute: dv.container.querySelector('[data-ring="h"]'),
    second: dv.container.querySelector('[data-ring="m"]'),
};

// 主更新函数
function updateClock() {
    moment.locale('zh-cn');
    const now = moment();
    const formatDate = now.format("dddd-MMMM-D-H-mm-ss-a").split("-");
    const [week, month, day, hour, minute, second, ampm] = formatDate;

    if (dateElements.week) dateElements.week.textContent = week;
    if (dateElements.month) dateElements.month.textContent = month;
    if (dateElements.day) dateElements.day.textContent = day;
    
    if (timeElements.hour) timeElements.hour.textContent = hour;
    if (timeElements.minute) timeElements.minute.textContent = minute;
    if (timeElements.second) timeElements.second.textContent = second;
    if (timeElements.ampm) timeElements.ampm.textContent = ampm;

    const daysInMonth = now.daysInMonth(); 
    const secProgress = second / 60;
    const minProgress = (parseInt(minute) + secProgress) / 60;
    const hourProgress = (parseInt(hour) + minProgress) / 24;
    const dayProgress = (parseInt(day) - 1 + hourProgress) / daysInMonth;

    const circumferences = { day: 465, hour: 565.5, minute: 666, second: 766.5 };

    if (ringFills.second) ringFills.second.setAttribute('stroke-dashoffset', (1 - secProgress) * circumferences.second);
    if (ringFills.minute) ringFills.minute.setAttribute('stroke-dashoffset', (1 - minProgress) * circumferences.minute);
    if (ringFills.hour) ringFills.hour.setAttribute('stroke-dashoffset', (1 - hourProgress) * circumferences.hour);
    if (ringFills.day) ringFills.day.setAttribute('stroke-dashoffset', (1 - dayProgress) * circumferences.day);
}

// 首次加载时立即执行一次
updateClock();

// **--- 以下是修正部分 ---**

// 设置定时器，每秒更新一次，并保存其 ID
const intervalId = window.setInterval(updateClock, 1000);

// 使用 dv.container.onunload 注册一个清理函数。
// 当这个 Dataview 块从 DOM 中卸载时 (例如切换笔记或关闭此文件)，
// 这个函数会被自动调用，从而清除我们的定时器。
dv.container.onunload = () => {
    window.clearInterval(intervalId);
}
```