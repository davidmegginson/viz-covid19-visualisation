<template>
  <div id="sankeyChart" ref="sankeyChart">
    <svg :width="width" :height="height">
      <pattern
        id="diagonalHatch"
        width="10"
        height="10"
        patternTransform="rotate(45 0 0)"
        patternUnits="userSpaceOnUse">
        <line x1="0" y1="0" x2="0" y2="10" style="stroke:black; stroke-width:1" />
      </pattern>
      <g>
        <rect
          v-for="(node) in nodes"
          v-bind="nodes"
          :key="node.index"
          :x="node.x0"
          :y="node.y0"
          :height="Math.max(node.y1-node.y0, 0)"
          :width="node.x1-node.x0"
          :fill="node.name=='» Direct Expenditure' ? 'url(#diagonalHatch)': color(node)"
          class="node" />
      </g>
      <g>
        <g
          v-for="(link) in links"
          v-bind="links"
          :id="`link-${link.index}`"
          :key="link.index"
          :class="selectedLink == link.index ? 'link linkHover' : 'link'"
          style="mix-blend-mode: multiply;">
          <linearGradient
            :id="`${link.index}-gradient`"
            :x1="link.source.x1"
            :x2="link.target.x0"
            gradientUnits="userSpaceOnUse">
            <stop
              :stop-color="color(link.source)"
              offset="0%" />
            <stop
              :stop-color="color(link.target)"
              offset="100%" />
          </linearGradient>
          <path
            :d="sankeyLinkPaths(link)"
            :stroke="`url(#${link.index}-gradient)`"
            :stroke-width="Math.max(1, link.width)"
            @mouseover="mouseoverLink(link.index)"
            @mouseleave="mouseleaveLink(link.index)"
          />
        </g>
      </g>
      <g font-family="sans-serif" font-size="12">
        <text
          v-for="(node) in nodes"
          v-bind="nodes"
          :key="node.index"
          :x="node.x0 < width / 2 ? node.x1 + 6 : node.x0 - 6"
          :y="(node.y1 + node.y0) / 2"
          :text-anchor="node.x0 < width / 2 ? 'start' : 'end'"
          dy="0.35em">
          {{ node.name }} <template v-if="node.name=='» Direct Expenditure'"> (no organisation)</template>
        </text>
      </g>
      <g font-family="sans-serif" font-size="12">
        <g
          v-for="(link) in links"
          v-bind="links"
          :key="`${link.index}-label`"
          :style="selectedLink == link.index ? 'display: block;' : 'display: none;'"
          class="linkText"
          @mouseover="mouseoverLink(link.index)"
          @mouseleave="mouseleaveLink(link.index)">
          <text
            :x="link.source.x1+(link.target.x0-link.source.x1)/2"
            :y="(link.y1 + link.y0) / 2"
            :width="link.width"
            dy="0.35em"
            text-anchor="middle">
            From {{ link.source.name }}<br />
          </text>
          <text
            :x="link.source.x1+(link.target.x0-link.source.x1)/2"
            :y="((link.y1 + link.y0) / 2)+15"
            :width="link.width"
            dy="0.35em"
            text-anchor="middle">
            To {{ link.target.name }}<br />
          </text>
          <text
            :x="link.source.x1+(link.target.x0-link.source.x1)/2"
            :y="((link.y1 + link.y0) / 2)+30"
            :width="link.width"
            dy="0.35em"
            text-anchor="middle">
            USD {{ numberFormatter(link.value) }}<br />
          </text>
        </g>
      </g>
    </svg>
  </div>
</template>
<style>
#sankeyChart {
  width: 100%;
  height: 400px;
}
.node rect {
  cursor: move;
  fill-opacity: .9;
  shape-rendering: crispEdges;
}
.node text {
  pointer-events: none;
  text-shadow: 0 1px 0 #fff;
}
.link {
  fill: none;
  stroke: #000;
  stroke-opacity: .3;
}
.linkHover {
  stroke-opacity: .5;
}
.linkText {
  fill: #000000;
  text-shadow: 1px 1px 3px #ffffff;
  cursor: default;
}
</style>
<script>
import { select as d3Select } from 'd3-selection'
import { scaleOrdinal as d3ScaleOrdinal, schemeCategory10 as d3SchemeCategory10 } from 'd3'
import { sankey as d3Sankey, sankeyLinkHorizontal as d3SsankeyLinkHorizontal } from 'd3-sankey'
export default {
  name: 'SankeyChart',
  props: ['chartData'],
  data () {
    return {
      chart: null,
      width: 10,
      height: 10,
      selectedLink: null
    }
  },
  computed: {
    colors () {
      return d3ScaleOrdinal(d3SchemeCategory10)
    },
    nodes () {
      return this.sankey.nodes
    },
    links () {
      return this.sankey.links
    },
    sankey () {
      const nodes = this.chartData.nodes
      const links = this.chartData.links
      const sankey = d3Sankey()
        .nodeId(d => d.name)
        .nodeWidth(20)
        .nodePadding(8)
        .extent([[1, 5], [this.width - 1, this.height - 10]])
      return sankey({
        nodes: nodes.map(d => Object.assign({}, d)),
        links: links.map(d => Object.assign({}, d))
      })
    },
    sankeyLinkPaths () {
      return d3SsankeyLinkHorizontal()
    }
  },
  mounted () {
    this.makeChart()
    window.addEventListener('resize', this.onResize)
    this.onResize()
  },
  methods: {
    mouseoverLink (index) {
      this.selectedLink = index
    },
    mouseleaveLink (index) {
      this.selectedLink = null
    },
    sankeyLinkPath (d) {
      return this.sankeyLinkPaths(d)
    },
    numberFormatter (value) {
      if (value === 0) { return '0' }
      return value
        ? value.toLocaleString(undefined, {
          maximumFractionDigits: 0
        })
        : ''
    },
    color (d) {
      if (d.name === '» Direct Expenditure') { return '#bbbbbb' }
      return this.colors(d.name)
    },
    makeChart () {
      d3Select('#sankeyChart')
        .append('svg')
        .attr('width', this.width)
        .attr('height', this.height)
    },
    onResize () {
      this.width = this.$el.offsetWidth
      this.height = 500
    }
  }
}
</script>
