<template>
  <div class="panel">
    <p class="panel-title">{{ result.title }}</p>

    <div v-if="result">
      {{ properties.adreca }}
    </div>

    <div class="action">
      <a @click="zoomResult()">
        <span class="oi oi-zoom-in"></span>
        Zoom</a>
    </div>
  </div>
</template>

<script>
import L from 'leaflet'

import Icon from 'vue-awesome/components/Icon'
import 'vue-awesome/icons/spinner'

import BaseResultMixin from './BaseResultMixin.js'

export default {
  mixins: [BaseResultMixin],
  components: {
    Icon
  },
  props: ['map'],
  computed: {
    properties () {
      if (this.result.geojson) {
        return this.result.geojson.properties
      } else {
        return {}
      }
    },
    resultsLayer () {
      return this.$store.state.resultsLayer
    },
    isResultClickable () {
      return this.result.geojson
    }
  },
  destroyed () {
    if (this.result.layer) {
      this.result.layer.removeFrom(this.resultsLayer)
      this.$store.commit('selectResult', null)
    }
  },
  methods: {
    isValidResult () {
      if (!this.isResultClickable) {
        console.log('Result is not clickable')
        return false
      }

      let element = this.result
      if (!element.geojson) {
        console.log('Result is not Geojson')
        return false
      }
      if (!('type' in element.geojson && element.geojson['type'] === 'Feature')) {
        console.log('Result is not Feature')
        return false
      }

      return true
    },
    viewResult () {
      if (!this.isValidResult()) {
        return
      }

      let element = this.result
      let mapInfo = this.map.giscube.getMapInfo()
      let bounds
      let geom = element.geojson.geometry

      if (geom.type === 'Point') {
        var point = L.latLng(
          geom.coordinates[1],
          geom.coordinates[0])
        bounds = point.toBounds(
          Math.min(mapInfo.visibleHeightMeters, mapInfo.visibleWidthMeters) * 0.5)
      } else {
        if (!element.layer) {
          element.layer = L.geoJSON.geometryToLayer(geom)
        }
        bounds = element.layer.getBounds().pad(0.1)
      }
      this.resultsLayer.addLayer(element.layer)

      let visible = mapInfo.visibleBounds.contains(bounds)

      if (!visible || !mapInfo.isVisible) {
        let latLng = bounds.getCenter()
        // use zoom/pan options in flyTo instead?
        if (mapInfo.isVisible) {
          latLng.lng = latLng.lng - mapInfo.coveredLng / 2
        }

        // do we need a smaller zoom?
        let maxZoom = this.map.getBoundsZoom(bounds)

        try {
          this.map.flyTo(latLng, this.map.getZoom() > maxZoom ? maxZoom : {maxZoom: 19})
        } catch (e) {
          console.error(e)
          mapInfo = this.map.giscube.getMapInfo()
          if (!mapInfo.isViewValid) {
            console.log('view is not valid, try to recover view')
            this.map.setView(latLng, 19)
          }
        }
      }
      element.layer.openPopup()
    },
    zoomResult () {
      if (!this.isValidResult()) {
        return
      }

      let element = this.result
      let mapInfo = this.map.giscube.getMapInfo()
      let geojson = L.geoJSON(element.geojson.geometry)
      let bounds = geojson.getBounds().pad(0.1)

      this.map.flyToBounds(bounds, {
        paddingTopLeft: [mapInfo.sidebarWidthPx, 0]
      })
    }
  }
}
</script>

<style>
.list-group-item {
  min-height: 65px;
}
.panel {
    padding: 0 20px 15px 20px;
}
.panel-title {
  font-size: 1.5em;
  font-family: 'Lato', sans-serif;
  font-weight: 400;
  margin-bottom: 10px;
}
div.action {
  cursor: pointer;
  line-height: 1.5em;
  display: inline-block;
  padding: 10px;
}
</style>
