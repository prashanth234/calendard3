<template>
  <div id="my_dataviz"></div>  
</template>

<script>
import * as d3 from "d3";
import odata from '@/views/data.json'

export default {
  /* eslint no-use-before-define: 0 */
  mounted () {
    const data = odata.map((item) => {
      item.date = new Date(item.date)
      return item
    })
    const cellSize = 17
    const width = 954
    const weekday = 'sunday'
    const height = cellSize * (weekday === "weekday" ? 7 : 9)
    const timeWeek = weekday === "sunday" ? d3.utcSunday : d3.utcMonday
    const countDay = weekday === "sunday" ? i => i : i => (i + 6) % 7

    function pathMonth(t) {
      const n = weekday === "weekday" ? 5 : 7;
      const d = Math.max(0, Math.min(n, countDay(t.getUTCDay())));
      const w = timeWeek.count(d3.utcYear(t), t);
      return `${d === 0 ? `M${w * cellSize},0`
          : d === n ? `M${(w + 1) * cellSize},0`
          : `M${(w + 1) * cellSize},0V${d * cellSize}H${w * cellSize}`}V${n * cellSize}`;
    }

    const formatValue = d3.format("+.2%")
    const formatClose = d3.format("$,.2f")
    const formatDate = d3.utcFormat("%x")
    const formatDay = i => "SMTWTFS"[i]
    const formatMonth = d3.utcFormat("%b")

    const max = d3.quantile(data, 0.9975, d => Math.abs(d.value));
    const color = d3.scaleSequential(d3.interpolatePiYG).domain([-max, +max]);

    const years = d3.groups(data, d => d.date.getUTCFullYear()).reverse()

    const svg = d3.select('#my_dataviz')
      .append("svg")
        .attr("viewBox", [0, 0, width, height * years.length])
        .attr("font-family", "sans-serif")
        .attr("font-size", 10);

    const year = svg.selectAll("g")
      .data(years)
      .join("g")
        .attr("transform", (d, i) => `translate(40.5,${height * i + cellSize * 1.5})`);

    year.append("text")
        .attr("x", -5)
        .attr("y", -5)
        .attr("font-weight", "bold")
        .attr("text-anchor", "end")
        .text(([key]) => {console.log(key); return key});

    year.append("g")
        .attr("text-anchor", "end")
      .selectAll("text")
      .data(weekday === "weekday" ? d3.range(1, 6) : d3.range(7))
      .join("text")
        .attr("x", -5)
        .attr("y", i => (countDay(i) + 0.5) * cellSize)
        .attr("dy", "0.31em")
        .text(formatDay);

    year.append("g")
      .selectAll("rect")
      .data(weekday === "weekday"
          ? ([, values]) => values.filter(d => ![0, 6].includes(d.date.getUTCDay()))
          : ([, values]) => values)
      .join("rect")
        .attr("width", cellSize - 1)
        .attr("height", cellSize - 1)
        .attr("x", d => timeWeek.count(d3.utcYear(d.date), d.date) * cellSize + 0.5)
        .attr("y", d => countDay(d.date.getUTCDay()) * cellSize + 0.5)
        .attr("fill", d => color(d.value))
      .append("title")
        .text(d => `${formatDate(d.date)}
  ${formatValue(d.value)}${d.close === undefined ? "" : `
  ${formatClose(d.close)}`}`);

    const month = year.append("g")
      .selectAll("g")
      .data(([, values]) => d3.utcMonths(d3.utcMonth(values[0].date), values[values.length - 1].date))
      .join("g");

    month.filter((d, i) => i).append("path")
        .attr("fill", "none")
        .attr("stroke", "#fff")
        .attr("stroke-width", 3)
        .attr("d", pathMonth);

    month.append("text")
        .attr("x", d => timeWeek.count(d3.utcYear(d), timeWeek.ceil(d)) * cellSize + 2)
        .attr("y", -5)
        .text(formatMonth);
  }
}
</script>