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
      v-if="autoRout"
      :coords="autoRout"
      :marker-stroke="autoRoutStyle"
      marker-id="autoRout"
      marker-type="Polyline"
    />
    <ymap-marker
      v-if="planeRout"
      :coords="planeRout"
      :marker-stroke="planeRoutStyle"
      marker-id="planeRout"
      marker-type="Polyline"
    />
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
      autoRoutStyle: { color: '#000', width: 5 },
      planeRout: null,
      planeRoutStyle: { color: '#000', width: 5, style: 'dot' },
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

      const closest = this.mkad.geometry.getClosest(points).position;
      this.planeRout = [points, closest];

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
          },
          error => {
            console.log(`Возникла ошибка: ${error.message}`);
          }
        );

      this.ymaps.geocode(points).then(res => {
        // Выведем в консоль данные, полученные в результате геокодирования объекта.
        this.snackbarMsg = `
        ${res.geoObjects.get(0).properties.get('balloonContent')}<b>Расстоянние на автомобиле</b>:
        км`;
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
