<template>
  <div id="app">
    <div class="sidebar" ref="sidebar">
      <div class="search">
        <form>
          <select v-model="chosenCity">
            <option value="請選擇縣市" selected>請選擇縣市</option>
            <option v-for="(city,index) in cityList" :key="index" :value="city">{{city}}</option>
          </select>
          <select v-model="chosenArea" @change="getMaskApi">
            <option value="請選擇地區" selected>請選擇地區</option>
            <option v-for="(area,index) in areaList[0]" :key="index" :value="area">{{area}}</option>
          </select>
        </form>
        <p>有取得口罩數量的藥局有<span>{{pharmacies.length}}</span>家</p>
      </div>

      <ul class="result">
        <li v-for="pharmacy in pharmacies" :key="pharmacy.properties.id" @click="penTo(pharmacy.geometry.coordinates[1] , pharmacy.geometry.coordinates[0], pharmacy)">
          <h5>{{pharmacy.properties.name}}</h5>
          <p><img src="./assets/icon/map-marker.svg" alt="">{{pharmacy.properties.address}}</p>
          <p><img src="./assets/icon/phone.svg" alt="">{{pharmacy.properties.phone}}</p>
          <div :class="{ 'noMask' : !pharmacy.properties.mask_adult }">成人：
            <span v-if="pharmacy.properties.mask_adult">{{pharmacy.properties.mask_adult}}</span>
            <span v-else>已售完</span>
          </div>
          <div :class="{ 'noMask' : !pharmacy.properties.mask_child }">兒童：
            <span v-if="pharmacy.properties.mask_child">{{pharmacy.properties.mask_child}}</span>
            <span v-else>已售完</span>
          </div>
        </li>
      </ul>

      <button @click="toggleClass">
        <i class="fas fa-chevron-left" v-if="sidebarOpen"></i>
        <i class="fas fa-chevron-right" v-if="!sidebarOpen"></i>
      </button>
    </div>

    <div id="map">
      <div class="locateImg" @click="getUserPosition">
        <i class="fas fa-crosshairs"></i>
      </div>
    </div>
  </div>
</template>

<script>
import 'reset-css'
import data from './assets/AllData.json'
import markerIcon from './assets/icon/map-marker.svg'
import phoneIcon from './assets/icon/phone.svg'
import mapIcon from './assets/icon/mapIcon.svg'
import L from 'leaflet'

let mymap = {}
const customIcon = new L.Icon({ iconUrl: mapIcon })

export default {
  name: 'App',
  data () {
    return {
      sidebarOpen: true,
      chosenCity: '請選擇縣市',
      chosenArea: '請選擇地區',
      pharmacies: [],
      userPosition: []
    }
  },

  mounted () {
    // 先弄個初始化位置，之後會再定位user位置
    mymap = L.map('map', {
      center: [22.604799, 120.2976256],
      zoom: 16
    })
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      maxZoom: 18
    }).addTo(mymap)

    this.getUserPosition()
  },

  computed: {
    cityList () {
      return data.map(item => {
        return item.CityName
      })
    },
    areaList () {
      return data.map(item => {
        if (this.chosenCity === item.CityName) {
          return item.AreaList.map(area => {
            return area.AreaName
          })
        }
      }).filter(item => item)
    }
  },

  methods: {
    getMaskApi () {
      this.pharmacies = []
      const api = 'https://raw.githubusercontent.com/kiang/pharmacies/master/json/points.json'
      this.axios.get(api).then(response => {
        this.pharmacies = response.data.features.filter(item =>
          item.properties.county === this.chosenCity && item.properties.town === this.chosenArea
        )
        this.renderMap()
        // 飛到第一個藥局的位置
        mymap.panTo([this.pharmacies[0].geometry.coordinates[1], this.pharmacies[0].geometry.coordinates[0]], { animate: true, duration: 1 })
      })
    },

    renderMap () {
      // 清除之前的marker
      mymap.eachLayer((layer) => {
        if (layer instanceof L.Marker) {
          mymap.removeLayer(layer)
        }
      })
      // 插位置marker，並替換樣式
      this.pharmacies.forEach(pharmacy => {
        const { properties, geometry } = pharmacy

        L.marker([
          geometry.coordinates[1],
          geometry.coordinates[0]
        ], { icon: customIcon }).addTo(mymap).bindPopup(
        // 加入marker資訊
          `<h5>${properties.name}</h5>
          <p><img src=${markerIcon} alt="">
          <a href="https://www.google.com.tw/maps/place/${properties.address}" target="_blank">${properties.address}</a>
          </p>
          <p><img src=${phoneIcon} alt="">${properties.phone}</p>
          <div ${properties.mask_adult > 0 ? '' : 'class="noMask"'}>成人：
            <span>${properties.mask_adult > 0 ? properties.mask_adult : '已售完'}</span>
          </div>
          <div ${properties.mask_child > 0 ? '' : 'class="noMask"'}>兒童：
            <span>${properties.mask_child > 0 ? properties.mask_child : '已售完'}</span>
          </div>`
        )
      })
    },

    penTo (x, y, item) {
      mymap.panTo([x, y], { animate: true, duration: 1 })
      const marker = L.marker([x, y], { icon: customIcon }).addTo(mymap).bindPopup(
        `<h5>${item.properties.name}</h5>
        <p><img src=${markerIcon} alt="">
          <a href="https://www.google.com.tw/maps/place/${item.properties.address}" target="_blank">${item.properties.address}</a>
          </p>
        <p><img src=${phoneIcon} alt="">${item.properties.phone}</p>
        <div ${item.properties.mask_adult > 0 ? '' : 'class="noMask"'}>成人：
          <span>${item.properties.mask_adult > 0 ? item.properties.mask_adult : '已售完'}</span>
        </div>
        <div ${item.properties.mask_child > 0 ? '' : 'class="noMask"'}>兒童：
          <span>${item.properties.mask_child > 0 ? item.properties.mask_child : '已售完'}</span>
        </div>`
      )
      setTimeout(() => {
        // 晚一點再開，跟移動的動畫同步，太早開就不會移到中間
        marker.openPopup()
      }, 1000)
    },

    getUserPosition () {
      if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition(showPosition)
      } else { alert('無法取得您的定位') }

      const vm = this

      function showPosition (position) {
        vm.userPosition = [position.coords.latitude, position.coords.longitude]
        mymap.panTo([position.coords.latitude, position.coords.longitude], { animate: true, duration: 1 })

        // 取得並計算定位附近的藥局，並渲染在地圖上
        vm.pharmacies = []
        const api = 'https://raw.githubusercontent.com/kiang/pharmacies/master/json/points.json'
        vm.axios.get(api).then(response => {
          vm.pharmacies = response.data.features.filter(item =>
            Math.pow((item.geometry.coordinates[1] - vm.userPosition[0]), 2) + Math.pow((item.geometry.coordinates[0] - vm.userPosition[1]), 2) < 0.0001
          )
          vm.renderMap()

          vm.chosenCity = '請選擇縣市'
          vm.chosenArea = '請選擇地區'

          // 插上自己定位的marker
          const redIcon = new L.Icon({
            iconUrl: 'https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-red.png'
          })

          const userMarker = L.marker([
            position.coords.latitude,
            position.coords.longitude
          ], { icon: redIcon }).addTo(mymap).bindPopup('<strong>您的位置</strong>')

          setTimeout(() => {
          // 晚一點再開，跟移動的動畫同步，看起來也比較順
            userMarker.openPopup()
          }, 1000)
        })
      }
    },

    toggleClass () {
      this.sidebarOpen = !this.sidebarOpen
      if (this.sidebarOpen === false) {
        this.$refs.sidebar.classList.add('sidebar_close')
      } else {
        this.$refs.sidebar.classList.remove('sidebar_close')
      }
    }
  }
}
</script>

<style lang="scss">
@import "./assets/all.scss";
</style>
