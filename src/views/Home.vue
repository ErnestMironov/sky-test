<template>
  <yandex-map
    :settings="settings"
    @click="setMarker"
    @map-was-initialized="mapInit"
    :coords="coords"
    :zoom="zoom"
  >
    <v-snackbar top right timeout="-1" v-model="snackbar">
      <div v-html="snackbarMsg"></div>
      <template v-slot:action="{ attrs }">
        <v-btn color="pink" text v-bind="attrs" @click="snackbar = false">
          Close
        </v-btn>
      </template>
    </v-snackbar>

    <ymap-marker
      v-if="markerCoords"
      :coords="markerCoords"
      marker-id="marker"
      hint-content="some hint"
    />
  </yandex-map>
</template>

<script>
import { yandexMap, ymapMarker, loadYmap } from 'vue-yandex-maps';
import { mapGetters } from 'vuex';

export default {
  name: 'Home',
  components: { yandexMap, ymapMarker },
  data() {
    return {
      snackbar: false,
      snackbarMsg: '',
      coords: [55.76, 37.64],
      autoRout: null,
      planeRout: null,
      zoom: 10,
      map: null,
      mkad: null,
      markerCoords: null,
      settings: {
        apiKey: '2a51d234-e1e7-496f-9478-a9a99d9d9673',
        lang: 'ru_RU',
        coordorder: 'latlong',
        version: '2.1'
      },
      ymaps: null
    };
  },
  computed: {
    ...mapGetters({
      mkadCoords: 'getMkadCoords'
    })
  },
  methods: {
    mapInit(map) {
      this.map = map;

      this.mkad = new this.ymaps.Polygon(this.mkadCoords, {});

      this.map.geoObjects.add(this.mkad);
    },
    setMarker(e) {
      this.markerCoords = e.get('coords');
    },
    setRoad(points) {
      this.snackbar = false;
      const planeRoutStyle = {
        strokeColor: '#000000',
        strokeWidth: 4,
        strokeStyle: 'dot'
      };
      const closest = this.mkad.geometry.getClosest(points).position;

      if (this.planeRout) this.map.geoObjects.remove(this.planeRout);
      if (this.planeRout) this.map.geoObjects.remove(this.autoRout);

      this.planeRout = new this.ymaps.Polyline([points, closest], {}, planeRoutStyle);
      this.map.geoObjects.add(this.planeRout);

      let autoRoutLength;
      const planeRoutLength = (this.planeRout.geometry.getDistance() / 1000).toFixed(2);

      this.ymaps
        .route([points, closest], {
          mapStateAutoApply: true
        })
        .then(
          route => {
            const routeCoords = [];

            for (let i = 0; i < route.getPaths().getLength(); i += 1) {
              const way = route.getPaths().get(i);
              const segments = way.getSegments();
              for (let j = 0; j < segments.length; j += 1) {
                routeCoords.push(...segments[j].getCoordinates());
              }
            }

            this.autoRout = routeCoords;
            this.autoRout = new this.ymaps.Polyline(
              routeCoords,
              {},
              { strokeColor: '#000000', strokeWidth: 4 }
            );
            this.map.geoObjects.add(this.autoRout);
            autoRoutLength = (this.autoRout.geometry.getDistance() / 1000).toFixed(2);
          },
          error => {
            console.log(`Возникла ошибка: ${error.message}`);
          }
        );

      this.ymaps.geocode(points).then(res => {
        this.snackbarMsg = `
        ${res.geoObjects.get(0).properties.get('balloonContent')}<b>Расстоянние на автомобиле:</b>
        ${autoRoutLength}км<br /><b>Расстоянние по воздуху:</b>${planeRoutLength}км`;
      });
      this.snackbar = true;
    }
  },
  watch: {
    markerCoords(coords) {
      this.setRoad(coords);
    }
  },
  async mounted() {
    await loadYmap(this.settings);
    this.ymaps = window.ymaps;
  }
};
</script>

<style lang="scss" scoped>
.ymap-container {
  height: 100vh;
  width: 100%;
}
</style>
