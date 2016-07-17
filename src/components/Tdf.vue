<template>
  <div class="race-selector">
    <race v-for="race in races | orderBy 'stage'"
      :race="race">
    </race>
  </div>
  <section class="race-overview">
    <header>
      <div v-if="currentRace != null">
        <div class="navigator">
          <p v-show="previousRace > 0" v-on:click=highlightFeature(previousRace)>
            Étape {{ previousRace }} ←
          </p>
          <p v-show="previousRace < races.length" v-on:click=highlightFeature(nextRace)>
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
  </section>
  <div id="mapid"></div>
</template>

<script>
import Race from './Race.vue'

var L = require('leaflet')
var Vue = require('vue')
Vue.use(require('vue-resource'))
var map
var layerGroup

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
      return this.currentRace.stage - 1
    },

    // Return the next race
    nextRace: function () {
      return this.currentRace.stage + 1
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
      maxZoom: 18,
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
      var that = this

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

            // Store it.
            let race = {
              stage: stage,
              data: {
                raw: featureName,
                name: name,
                distance: parseInt(distance)
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
                'click': function (e) {
                  // highlight the feature
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
    },

    // Main function for higlighting.
    highlightFeature: function (stage) {
      this.currentRace = this.updateRaces(stage)
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

    // Move the camera to the specific race
    updateCamera: function () {
      var bounds = L.latLngBounds(
        L.latLng(this.currentRace.start[1], this.currentRace.start[0]),
        L.latLng(this.currentRace.end[1], this.currentRace.end[0])
      )

      // map.panInsideBounds(bounds)
      map.fitBounds(
        bounds,
        {
          animate: true,
          pan: {
            animate: true,
            duration: 1
          }
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
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 1;
  background: #f5da3f;
  color: #000;
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
</style>
