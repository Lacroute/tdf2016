<template>
  <svg></svg>
</template>

<script>
var d3 = require('d3')
var topojson = require('topojson')

export default {
  data () {
    return {
    }
  },

  ready: function () {
    let width = 960
    let height = 960

    let svg = d3.select('svg')
      .attr('width', width)
      .attr('height', height)

    d3.json('static/tdf2016.json', function (error, data) {
      if (error) return console.error(error)

      console.log(data)

      let tdf = topojson.feature(data, data.objects.tdf2016)

      var projection = d3.geo.mercator()
        .scale(1)

      var path = d3.geo.path()
        .projection(projection)

      // Init the center of map
      var b = path.bounds(tdf)
      var s = 0.90 / Math.max(
                   (b[1][0] - b[0][0]) / width,
                   (b[1][1] - b[0][1]) / height
               )

      projection.scale(s)
      b = d3.geo.bounds(tdf)

      projection.center([(b[1][0] + b[0][0]) / 2, (b[1][1] + b[0][1]) / 2])
        .translate([width / 2, height / 2])

      svg.selectAll('stage')
        .data(tdf.features)
        .enter()
        .append('path')
        .attr('class', function (d) {
          var classname = d.properties.name.match(/Stage [0-9]+/)[0]
          classname = classname.replace(' ', '-').toLowerCase()
          console.log(classname)
          return classname
        })
        .attr('d', path)
        .attr('stroke', function () {
          return '#' + ((1 << 24) * Math.random() | 0).toString(16)
        })
        .attr('stroke-width', '1')
        .attr('fill', 'none')
        .append('text')
        .text(function (d) { return d.properties.name })
    })
  }
}
</script>

<!-- Add 'scoped' attribute to limit CSS to this component only -->
<style>
/*.stage-1 { fill: red; }*/
</style>
