<template>
  <div class="race-selector" v-if="currentRace != null">
    <race v-for="race in races | orderBy 'stage'"
      :race="race">
    </race>
  </div>

  <section class="race-overview">
    <header>
      <div v-if="currentRace != null">
        <div class="navigator">
          <p
            v-show="previousRace > 0"
            v-on:click=highlightFeature(previousRace) >
            Étape {{ previousRace }} ←
          </p>
          <p v-show="nextRace <= races.length" v-on:click=highlightFeature(nextRace)>
            → Étape {{ nextRace }}
          </p>
        </div>

        <p>
          Étape {{ currentRace.stage }}
          <span>{{ currentRace.data.name }}<span>
        </p>
      </div>
      <div v-else class="start">
        <p v-on:click=highlightFeature(1)>
          Tour de France 2016
          <span>Cliquez pour commencer</span>
        </p>
      </div>
    </header>
    <div class="race-details" v-show="currentRace != null">
      <span v-if="currentRace != null">
        Distance {{ currentRace.data.distance }} km
      </span>
      <span v-if="currentRace != null">
        Dénivelé max {{ elevation }} m
      </span>
      <svg id="summary"></svg>
    </div>
  </section>
  <div id="mapid"></div>
</template>

<script>
import Race from './Race.vue'

var L = require('leaflet')
var d3 = require('d3')
var Vue = require('vue')
Vue.use(require('vue-resource'))
var map
var layerGroup

// Make somme d3 global
var svg
var width = 430
var height = 100
var x = d3.scale.linear()
  .range([0, width])

var y = d3.scale.linear()
  .range([height, 0])

var valueLine = d3.svg.line()
  .x(function (d, i) { return x(i) })
  .y(function (d) { return y(+d[2]) })
// END D3

export default {
  name: 'Tdf',
  components: {Race},

  data () {
    return {
      races: [],
      currentRace: null
    }
  },

  computed: {
    // Return the previous race
    previousRace: function () {
      if (this.currentRace) return this.currentRace.stage - 1
    },

    // Return the next race
    nextRace: function () {
      if (this.currentRace) return this.currentRace.stage + 1
    },

    elevation: function () {
      return this.currentRace.data.elevation[1] - this.currentRace.data.elevation[0]
    }
  },

  ready: function () {
    // Get asynchronous data.
    this.$http.get('static/tdf2016.geojson').then(
      function (data) {
        this.jsonReady(JSON.parse(data.body))
      },
      function (error) {
        console.log(error)
      }
    )

    // Build the map.
    map = L.map('mapid').setView([46, 2], 6)

    // Set custom tile layer from mapbox
    L.tileLayer('https://api.tiles.mapbox.com/v4/{id}/{z}/{x}/{y}.png?access_token={accessToken}', {
      attribution: 'Map data &copy; <a href="http://openstreetmap.org">OpenStreetMap</a> contributors, <a href="http://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>, Imagery © <a href="http://mapbox.com">Mapbox</a>',
      maxZoom: 13,
      id: 'mapbox.light',
      accessToken: 'pk.eyJ1IjoibGFjcm91dGUiLCJhIjoiY2lxcDhoZXpqMDA1M2h4bms4YWRyNGlzbiJ9.VLlfBQGaxNFE9M_Ug3YXhQ'
    }).addTo(map)
  },

  methods: {

    // Defult style for the race vectors.
    defaultStyle: function () {
      return {
        color: '#000',
        weight: 5,
        opacity: 0.4
      }
    },

    // Default style for the cities
    defaultStyleCircle: function (c, f) {
      return {
        color: '#333',
        fillColor: '#000',
        fillOpacity: 0.3
      }
    },

    // When a race is on focus.
    hightlightStyle: function () {
      return {
        color: '#f5da3f',
        weight: 3,
        opacity: 1
      }
    },

    // Highlight city circle start
    hightlightStyleCircleStart: function () {
      return {
        color: '#00A388',
        fillColor: '#79BD8F',
        fillOpacity: 0.8
      }
    },

    // Highlight city circle end
    hightlightStyleCircleEnd: function () {
      return {
        color: '#CD2C24',
        fillColor: '#F2836B',
        fillOpacity: 0.8
      }
    },

    // Helper to extract the integer stage from the string name.
    getStageFromName (name) {
      return parseInt(name.match(/Stage [0-9]+/)[0].replace(/\D/g, ''))
    },

    // When data is loaded.
    jsonReady: function (tdf) {
      // Reference to the component scope.
      var that = this

      // Hot reload fix
      this.races = []
      this.currentRace = null

      // Draw the races.
      layerGroup = L.geoJson(
        tdf,
        {
          onEachFeature: function (feature, layer) {
            // Clean data to build better format.
            var featureName = feature.properties.name
            var stage = that.getStageFromName(featureName)
            var distance = featureName.match(/[0-9]+ km/)[0].replace(/\D/g, '')
            featureName = featureName.split(':')[1]
            var name = featureName.split('km')[1]
            var coords = feature.geometry.coordinates
            var start = coords[0]
            var end = coords[coords.length - 1]
            let elevation = that.arrayMinMax(feature.geometry.coordinates)

            // Store it.
            let race = {
              stage: stage,
              data: {
                raw: featureName,
                name: name,
                distance: parseInt(distance),
                elevation: elevation
              },
              start: start,
              end: end,
              selected: false
            }

            // Draw default start and end circles.
            race.start.circle = that.circleFactory(race.start, that.defaultStyleCircle())
            race.end.circle = that.circleFactory(race.end, that.defaultStyleCircle())

            that.races.push(race)

            layer.on(
              {
                // highlight the race on click
                'click': function (e) {
                  var stage = that.getStageFromName(e.target.feature.properties.name)
                  that.highlightFeature(stage)
                }
              })
          },

          style: this.defaultStyle
        }
      ).addTo(map)

      // Iterate through the layer because the _leaflet_id is not readable in onEachFeature function
      let l = layerGroup._layers
      for (let i in l) {
        let layer = l[i]
        let layerId = layer._leaflet_id
        let stage = this.getStageFromName(layer.feature.properties.name)

        let race = this.races.find(function (r) {
          return r.stage === stage
        })

        race.layerId = layerId
      }
      // Init svg
      this.drawRaceSummary()
    },

    // Main function for higlighting.
    highlightFeature: function (stage) {
      this.currentRace = this.updateRaces(stage)
      this.updateRaceSummary()
      this.currentRace.start.circle.setStyle(this.hightlightStyleCircleStart())
      this.currentRace.end.circle.setStyle(this.hightlightStyleCircleEnd())
      this.updateCamera()
    },

    // Update all the races.
    updateRaces: function (stage) {
      let r
      let that = this
      this.races.some(function (race) {
        if (race.selected) {
          layerGroup.getLayer(race.layerId).setStyle(
            that.defaultStyle()
          )
          race.start.circle.setStyle(that.defaultStyleCircle())
          race.end.circle.setStyle(that.defaultStyleCircle())
          race.selected = false
        }
        if (race.stage === stage) {
          race.selected = true
          r = race
          layerGroup.getLayer(r.layerId).setStyle(
            that.hightlightStyle()
          )
        }
      })

      return r
    },

    // Draw a graphic who summarize the race.
    drawRaceSummary: function () {
      var coords = [[0, 0, 0]]

      svg = d3.select('#summary')

      svg.append('svg:path')
        .attr('class', 'line-race')
        .attr('d', valueLine(coords))

      // console.log(svg.selectAll('path.line-race'))
    },

    // Update the line with the current coordinates.
    updateRaceSummary: function () {
      var coords = layerGroup.getLayer(this.currentRace.layerId).feature.geometry.coordinates

      x.domain([0, coords.length])
      y.domain([0, this.currentRace.data.elevation[1]])

      svg.select('path.line-race')
          .transition()
          .attr('d', valueLine(coords))
					.duration(300)
    },

    // Move the camera to the specific race
    updateCamera: function () {
      var bounds = L.latLngBounds(
        L.latLng(this.currentRace.start[1], this.currentRace.start[0]),
        L.latLng(this.currentRace.end[1], this.currentRace.end[0])
      )

      // Add padding because of the overview
      map.fitBounds(
        bounds,
        {
          maxZoom: 10,
          animate: true,
          paddingTopLeft: [300, 300],
          paddingBottomRight: [0, 200]
        }
      )
    },

    // Helper to draw circles.
    circleFactory: function (coords, style) {
      return L.circle(
        [coords[1], coords[0]],
        500,
        style
      ).addTo(map)
    },

    // Helper to find the min and max of the altitude cell.
    arrayMinMax: function (arr) {
      let len = arr.length
      let min = Infinity
      let max = -Infinity
      let cell

      while (len--) {
        cell = arr[len][2]
        if (cell < min) {
          min = cell
        }
        if (cell > max) {
          max = cell
        }
      }

      return [min, max]
    }
  },

  // Triggers when race emits events.
  events: {

    // On click
    'race-selected': function (race) {
      this.highlightFeature(race.stage)
    }
  }
}
</script>

<style>
/*.stage-1 { fill: red; }*/
#mapid {
  height: 100%;
  width: 100%;
}

.start{
  cursor: pointer;
}

.race-overview{
  font-family: 'Lato', sans-serif;
  font-weight: 300;
  position: absolute;
  top: 10px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 1;
  background: #f5da3f;
  color: #000;
  min-width: 450px;
}

.race-overview p{
  text-align: center;
  font-size: 1.250rem;
  margin: 1em;
  letter-spacing: 0.1em;
}

.race-overview p span{
  display: block;
  margin: auto;
  font-size: 1.500rem;
  text-transform: uppercase;
  margin-top: 15px;
  font-weight: 700;
}

.race-selector{
  background: rgba(0, 0, 0, 0.1);
  color: #fff;
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  z-index: 9999;
  left: 10px;
}

.navigator{
  position: absolute;
  width: 150%;
  left: 50%;
  transform: translateX(-50%);
}
.navigator p{
  color: #fff;
  background: #000;
  display: inline-block;
  padding: 1em;
  margin: 0 -2.7em;
  cursor: pointer;
}

.navigator p + p{
  float: right;
}

.race-details{
  font-size: 0;
  padding-bottom: 10px;
}
.race-details span{
  display: inline-block;
  width: 50%;
  text-align: center;
  font-size: 1.150rem;
}

#summary{
  width: 430px;
  height: 100px;
  margin: 10px auto 0;
  display: block;
}

#summary path {
  stroke: black;
  stroke-width: 1;
  fill: none;
}
</style>
