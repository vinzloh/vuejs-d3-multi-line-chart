<template>
  <section data-root>
    <div data-line-chart :id="chartId" ref="chart" :style="chartStyle">
      <div v-if="!isChartOptionsValid" class="invalid-options">
        <div>Line Chart</div>
        <div v-if="!hasData">No chart data provided</div>
        <div v-if="!hasXKey">No x-axis key defined for data</div>
        <div v-if="!hasYKey">No y-axis key defined for data</div>
      </div>
    </div>
    <div data-line-chart-legend>
      <div class="legend-key">Key</div>
      <ul>
        <li class="legend-series" v-for="(d,i) in chartData" :key="d.name">
          <div class="legend-icon" :style="`background-color: ${ordinalColors(i)}`"></div>
          <div class="legend-text">{{d.name}}</div>
        </li>
      </ul>
    </div>
    <div class="line-chart-tooltip" :data-tooltip="chartId"></div>
  </section>
</template>

<script>
import _ from "lodash";
import Popper from "popper.js";
import { scaleTime, scaleLinear, scaleOrdinal } from "d3-scale";
import { line } from "d3-shape";
import { select, selectAll, mouse } from "d3-selection";
import { bisector, extent, max } from "d3-array";
import { axisBottom, axisLeft } from "d3-axis";
import { schemeCategory10 } from "d3-scale-chromatic";
import { transition } from "d3-transition";
import { timeParse, timeFormat } from "d3-time-format";
import { format as d3format } from "d3-format";
import {
  timeSecond,
  timeMinute,
  timeHour,
  timeDay,
  timeMonth,
  timeWeek,
  timeYear
} from "d3-time";
const d3 = {
  select,
  selectAll,
  mouse,
  bisector,
  extent,
  max,
  axisBottom,
  axisLeft,
  transition,
  timeParse,
  timeFormat,
  timeSecond,
  timeMinute,
  timeHour,
  timeDay,
  timeMonth,
  timeWeek,
  timeYear,
  line,
  schemeCategory10,
  scaleTime,
  scaleLinear,
  scaleOrdinal,
  format: d3format
};

export default {
  props: {
    chartId: {
      type: String,
      default: () => `linechart-${Date.now()}`
    },
    colors: { type: Array, default: () => d3.schemeCategory10 },
    width: { type: String, default: "100%" },
    height: { type: String, default: "300px" },
    xAxisLabel: { type: String, default: null },
    yAxisLabel: { type: String, default: null },
    xKey: { type: String, required: true /* date */ },
    yKey: { type: String, required: true /* price */ },
    interval: { type: String, default: "year" },
    data: {
      type: Array,
      default: () => []
    }
  },
  computed: {
    ordinalColors() {
      return d3.scaleOrdinal(this.colors);
    },
    chartStyle() {
      return {
        width: this.width,
        height: this.height
      };
    },
    hasData() {
      return this.data.length !== 0;
    },
    hasXKey() {
      return !!this.xKey;
    },
    hasYKey() {
      return !!this.yKey;
    },
    isChartOptionsValid() {
      return this.hasData && this.hasXKey && this.hasYKey;
    },
    chartData() {
      const parseDate = date => {
        const dateFormat = "%d/%m/%Y";
        const r = d3.timeParse(dateFormat)(date);
        if (!r) throw Error(`String date format expected: ${dateFormat}`);
        return r;
      };
      return _.map(this.data, o => ({
        ...o,
        values: _.map(o.values, d => ({
          x: parseDate(d[this.xKey]),
          y: +d[this.yKey]
        }))
      }));
    },
    chartHeight() {
      return this.$refs.chart.clientHeight;
    },
    chartWidth() {
      return this.$refs.chart.clientWidth;
    },
    tooltipEl() {
      return document.querySelector(`[data-tooltip='${this.chartId}']`);
    },
    xAxisMax() {
      return _.max(this.chartData.map(({ values }) => values.length));
    },
    longestSeries() {
      return _.find(this.chartData, d => d.values.length === this.xAxisMax);
    },
    xAxisTicksByInterval() {
      const { interval, xAxisMax } = this;
      const axisBy = {
        year: d3.timeYear,
        // day: d3.timeDay,
        week: d3.timeWeek,
        month: d3.timeMonth
      };
      return axisBy[interval] || xAxisMax;
    }
  },
  watch: {
    data() {
      this.renderChart();
    }
  },
  methods: {
    multiDateFormat(date, i = this.getIndexOfFocusPoint(date)) {
      // const  formatMillisecond = d3.timeFormat('.%L'),
      // formatSecond = d3.timeFormat(':%S'),
      // formatMinute = d3.timeFormat('%I:%M'),
      // formatHour = d3.timeFormat('%I %p'),
      // formatWeek = d3.timeFormat('Week %U'),
      // formatMonth = d3.timeFormat('%B'),

      const formatBy = {
        day: d3.timeFormat("%b %d"),
        week: d3.timeFormat(`Week ${i + 1}`),
        month: d3.timeFormat("%B"),
        quarter: d3.timeFormat(`Quarter ${i + 1}`),
        year: d3.timeFormat("%Y")
      };

      const formatter = formatBy[this.interval];
      // const formatter =
      //   (d3.timeSecond(date) < date && formatMillisecond) ||
      //   (d3.timeMinute(date) < date && formatSecond) ||
      //   (d3.timeHour(date) < date && formatMinute) ||
      //   (d3.timeDay(date) < date && formatHour) ||
      //   (d3.timeMonth(date) < date
      //     ? (d3.timeWeek(date) < date && formatDay) || formatWeek
      //     : (d3.timeYear(date) < date && formatMonth) || formatYear);

      // console.log(`${date} formatter:`, formatter);
      return formatter && _.escape(formatter(date));
    },

    getDatumInSeries({ series: { values }, xPoint, focusPoint }) {
      const bisectDate = d3.bisector(d => d.x).left;
      const i = bisectDate(values, xPoint, 1);
      const d0 = values[i - 1];
      const d1 = values[i] || {};
      const closestD = xPoint - d0.x > d1.x - xPoint ? d1 : d0;
      const xPointMinusClosestD = Math.abs(xPoint - closestD.x);
      const focusPointMinusXPoint = Math.abs(xPoint - focusPoint);
      const r = focusPoint
        ? xPointMinusClosestD > focusPointMinusXPoint
          ? null
          : closestD
        : closestD;
      // console.log(`r:`, r);
      return r;
    },
    getDateSansTime(date) {
      return `${date.getDate()}-${date.getMonth()}-${date.getFullYear()}`;
    },
    getIndexOfFocusPoint(focusPoint) {
      const { getDateSansTime, longestSeries } = this;
      return _.findIndex(
        longestSeries.values,
        d => getDateSansTime(d.x) === getDateSansTime(focusPoint)
      );
    },
    initTooltipPopper(tooltipRefEl, tooltipEl) {
      return new Popper(tooltipRefEl, tooltipEl, {
        placement: "left",
        modifiers: {
          preventOverflow: {
            boundariesElement: document.querySelector(`#${this.chartId}`)
          },
          offset: {
            offset: "0, 8px"
          },
          flip: { behavior: ["left", "right"] }
        }
      });
    },
    composeTooltipHTML({ focusDataPoints, focusPoint }) {
      return `
        <div class="line-chart-tooltip-title">${this.multiDateFormat(
          focusPoint
        )}</div>
        <div class="series-container">
          ${focusDataPoints
            .map(
              d => `
                <div class="series-row">
                  <span class="color-bar" style="background-color:${_.escape(
                    d.color
                  )}">&nbsp;</span>
                  <div class="series-info">
                    <div class="series-name">${_.escape(d.series)}</div>
                    <div class="series-value">${_.escape(d.y)}</div>
                  </div>
                </div>
              `
            )
            .join("")}
        </div>
        <div class="popper-arrow" x-arrow=""></div>
      `;
    },
    renderChart() {
      const c = this;
      const {
        ordinalColors: colors,
        chartData: data,
        isChartOptionsValid,
        chartId,
        chartWidth,
        chartHeight,
        getDatumInSeries,
        tooltipEl,
        multiDateFormat,
        longestSeries,
        xAxisTicksByInterval,
        initTooltipPopper,
        composeTooltipHTML,
        xAxisLabel,
        yAxisLabel
      } = c;
      if (!isChartOptionsValid) return;

      const axisOffset = 16;
      const labelOffset = 21 * 1.75 + axisOffset;

      const margin = {
        top: 30,
        left: 30 + (yAxisLabel ? labelOffset : 0),
        right: 30,
        bottom: 30 + (xAxisLabel ? labelOffset : 0)
      };
      const width = chartWidth - margin.left - margin.right;
      const height = chartHeight - margin.top - margin.bottom;

      /* Scales */
      const xScale = d3
        .scaleTime()
        .domain(d3.extent(longestSeries.values, d => d.x))
        .range([0, width]);

      const yScale = d3
        .scaleLinear()
        .domain([0, d3.max(longestSeries.values, d => d.y)])
        .range([height - 0, 0]);

      /* draw line fn for each series */
      const drawLine = ({ values }) =>
        d3
          .line()
          .x(d => xScale(d.x))
          .y(d => yScale(d.y))(values);

      const tooltip = d3.select(tooltipEl);

      /* remove previously rendered, if any */
      d3.select(`#${chartId} svg`).remove();
      d3.select(`#${chartId} div`).remove();
      tooltip.html("");

      /* SVG */
      const svg = d3
        .select(`#${chartId}`)
        .append("svg")
        .attr("width", width + margin.left + margin.right + "px")
        .attr("height", height + margin.top + margin.bottom + "px");

      /* position main g element */
      const g = svg
        .append("g")
        .attr("transform", `translate(${margin.left}, ${margin.top})`);

      /* add axes */
      const xAxis = d3
        .axisBottom(xScale)
        .tickSize(0)
        .tickFormat(multiDateFormat)
        .ticks(xAxisTicksByInterval);

      g.append("g")
        .attr("class", "x axis")
        .attr("transform", `translate(0, ${height})`)
        .call(xAxis)
        .selectAll("text")
        .attr("y", axisOffset);

      const yAxis = d3
        .axisLeft(yScale)
        .tickSize(-width)
        .ticks(4)
        .tickFormat(d3.format("~s"))
        .scale(yScale.nice());

      g.append("g")
        .attr("class", "y axis")
        .call(yAxis)
        .selectAll("text")
        .attr("x", -axisOffset);

      // Add X axis label
      const xAxisTicksApproxHeight = 30;
      const xLabelOffset = xAxisTicksApproxHeight + 21;
      svg
        .select(".x.axis")
        .append("text")
        .text(xAxisLabel)
        .attr("class", "line-graph-label")
        .attr("transform", `translate(${width / 2}, ${xLabelOffset})`);

      // Add Y axis label
      svg
        .select(".y.axis")
        .append("text")
        .text(yAxisLabel)
        .attr("class", "line-graph-label")
        .attr(
          "transform",
          `translate(${-labelOffset}, ${(height - margin.top) / 2}) rotate(-90)`
        );

      /* add series lines */
      const lines = g.append("g").attr("class", "lines");

      const series = lines
        .selectAll(".line-group")
        .data(data)
        .enter()
        .append("g")
        .attr("class", "line-group");

      series
        .append("path")
        .attr("class", "line")
        .attr("d", d => drawLine(d))
        .style("stroke", (d, i) => colors(i))
        .style("opacity", "0.85");

      /* Tooltip reference point */
      const tooltipRef = d3
        .select(`#${chartId}`)
        .append("div")
        .attr("class", "tooltip-ref");

      let tooltipPopper; /* init once, reuse popper */

      const overlay = g.append("rect");
      overlay
        .attr("class", "overlay")
        .attr("width", width)
        .attr("height", height)
        .on("mousemove", function() {
          const [mouseX] = d3.mouse(this);
          const xPoint = xScale.invert(mouseX);
          const focusDataPoints = []; /* all dots on focused xAxis */

          g.selectAll(".circle").remove(); /* remove rendered dots */

          /* determine focus point from mouse hover X */
          const { x: focusPoint } = getDatumInSeries({
            series: longestSeries,
            xPoint
          });

          /* for each series, render dot, push to array for tooltip render */
          data.forEach((series, i) => {
            const d = getDatumInSeries({ series, xPoint, focusPoint });
            if (!d) return; /* skip if not focus point */
            focusDataPoints.push({
              ...d,
              series: series.name,
              color: colors(i)
            });

            /* dot on focus point */
            const circleRadius = 3;
            g.append("g")
              .attr("class", "circle")
              .append("circle")
              .style("fill", colors(i))
              .attr("cx", xScale(d.x))
              .attr("cy", yScale(d.y))
              .attr("r", circleRadius);
          });

          /* update tooltip reference position */
          tooltipRef
            .style("left", `calc(${xScale(focusPoint)}px + ${margin.left}px)`)
            .style("transform", `translateY(-50%)`)
            .style("top", `calc(50% - ${margin.top}px)`);

          /* compose HTML for tooltip */
          tooltip.style("display", "inherit").html(
            composeTooltipHTML({
              focusDataPoints,
              focusPoint
            })
          );

          /* init popper once */
          if (!tooltipPopper) {
            const [tooltipRefEl] = tooltipRef.nodes();
            tooltipPopper = initTooltipPopper(tooltipRefEl, tooltipEl);
          }
          /* update popper on mousemove */
          if (tooltipPopper) tooltipPopper.update();
        })
        .on("mouseout", function() {
          const [mouseX, mouseY] = d3.mouse(this);
          const maxWidth = overlay.node().getAttribute("width");
          const maxHeight = overlay.node().getAttribute("height");
          /* remove dots and tooltips */
          if (
            mouseX < 0 ||
            mouseX >= maxWidth ||
            mouseY < 0 ||
            mouseY >= maxHeight
          ) {
            g.selectAll(".circle").remove();
            tooltip.html("");
          }
        });
    }
  },
  mounted() {
    this.renderChart();
  }
};
</script>

<style lang="scss">
@import "../assets/styles/base";
@import "../assets/styles/colors";
@import "../assets/styles/fonts";

[data-root] {
  display: flex;
  > [data-line-chart-legend],
  > [data-line-chart] {
    flex-flow: row wrap;
  }
  .line-chart-tooltip {
    flex-direction: column;
  }
  [data-line-chart-legend] {
    text-align: left;
    width: 30%;
    padding: 1.5rem;
    color: $chart-axis-color;
    border: 1px solid transparent;
    border-left-color: $chart-border-color;
    .legend-key {
      @include titillium-web-semibold;
      font-size: 1rem;
      margin-bottom: 0.875rem;
    }
    ul {
      padding: 0;
      list-style-type: none;
      li.legend-series {
        @include titillium-web-regular;
        font-size: 0.875rem;
        margin-bottom: 0.875rem;
        line-height: 1.3125rem;
        height: 1.3125rem;
        &:last-child {
          margin-bottom: 0;
        }
        > div {
          display: inline-block;
        }
        .legend-icon {
          margin-right: 0.625rem;
          width: 0.625rem;
          height: 0.625rem;
        }
      }
    }
  }
}
[data-line-chart] {
  font-size: 0.875rem;
  position: relative;
  .invalid-options {
    border: 1px solid red;
    padding: 1rem;
  }
  .overlay {
    fill: none;
    pointer-events: all;
  }
  .line {
    stroke-width: 2;
    fill: none;
  }
  .tick {
    line {
      stroke: $chart-border-color;
    }
  }
  .axis {
    .tick {
      text {
        @include titillium-web-regular;
        font-size: 0.75rem;
        fill: $chart-axis-color;
      }
    }
    &.y .domain {
      display: none;
    }
  }
  .tooltip-ref {
    position: absolute;
  }
  .line-graph-label {
    @include titillium-web-semibold;
    font-size: 0.625rem;
    text-transform: uppercase;
    fill: $chart-axis-color;
  }
}

$arrow-width: 0.5rem;
.line-chart-tooltip {
  @extend %transition-all-ease;
  filter: drop-shadow(0 0px 2px $shadow-color);
  display: none;
  border: 1px solid $chart-border-color;
  padding: 0;
  background: $white;
  // width: auto;
  max-width: 15rem;

  .popper-arrow {
    width: 0;
    height: 0;
    margin: $arrow-width 0;
    border: $arrow-width solid $white;
    border-top-color: transparent;
    border-bottom-color: transparent;
    position: absolute;
    top: calc(50% - #{$arrow-width});
  }
  &[x-placement^="right"] {
    margin-left: $arrow-width;
    .popper-arrow {
      border-left-width: 0;
      left: -#{$arrow-width};
    }
  }
  &[x-placement^="left"] {
    margin-right: $arrow-width;
    .popper-arrow {
      border-right-width: 0;
      right: -#{$arrow-width};
    }
  }

  .line-chart-tooltip-title {
    @include titillium-web-semibold;
    font-size: 0.875rem;
    color: $chart-axis-color;
    padding: 0.625rem 1rem;
    line-height: 0.875rem;
    border-bottom: 1px solid $chart-border-color;
    text-align: left;
  }

  .series-container {
    display: flex;
    flex-flow: row wrap;
    padding: 0.96875rem 1rem;
    .series-row {
      text-align: left;
      width: 42%;
      margin-bottom: 1rem;
      padding-right: 1rem;
      display: flex;
      .color-bar {
        margin-right: 0.59rem;
        width: 4px;
        height: 2rem;
        line-height: 2rem;
      }
      .series-name {
        @include titillium-web-regular;
        @extend %truncate;
        font-size: 0.75rem;
        line-height: 0.875rem;
        height: 0.875rem;
        width: 4.5rem;
      }
      .series-value {
        @include titillium-web-bold;
        font-size: 1rem;
        line-height: 0.875rem;
        height: 0.875rem;
      }
    }
  }
}
</style>
