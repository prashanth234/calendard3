<template>
<div>
  <div id="calendar-heatmap"></div>
  <div id="test"></div>
</div>
</template>

<script>
import * as d3 from "d3";
import odata from '@/views/data.json';
// import opdata from '@/views/data2.json';

export default {
  mounted () {
  var data = odata.map((item) => {
    var date = this.$moment(item.date)
    item.date = date.format('YYYY-MM-DD')
    item.year = date.year()
    item.day = date.day()
    return item
  })

  data = data.filter(item => [2000, 2001].includes(item.year))

  const width = 960, height = 150, cellSize = 17;
  const displayFormatDate = d3.utcFormat('%B %d, %Y')
  const formatDate = d3.utcFormat('%Y-%m-%d')
  const formatMonth = d3.utcFormat("%b")
  // function pathMonth(t) {
  //   const n = 7
  //   const d = Math.max(0, Math.min(n, t.getUTCDay()))
  //   const w = d3.utcSunday.count(d3.utcYear(t), t)
  //   return `${d === 0 ? `M${w * cellSize},0`
  //       : d === n ? `M${(w + 1) * cellSize},0`
  //       : `M${(w + 1) * cellSize},0V${d * cellSize}H${w * cellSize}`}V${n * cellSize}`
  // }
  const dataMap = new Map(data.map(value => {return [value.date, value.value]}));

  const max = d3.quantile(data, 1, d => Math.abs(d.value));
  
  const color = d3.scaleSequential()
    .interpolator(d3.interpolateBlues)
    .domain([-max, +max])

  
  const years = d3.groups(data, d => d.year).reverse()
  // [[2002, [{date: "2002-01-02", value: 0.005171851953457331, close: 10073.400391, year: 2002}, ...........]],............]
  
  const onlyYears = years.map(year => year[0])

  const svg1 = d3.select("#test")
      .append("svg")
      .attr("viewBox", [0, 0, width, height * years.length])
      .attr("font-family", "sans-serif")
      .attr("font-size", 10);

  const year = svg1.selectAll("g")
    .data(years)
    .join("g")
    .attr("transform", (d, i) => `translate(40.5,${height * i + cellSize * 1.5})`);

  year.append("text")
      .attr("x", -5)
      .attr("y", -5)
      .attr("font-weight", "bold")
      .attr("text-anchor", "end")
      .text(([year]) => year);
  
  year.append("g")
    .attr("text-anchor", "end")
    .selectAll("text")
    .data(d3.range(7))
    .join("text")
      .attr("x", -5)
      .attr("y", i => (i + 0.5) * cellSize)
      .attr("dy", "0.31em")
      .text(i => "SMTWTFS"[i]);

  year.append("g")
      .selectAll("rect")
      // Gives only single year values at a time
      // Data consists of all the days of that particular year
      .data(([year]) => { return d3.utcDays(new Date(Date.UTC(year, 0)), new Date(Date.UTC(year + 1, 0))) })
      .join("rect")
        .attr("stroke", "#000")
        .attr("stroke-width", "0.05px")
        .attr("width", cellSize - 1)
        .attr("height", cellSize - 1)
        // For x getting which week in year.
        .attr("x", d => { return d3.utcSunday.count(d3.utcYear(d), d) * cellSize + 0.5} )
        // For y getting which day of the week in year it is.
        .attr("y", d =>{  return d.getUTCDay() * cellSize + 0.5} )
        .attr("fill", d => {
          var value = dataMap.get(formatDate(d))
          return value || value === 0 ? color(value) : '#EEEEEE'
        })
        .on("mouseover", function () {
          d3.select(this)
            .attr("stroke", "#000")
            .attr('stroke-width', "1.3px");
        })
        .on("mouseout", function () {
          d3.select(this)
          .attr("stroke", "")
          .attr('stroke-width', "0.1px");
        })
      .append("title")
        .text(d => `${displayFormatDate(d)}\n${dataMap.get(formatDate(d)) || ''}`);
  
  // year.append("g")
  //   .attr("fill", "none")
  //   .attr("stroke", "#000")
  //   .attr("stroke-width", "1px")
  //   .selectAll("path")
  //   .data(([year]) => { return d3.utcMonths(new Date(Date.UTC(year, 0)), new Date(Date.UTC(year + 1, 0))) })
  //   .enter().append("path")
  //   .attr("d", function (d) {
  //       const t1 = new Date(Date.UTC(d.getUTCFullYear(), d.getUTCMonth() + 1, 0)),
  //           d0 = d.getUTCDay(), w0 = d3.utcWeek.count(d3.utcYear(d), d),
  //           d1 = t1.getUTCDay(), w1 = d3.utcWeek.count(d3.utcYear(t1), t1);
  //       return "M" + (w0 + 1) * cellSize + "," + d0 * cellSize
  //           + "H" + w0 * cellSize + "V" + 7 * cellSize
  //           + "H" + w1 * cellSize + "V" + (d1 + 1) * cellSize
  //           + "H" + (w1 + 1) * cellSize + "V" + 0
  //           + "H" + (w0 + 1) * cellSize + "Z";
  //   })
  
  const month = year.append("g")
    .selectAll("g")
    .data(([year]) => { return d3.utcMonths(new Date(Date.UTC(year, 0)), new Date(Date.UTC(year + 1, 0))) })
    .join("g");

  month.append("path")
      .attr("fill", "none")
      .attr("stroke", "#000")
      .attr("stroke-width", 1.1)
      .attr("d", function (d) {
        const t1 = new Date(Date.UTC(d.getUTCFullYear(), d.getUTCMonth()+1, 0)),
            d0 = d.getUTCDay(), w0 = d3.utcWeek.count(d3.utcYear(d), d),
            d1 = t1.getUTCDay(), w1 = d3.utcWeek.count(d3.utcYear(t1), t1);
        return "M" + (w0 + 1) * cellSize + "," + d0 * cellSize
            + "H" + w0 * cellSize + "V" + 7 * cellSize
            + "H" + w1 * cellSize + "V" + (d1 + 1) * cellSize
            + "H" + (w1 + 1) * cellSize + "V" + 0
            + "H" + (w0 + 1) * cellSize + "Z";
    });

  month.append("text")
      .attr("x", d => d3.utcSunday.count(d3.utcYear(d), d3.utcSunday.ceil(d)) * cellSize + 2)
      .attr("y", -5)
      .text(formatMonth);

  var svg = d3.select("#calendar-heatmap")
    .selectAll("svg")
    .data(d3.range(parseInt(Math.min(...onlyYears)), parseInt(Math.max(...onlyYears)) + 1))
    .enter().append("svg")
    .attr("width", width)
    .attr("height", height)
    .append("g")
    .attr("transform", "translate(" + ((width - cellSize * 53) / 2) + "," + (height - cellSize * 7 - 1) + ")");

  svg.append("g")
    .attr("fill", "none")
    .attr("stroke", "#000")
    .attr("stroke-width", "0.1px")
    .selectAll("rect")
    .data(d => d3.timeDays(new Date(d, 0, 1), new Date(d + 1, 0, 1))) // Returns the array of date's between the range of years
    .enter().append("rect")
    .attr("width", cellSize)
    .attr("height", cellSize)
    .attr("x", d => d3.timeWeek.count(d3.timeYear(d), d) * cellSize)
    .attr("y", d => d.getDay() * cellSize)
    .datum(d3.timeFormat("%Y-%m-%d"))
    .attr('fill', d => { return color(dataMap.get(d)) })
    .on("mouseover", function () {
        d3.select(this).attr('stroke-width', "1px");
    })
    .on("mouseout", function () {
        d3.select(this).attr('stroke-width', "0.1px");
    })
    .append("title")
    .text(d => d + "\n" + dataMap.get(d))

  svg.append("text")
    .attr("transform", "translate(-6," + cellSize * 3.5 + ")rotate(-90)")
    .attr("font-family", "sans-serif")
    .attr("font-size", "1em")
    .attr("text-anchor", "middle")
    .text(d => d)

  svg.append("g")
    .attr("fill", "none")
    .attr("stroke", "#000")
    .attr("stroke-width", "1.25px")
    .selectAll("path")
    .data(d => d3.timeMonths(new Date(d, 0, 1), new Date(d + 1, 0, 1)))
    .enter().append("path")
    .attr("d", function (d) {
      // console.log(d)
        const t1 = new Date(d.getFullYear(), d.getMonth() + 1, 0),
            d0 = d.getDay(), w0 = d3.timeWeek.count(d3.timeYear(d), d),
            d1 = t1.getDay(), w1 = d3.timeWeek.count(d3.timeYear(t1), t1);
        return "M" + (w0 + 1) * cellSize + "," + d0 * cellSize
            + "H" + w0 * cellSize + "V" + 7 * cellSize
            + "H" + w1 * cellSize + "V" + (d1 + 1) * cellSize
            + "H" + (w1 + 1) * cellSize + "V" + 0
            + "H" + (w0 + 1) * cellSize + "Z";
    });

  }
}
</script>