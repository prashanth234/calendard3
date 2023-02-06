<template>
  <div>
    <div id="legend" class="legend"></div>
    <div id="calendar-heatmap"></div>
  </div>
</template>

<script>
import * as d3 from "d3";
import opdata from '@/views/data2.json';

export default {
  props: {
    data: {
      type: Object,
      required: true
    },
    options: {
      type: Object,
      default: () => { return {chart: {}, legend: {}} }
    }
  },
  mounted () {
    
    // Need to remove
    var data = this._.toPairs(opdata)
    data.forEach((item) => {
      item[0] = this.$moment(item[0]).toDate()
    })
    // Need to remove

    ///////////////////// Chart /////////////////////

    const cellSize = this.options.cellSize || 19
    var width, height

    if (this.options.chart) {
      width = this.options.chart.width
      height = this.options.chart.height
    } else {
      width = 56 * cellSize
      height = 11 * cellSize
    }

    const displayFormatDate = d3.utcFormat('%B %d, %Y')
    const formatDate = d3.utcFormat('%Y-%m-%d')
    const formatMonth = d3.utcFormat("%b")

    const max = d3.quantile(data, 1, d => d[1]);
    const min = d3.quantile(data, 0, d => d[1]);
    
    const color = d3.scaleSequential()
      .interpolator(d3.interpolateBlues)
      .domain([min, max])

    const years = d3.groups(data, d => d[0].getUTCFullYear()).reverse()
    
    let monthNumber, months, dataMap = {}
    years.forEach((year, index) => {
      months = []
      year[1].forEach(value => {
        dataMap[formatDate(value[0])] = value[1] 
        monthNumber = value[0].getUTCMonth()
        if (!months.includes(monthNumber)) { months.push(monthNumber) }
      })
      years[index].push(months)
    })

    const svg = d3.select("#calendar-heatmap")
        .append("svg")
        .attr("width", width)
        .attr("height", height * years.length)
        // .attr("viewBox", [0, 0, width, height * years.length])
        .attr("font-family", "sans-serif")
        .attr("font-size", 10)
        .style("overflow", "auto");

    const year = svg.selectAll("g")
      .data(years)
      .join("g")
      .attr("transform", (d, i) => `translate(40.5,${height * i + cellSize * 1.5})`);

    year.append("text")
        .attr("x", -5)
        .attr("y", -5)
        .attr("font-weight", "bold")
        .attr("text-anchor", "end")
        .text(([year]) => year)
        .style('font-size', 14)
    
    year.append("g")
      .attr("text-anchor", "end")
      .selectAll("text")
      .data(d3.range(7))
      .join("text")
        .attr("x", -5)
        .attr("y", i => (i + 0.5) * cellSize)
        .attr("dy", "0.31em")
        .text(i => "SMTWTFS"[i])
        .style('font-size', 12)

    year.append("g")
        .selectAll("rect")
        // Gives only single year values at a time
        // Data consists of all the days of that particular year
        .data(([year]) => { return d3.utcDays(new Date(Date.UTC(year, 0)), new Date(Date.UTC(year + 1, 0))) })
        .join("rect")
          .attr("stroke", "#000")
          .attr("stroke-width", "0.03px")
          .attr("width", cellSize)
          .attr("height", cellSize)
          // For x getting which week in year.
          .attr("x", d => { return d3.utcSunday.count(d3.utcYear(d), d) * cellSize + 0.5} )
          // For y getting which day of the week in year it is.
          .attr("y", d =>{  return d.getUTCDay() * cellSize + 0.5} )
          .attr("fill", d => {
            var value = dataMap[formatDate(d)]
            return value || value === 0 ? color(value) : '#EEEEEE'
          })
          .on("mouseover", function () {
            d3.select(this)
              .attr("stroke", "#000")
              .attr('stroke-width', "1.2px");
          })
          .on("mouseout", function () {
            d3.select(this)
            .attr("stroke", "#000")
            .attr('stroke-width', "0.05px");
          })
        .append("title")
          .text(d => `${displayFormatDate(d)}\n${dataMap[formatDate(d)] || ''}`);
    
    
    const month = year.append("g")
      .selectAll("g")
      .data(([year]) => { return getMonths(year) })
      .join("g");

    function getMonths (year) {
      let allMonths = d3.utcMonths(new Date(Date.UTC(year, 0)), new Date(Date.UTC(year + 1, 0)))
      let hasValueMonths
      years.forEach(temp => {
        if (temp[0]=== year) { hasValueMonths = temp[2] }
      })
      return allMonths.map(date => {return {date: date, months: hasValueMonths}})
    }

    month.append("path")
      .attr("fill", "none")
      .attr("stroke", "#000")
      .attr("stroke-width", "0.15px")
      .attr("d", function ({date: d}) {
        const t1 = new Date(Date.UTC(d.getUTCFullYear(), d.getUTCMonth()+1, 0)),
            // which day is the first of the month // which week the day is in the year
            d0 = d.getUTCDay(), w0 = d3.utcWeek.count(d3.utcYear(d), d),
            // which day is the end of the month // which week the day is in the year
            d1 = t1.getUTCDay(), w1 = d3.utcWeek.count(d3.utcYear(t1), t1);
        return "M" + (w0 + 1) * cellSize + "," + d0 * cellSize
            + "H" + w0 * cellSize + "V" + 7 * cellSize
            + "H" + w1 * cellSize + "V" + (d1 + 1) * cellSize
            + "H" + (w1 + 1) * cellSize + "V" + 0
            + "H" + (w0 + 1) * cellSize + "Z";
    });
    
    month.append("path")
      .attr("fill", "none")
      .attr("stroke", '#000')
      .attr("stroke-width", 1)
      .attr("d", function ({date: d, months}) {
        monthNumber = d.getUTCMonth()
        if (months.includes(monthNumber)) {
          const t1 = new Date(Date.UTC(d.getUTCFullYear(), monthNumber+1, 0)),
              d0 = d.getUTCDay(), w0 = d3.utcWeek.count(d3.utcYear(d), d),
              d1 = t1.getUTCDay(), w1 = d3.utcWeek.count(d3.utcYear(t1), t1);
          return "M" + (w0 + 1) * cellSize + "," + d0 * cellSize
              + "H" + w0 * cellSize + "V" + 7 * cellSize
              + "H" + w1 * cellSize + "V" + (d1 + 1) * cellSize
              + "H" + (w1 + 1) * cellSize + "V" + 0
              + "H" + (w0 + 1) * cellSize + "Z";
        }
      })

    month.append("text")
        .attr("x", ({date: d}) => d3.utcSunday.count(d3.utcYear(d), d3.utcSunday.ceil(d)) * cellSize + 2)
        .attr("y", -8)
        .text(d => formatMonth(d.date))
        .style('font-size', 14)

    ///////////////////// Legend /////////////////////

    var legendHeight = 20
    var legendWidth = 320

    if (this.options.legend) {
      legendWidth = this.options.legend.width
      legendHeight = this.options.legend.height
    }

    const legendSvg = d3.select("#legend")
      .append('svg')
      .attr("width", legendWidth)
      .attr("height", legendHeight)
      .style("overflow", "visible")
      .style("display", "block")
      .style("border", '1px solid #BDBDBD')
      .style("margin-left", width - legendWidth - 20)

    legendSvg.append('text')
      .attr('x', -2)
      .attr('y', -10)
      .text(min)
    
    legendSvg.append('text')
      .attr('x', legendWidth)
      .attr('y', -10)
      .attr('text-anchor', 'end')
      .text(max)

    // Sequential
    legendSvg.append("image")
      .attr("x", 0)
      .attr("y", 0)
      .attr("width", legendWidth - 2)
      .attr("height", legendHeight - 2)
      .attr("preserveAspectRatio", "none")
      .attr("xlink:href", ramp(color.interpolator()).toDataURL());


    function ramp(color, n = 256) {
      var canvas = document.createElement('canvas');
      canvas.width = n;
      canvas.height = 1;
      const context = canvas.getContext("2d");
      for (let i = 0; i < n; ++i) {
        context.fillStyle = color(i / (n - 1));
        context.fillRect(i, 0, 1, 1);
      }
      return canvas;
    }

  }
}
</script>

<style scoped>
  .legend {
    /* float: right; */
    margin-bottom: 10px;
    margin-right: 20px;
    margin-top: 40px;
  }
</style>