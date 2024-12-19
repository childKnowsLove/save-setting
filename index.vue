<template>
  <div class="map-monitor" :class="{ 'flag-hide-btn': flagHideVisible }">
    <div v-if="loadingFlagMap" class="content-block" style="position: absolute; z-index: 999; background: white; width: 100%; height: 100%;">
      <div class="loading-dasboard-traffic">
        <svg-icon icon-class="loadingDashboard" class="svg-icon-loading" />
      </div>
    </div>
    <div id="container-map">
      <div id="map" ref="map" class="map" />
      <div id="popup-device" class="ol-popup-device">
        <template v-if="deviceDetail">
          <div class="info-divice px-3 py-1">
            <span class="text-break">
              {{ `${deviceDetail.name} - ` }}
              <span v-if="deviceDetail.siteId">{{
                deviceDetail.siteId.name
              }}</span>
            </span>
          </div>
        </template>
      </div>
      <div
        id="popup-events"
        class="ol-popup-device"
        :class="'fit-content-' + groupEvents.length"
      >
        <div v-if="groupEvents.length > 0" class="ol-popup-events scroll-bar">
          <template v-for="(event, i) in groupEvents">
            <div
              v-if="event.statusDevice && listDeviceFilter.includes(event.type)"
              :key="i + 'device'"
              class="mb-2"
              role="button"
              @click="showDialogDevice(event.id, event.type)"
            >
              <el-tooltip effect="dark" :content="event.name" placement="top">
                <img :src="getIconCam(event)" :title="event.name">
              </el-tooltip>
            </div>
            <div
              v-else
              :key="i + 'event'"
              class="mb-2"
              role="button"
              @click="eventClick(event)"
            >
              <el-tooltip
                effect="dark"
                placement="top"
                :content="event.name ? event.name : getJobName(event.eventCode)"
              >
                <img
                  :src="getIconEvent(
                    event.eventCode,
                    event.speedLimit,
                    event.eventStatus
                  )
                  "
                  :title="event.name ? event.name : getJobName(event.eventCode)"
                >
              </el-tooltip>
            </div>
          </template>
          <font-awesome-icon
            icon="fa-solid fa-xmark"
            class="close-popup"
            @click="hiddenPopup()"
          />
        </div>
      </div>
      <div id="stage-name">
        <div v-if="stageName">{{ stageName }}</div>
      </div>
    </div>
    <el-dialog
      id="popup"
      :visible.sync="showPopup"
      class="dialog-device"
      :width="widthPopup"
      :destroy-on-close="true"
      :close-on-press-escape="false"
      @closed="closePopup"
    >
      <template
        v-if="typeItem == 1 || typeItem == 2 || typeItem == 3 || typeItem == 8"
      >
        <span slot="title" class="text-white">{{ $t('config.deviceInformation') }}</span>
        <el-row>
          <el-col ref="cameraInfo" :span="8" class="detail-info">
            <div class="mb-1">
              <div class="item-icon">
                <img :src="labelIcon" class="icon-detail">
              </div>
              <span class="item item__hidden">
                <span class="title">{{ $t('config.deviceName') }}</span>
                <br>
                <el-tooltip
                  :content="itemInfo.label ? itemInfo.label : itemInfo.name"
                  placement="top"
                >
                  <span>{{
                    itemInfo.label ? itemInfo.label : itemInfo.name
                  }}</span>
                </el-tooltip>
              </span>
            </div>
            <div class="mb-1">
              <div class="item-icon">
                <img :src="idIcon" class="icon-detail">
              </div>
              <span class="item item__hidden">
                <span class="title">ID</span>
                <br>
                <el-tooltip :content="itemInfo.cameraKey" placement="top">
                  <span>{{ itemInfo.cameraKey }}</span>
                </el-tooltip>
              </span>
            </div>
            <div class="mb-1">
              <div class="item-icon">
                <img :src="deviceTypeIcon" class="icon-detail">
              </div>
              <span class="item">
                <span class="title">{{ $t('config.deviceType') }}</span>
                <br>
                <span>{{ getCameraType(itemInfo.cameraType, 'string') }}</span>
              </span>
            </div>
            <div class="mb-1">
              <div class="item-icon">
                <img :src="milePostIcon" class="icon-detail">
              </div>
              <span class="item">
                <span class="title">{{ $t('config.workflow') }}</span>
                <br>
                <span>KM {{ itemInfo.siteId.km }}+ {{ itemInfo.siteId.m }}</span>
                <span>{{ ` ${$detectDirection(itemInfo.directionCode, 'value')}` }}</span>
              </span>
            </div>
            <div class="mb-1">
              <div class="item-icon">
                <img
                  src="@/assets/images/icon120822/site.png"
                  class="icon-detail"
                >
              </div>
              <span class="item">
                <span class="title">{{ $t('config.location') }}</span>
                <br>
                <span>
                  {{ itemInfo.siteId.districtId.prefix }}
                  {{ itemInfo.siteId.districtId.name }}
                </span>
              </span>
            </div>
            <div class="mb-1">
              <div class="item-icon">
                <img :src="statusIcon" class="icon-detail">
              </div>
              <span class="item">
                <span class="title">{{ $t('config.status') }}</span>
                <br>
                <span v-if="itemInfo.cameraStatus == 'RUNNING'">Kết nối</span>
                <span v-else>Mất kết nối</span>
              </span>
            </div>
          </el-col>
          <el-col :span="16" class="media-info fill rounded-0">
            <div id="vds-cam" class="cam-detail">
              <template v-if="itemInfo">
                <MileStoneLive
                  v-if="itemInfo"
                  :key="flagMileStone"
                  :camera-config="itemInfo"
                  :camera-real="itemInfo.id"
                />
              </template>
            </div>
          </el-col>
        </el-row>
      </template>
    </el-dialog>
    <ClockDigit />
  </div>
</template>
<script>
import 'ol/ol.css'
import Map from 'ol/Map'
import Feature from 'ol/Feature'
import Overlay from 'ol/Overlay'
import View from 'ol/View'
import { defaults as defaultControls } from 'ol/control'
import {
  DragRotateAndZoom,
  defaults as defaultInteractions
} from 'ol/interaction'
import { Tile as TileLayer, Vector as VectorLayer } from 'ol/layer'
import { Vector as VectorSource } from 'ol/source'
import { Point, LineString, MultiPolygon } from 'ol/geom'
import {
  Style,
  Icon,
  Text,
  Fill,
  Stroke,
  Circle as CircleStyle
} from 'ol/style'
import { fromLonLat, toLonLat, transformExtent } from 'ol/proj'
import TileWMS from 'ol/source/TileWMS'
import { getCenter } from 'ol/extent'
import Resource from '@/api/resource'
import cameraQuanSatIcon from '@/assets/images/map/cam_giam_sat_online.png'
import cameraQuanSatNotIcon from '@/assets/images/map/cam_giam_sat_not.png'
import cameraQuanSatOfflineIcon from '@/assets/images/icons_map/cam_giam_sat_offline.png'
import cameraGiamSatIcon from '@/assets/images/map/cam_giam_sat_online.png'
import cameraGiamSatNotIcon from '@/assets/images/map/cam_giam_sat_not.png'
import cameraGiamSatOfflineIcon from '@/assets/images/icons_map/cam_giam_sat_offline.png'
import cameraDoXeIcon from '@/assets/images/map/cam_ai_online.png'
import cameraDoXeNotIcon from '@/assets/images/map/cam_ai_not.png'
import cameraDoXeOfflineIcon from '@/assets/images/icons_map/cam_ai_offline.png'
import bienBaoVmsIcon from '@/assets/images/map/bien_bao_dien_tu.png'
import bienBaoVmsNotIcon from '@/assets/images/map/bien_bao_dien_tu_not.png'
import bienBaoVmsOfflineIcon from '@/assets/images/icons_map/bien_bao_dientu_offline.png'
import tramThuPhiIcon from '@/assets/images/icons_map/tollStation.png'
import cabinetIcon from '@/assets/images/icons_map/cabinet.png'
import warningIcon from '@/assets/images/icons_map/warning.png'
import tainanIcon from '@/assets/images/event/jobs/tainan.png'
import xehongIcon from '@/assets/images/event/jobs/xehong.png'
import vatcanIcon from '@/assets/images/event/jobs/vatcan.png'
import fireIcon from '@/assets/images/event/jobs/chaykhoi.png'
import rainIcon from '@/assets/images/event/jobs/muato.png'
import landslideIcon from '@/assets/images/event/jobs/satlo.png'
import snowIcon from '@/assets/images/event/jobs/tuyetroi.png'
import fogIcon from '@/assets/images/event/jobs/suongmu.png'
import crowdIcon from '@/assets/images/event/jobs/damdong.png'
import contructionIcon from '@/assets/images/event/jobs/duongthicong.png'
import blockLaneIcon from '@/assets/images/event/jobs/donglan.png'
import roadBlockIcon from '@/assets/images/event/jobs/camduong.png'
import dongloiravaoBlockIcon from '@/assets/images/event/jobs/dongloiravao.png'
import nhietdotangIcon from '@/assets/images/event/jobs/nhietdotang.png'
import phahoaiIcon from '@/assets/images/event/jobs/phahoai.png'
import duonghongIcon from '@/assets/images/event/jobs/duonghong.png'
import nothingIcon from '@/assets/images/icons_map/nothing.png'
import replaceIcon from '@/assets/images/icons/loai_sukien.png'
import replaceCamOnIcon from '@/assets/images/icons_map/group_camera.png'
import replaceCamOffIcon from '@/assets/images/icons_map/group_camera.png'
import nbhnIcon from '@/assets/images/icons_map/nbhn.png'
import hnnbIcon from '@/assets/images/icons_map/hnnb.png'
import DistrictList from './data/DistrictList.json'
import EntranceExitList from './data/EntranceExitList.json'
import config from '@/config'
import MileStoneLive from './components/liveCam'
import { getLaneImage } from './constant'
import { numberWithCommas } from '@/utils'
import { library } from '@fortawesome/fontawesome-svg-core'
import { faXmark } from '@fortawesome/free-solid-svg-icons'
import { FontAwesomeIcon } from '@fortawesome/vue-fontawesome'
import ClockDigit from '@/components/ClockDigit/index.vue'
library.add(faXmark)
import ResourceNew from '@/api/resourceNew'
const listStageResource = new ResourceNew('system/map/stages')
const listDeviceStageResource = new ResourceNew('system/map/system-device')
const detailJobResource = new ResourceNew('system/management/common/job')
const positionCenterResource = new ResourceNew('system/map/stage/mid-point')
const centerPoint = process.env.VUE_APP_MAP_CENTER_POINT.split(',')
export default {
  components: {
    MileStoneLive,
    FontAwesomeIcon,
    ClockDigit
  },
  props: {
    typePage: {
      type: String,
      default: 'cctv'
    }
  },
  sockets: {
    receiveNotificationMap(data) {
      console.log('receiveNotificationMap ~ data:', data)
      const { content } = data
      if (content && typeof obj === 'object') {
        if ('deviceId' in content) {
          this.setDeviceStatus(content)
        }
      }
    }
  },
  data() {
    return {
      imgLaneActive: require('@/assets/images/station/lane-active.png'),
      imgLaneBlock: require('@/assets/images/station/lane-block.png'),
      windowHeight: window.innerHeight,
      mapInstance: null,
      popup: null,
      flagMileStone: 1,
      popupGroupEvents: null,
      selectedIcon: null,
      zoom: 10,
      minZoom: 9.5,
      maxZoom: 14,
      centerLongLat: {
        longitude: centerPoint[1],
        latitude: centerPoint[0]
      },
      currentTime: moment().format('DD/MM/YYYY HH:mm'),
      widthPopup: '900px',
      eventListAllow: [
        'ACCIDENT',
        'BROKENVEHICLE',
        'BARRIER',
        'FIRE',
        'RAIN',
        'LANDSLIDE',
        'SNOW',
        'FOG',
        'CROWD',
        'HIGH_TEMPERATURE',
        'DAMAGED_ROAD',
        'DESTRUCTION',
        'CONSTRUCTION'
      ],
      jobListAllow: [
        'BLOCKLANE',
        'BLOCKROAD',
        'LIMIT_SPEED',
        'CLOSE_OPEN_ENTRANCE_EXIT'
      ],
      colorList: config.trafficColorList,
      jobNameList: config.listJobMap,
      eventNameList: {
        ACCIDENT: 'Tai nạn',
        BROKENVEHICLE: 'Xe hỏng',
        BARRIER: 'Vật cản',
        FIRE: 'Hỏa hoạn',
        RAIN: 'Mưa',
        LANDSLIDE: 'Sạt lở',
        SNOW: 'Tuyết rơi',
        FOG: 'Sương mù',
        CROWD: 'Biểu tình, đám đông',
        HIGH_TEMPERATURE: 'Nhiệt độ cao',
        DAMAGED_ROAD: 'Đường hư hỏng',
        DESTRUCTION: 'Phá hoại',
        CONSTRUCTION: 'Đường đang thi công',
        CROWDING: 'Sự cố, va chạm giao thông'
      },
      idIcon: require('@/assets/images/icon120822/id_label.png'),
      labelIcon: require('@/assets/images/icon120822/id_label.png'),
      deviceTypeIcon: require('@/assets/images/icon120822/device_type.png'),
      milePostIcon: require('@/assets/images/icon120822/mile_post.png'),
      statusIcon: require('@/assets/images/icon120822/status.png'),
      displayIcon: require('@/assets/images/icons_map/display.png'),
      imgStationIcon: require('@/assets/images/icons_map/weighing-station.png'),
      numberLanesClosedIcon: require('@/assets/images/station/numberLane.png'),
      trafficStationIcon: require('@/assets/images/station/totalTrafficStation.png'),
      weighingStationImg: require('@/assets/images/station/station-weighing.png'),
      contentVmsDefault: require('@/assets/images/vms/120.png'),
      vehicleDefault: require('@/assets/images/default_img_cam.png'),
      activeNames: [],
      milepostList: [],
      listStageId: [],
      devicesList: [],
      listSiteStage: [],
      overlayDirections: [],
      listSiteStatus: [],
      districtList: DistrictList.data,
      typeAccidents: [
        { label: 'Tai nạn', value: 1 },
        { label: 'Hỏng xe', value: 2 }
      ],
      priorities: [
        { label: 'Thấp', value: 1 },
        { label: 'Trung bình', value: 2 },
        { label: 'Cao', value: 3 }
      ],
      keyword: '',
      stageName: '',
      stageDirection: '',
      checkFilterEvent: [1, 8, 4],
      listDeviceFilter: [1, 2, 3, 4, 5, 6, 7, 8],
      showDrawer: false,
      typeItem: 0,
      itemInfo: {},
      showPopup: false,
      showPopupAccident: false,
      showPopupTrafficJam: false,
      showPopupViolent: false,
      showPopupVMS: false,
      flagHideVisible: false,
      iconLayer: null,
      googleMapLayer: null,
      polygonLayer: null,
      siteLayer: null,
      districtLayer: null,
      entranceExitLayer: null,
      confirmAccident: false,
      confirmViolent: false,
      confirmFireEvent: false,
      confirmPatrolman: false,
      contentVMS: 1,
      contentVMSshow: 0,
      typeDisplay: 0,
      activeLink: null,
      colorDefault: '#63d668',
      eventData: null,
      eventLayer: null,
      eventsData: [],
      deviceDetail: {
        name: ''
      },
      typeAccident: 2,
      eventIconShow: true,
      showStationVideo: false,
      jobsData: [],
      jobLayer: null,
      detailJob: null,
      groupEventLayer: null,
      groupCameraLayer: null,
      rangeGroup: 800,
      overlayAccident: null,
      overlayViolent: null,
      overlayFireEvent: null,
      overlayTrafficJam: null,
      cameraLink: '',
      baseUrlImg: '',
      filterIcon: null,
      listStations: [],
      listStationsId: [],
      overLayStation: null,
      priority: null,
      groupEvents: [],
      laneList: [],
      listNearbyCam: [],
      mapMidList: [],
      isAdmin: false,
      loadingVMSdisplay: false,
      activeMediaStation: null,
      loadingFlagMap: true
    }
  },
  watch: {
    overLayStation() {
      const self = this
      setTimeout(function() {
        self.removeOverlayStation()
      }, 2500)
    },
    '$store.state.app.eventConfirm': function(event) {
      if (event) {
        this.hiddenPopup()
      }
    }
  },
  beforeCreate() {
    this.$store.dispatch('app/closeSideBar', 1)
  },
  created() {
    this.isAdmin = this.$root.checkAdmin(this.$route.fullPath)
    this.getLane()
  },
  async mounted() {
    this.init()
    this.getMapMid()
    await this.renderMap()
    setTimeout(() => {
      this.loadingFlagMap = false
    }, 500)
    await this.getListStages()
    await this.getListDevice()
    const mapEvents = JSON.parse(localStorage.getItem('mapEvents'))
    const mapJobs = JSON.parse(localStorage.getItem('mapJobs'))
    if (Array.isArray(mapEvents)) {
      this.eventsData = mapEvents
    }
    if (Array.isArray(mapJobs)) {
      this.jobsData = mapJobs
    }
  },
  methods: {
    init() {
      this.activeLink = this.vehicleDefault
      localStorage.setItem('mapEvents', JSON.stringify([]))
      localStorage.setItem('mapJobs', JSON.stringify([]))
    },
    getMapMid() {
      positionCenterResource
        .list()
        .then((res) => {
          if (res.status == 200 && res.data) {
            this.mapMidList = res.data
          }
        })
        .catch((err) => console.log(err))
    },
    async renderMap() {
      await this.delay(300)
      const districtLabel = this.createLabelLayer()
      this.districtLayer = this.createLabelLayer()
      this.entranceExitLayer = this.createEntranceLayer()
      this.googleMapLayer = this.createGoogleMapLayer()
      this.mapInstance = await this.initMap(this.googleMapLayer, districtLabel)
      this.mapInstance = await this.initMap(
        this.googleMapLayer,
        this.districtLayer,
        this.entranceExitLayer
      )
      this.popup = new Overlay({
        element: document.getElementById('popup-device'),
        autoPan: true,
        offset: [0, 0],
        autoPanAnimation: {
          duration: 200
        }
      })
      this.popupGroupEvents = new Overlay({
        element: document.getElementById('popup-events'),
        autoPan: true,
        offset: [0, 0],
        autoPanAnimation: {
          duration: 200
        }
      })
      this.stageNameOverlay = new Overlay({
        element: document.getElementById('stage-name'),
        offset: [0, 0],
        positioning: 'bottom-center'
      })
      this.mapInstance.addOverlay(this.popup)
      this.mapInstance.addOverlay(this.popupGroupEvents)
      this.mapInstance.addOverlay(this.stageNameOverlay)
      this.setTriggerMap()
    },
    initMap(...layer) {
      document.getElementById('map').innerHTML = ''
      document.getElementById('map').style.height = '100%'
      document.getElementById('map').style.width = '100%'
      const extent = process.env.VUE_APP_MAP_EXTENT.split(',')
      const map = new Map({
        target: 'map',
        controls: defaultControls({
          rotate: false,
          attribution: false
        }),
        interactions: defaultInteractions().extend([
          new DragRotateAndZoom()
        ]),
        layers: [...layer],
        view: new View({
          center: fromLonLat([
            this.centerLongLat.longitude,
            this.centerLongLat.latitude
          ]),
          zoom: this.zoom,
          minZoom: this.minZoom,
          maxZoom: this.maxZoom,
          extent: transformExtent(extent, 'EPSG:4326', 'EPSG:3857')
        })
      })
      return map
    },
    setTriggerMap() {
      this.mapInstance.on('pointermove', this.handlePointerMove)
      this.mapInstance.on('click', this.handleClickIcon)
      this.mapInstance.on('moveend', this.handleMoveEnd)
      let arrCoordinate = {}
      this.mapInstance.on('singleclick', (evt) => {
        const coordinate = toLonLat(evt.coordinate)
        arrCoordinate = {
          longitude: coordinate[0],
          latitude: coordinate[1]
        }
        console.log('coordinate: ' + JSON.stringify(arrCoordinate))
      })
    },
    createPolygonLayer() {
      const styles = [
        new Style({
          stroke: new Stroke({
            color: '#127D00',
            width: 4
          })
        })
      ]
      if (this.milepostList.length > 0) {
        const stageTop = []
        const stageBottom = []
        this.milepostList.forEach((distance) => {
          let top = []
          let bottom = []
          let mid = []
          distance.listSite.map((val, i) => {
            top.push(val.top)
            bottom.push(val.bottom)
            mid.push(val.mid)
            if (i == 0) {
              top.unshift(val.mid)
              bottom.unshift(val.mid)
            }
          })
          mid = mid.reverse()
          top = top.concat(mid)
          bottom = bottom.concat(mid)
          stageTop.push(
            new Feature({
              geometry: new LineString(top),
              name: 'stageFeature',
              id: distance.stageTop.id,
              data: distance.stageTop
            })
          )
          stageBottom.push(
            new Feature({
              geometry: new LineString(bottom),
              name: 'stageFeature',
              id: distance.stageBottom.id,
              data: distance.stageBottom
            })
          )
        })
        return new VectorLayer({
          source: new VectorSource({
            features: [...stageTop, ...stageBottom]
          }),
          id: 'lineStage',
          style: styles,
          zIndex: 10
        })
      }
    },
    createSiteLayer() {
      const styles = [
        new Style({
          stroke: new Stroke({
            color: this.colorDefault,
            width: 1
          }),
          fill: new Fill({
            color: this.colorDefault
          })
        })
      ]
      if (this.listSiteStage.length > 0) {
        const siteTop = []
        const siteBottom = []
        this.listSiteStage.forEach((stage) => {
          for (let i = 0; i < stage.listSite.length; i++) {
            const top = []
            const bottom = []
            const mid = []
            if (i + 1 < stage.listSite.length) {
              const siteStart = stage.listSite[i]
              const siteEnd = stage.listSite[i + 1]
              mid.push(fromLonLat([siteEnd.longitude, siteEnd.latitude]))
              mid.push(fromLonLat([siteStart.longitude, siteStart.latitude]))
              top.push(
                fromLonLat([siteStart.longitudeTop, siteStart.latitudeTop])
              )
              top.push(fromLonLat([siteEnd.longitudeTop, siteEnd.latitudeTop]))
              bottom.push(
                fromLonLat([
                  siteStart.longitudeBottom,
                  siteStart.latitudeBottom
                ])
              )
              bottom.push(
                fromLonLat([siteEnd.longitudeBottom, siteEnd.latitudeBottom])
              )
              siteTop.push(
                new Feature({
                  geometry: new MultiPolygon([[top.concat(mid)]]),
                  name: 'siteFeature',
                  id: siteEnd.siteId + '-top',
                  data: stage.stageTop
                })
              )
              siteBottom.push(
                new Feature({
                  geometry: new MultiPolygon([[bottom.concat(mid)]]),
                  name: 'siteFeature',
                  id: siteStart.siteId + '-bottom',
                  data: stage.stageBottom
                })
              )
            }
          }
        })
        return new VectorLayer({
          source: new VectorSource({
            features: [...siteTop, ...siteBottom]
          }),
          id: 'sitePolygon',
          style: styles,
          zIndex: 1
        })
      }
    },
    createIconLayer(icons, event = '') {
      const vectorSource = new VectorSource({
        features: []
      })
      vectorSource.clear()
      if (icons.length > 0) {
        icons.forEach((icon) => {
          if (this.isValidLngLat(icon.longitude, icon.latitude)) {
            const newFeature = new Feature({
              geometry: new Point(fromLonLat([icon.longitude, icon.latitude])),
              id: 'IconLayer',
              name: icon.id,
              status: icon.status ?? '',
              type: icon.type,
              data: icon
            })
            let scale = 0.7
            switch (icon.type) {
              case 2:
                scale = 0.7
                break
              case 4:
                scale = 0.65
                break
              case 6:
                scale = 0.65
                break
            }
            const iconStyle = new Style({
              image: new Icon({
                anchor: [0.5, 0.5],
                src: this.getIconCam(icon),
                scale: scale
              })
            })
            newFeature.setStyle(iconStyle)
            vectorSource.addFeature(newFeature)
          }
        })
      }
      return new VectorLayer({
        source: vectorSource,
        zIndex: 100,
        id: 'IconLayer'
      })
    },
    createGoogleMapLayer() {
      return new TileLayer({
        source: new TileWMS({
          url: process.env.VUE_APP_MAP_OFFLINE_URL,
          params: {
            LAYERS: process.env.VUE_APP_MAP_OFFLINE_LAYER,
            TILED: true,
            SERVICE: 'WMS',
            VERSION: '1.1.0',
            REQUEST: 'GetMap',
            WIDTH: '768',
            HEIGHT: '768',
            SRS: 'EPSG:4326',
            STYLES: ''
          },
          projection: 'EPSG:4326',
          serverType: 'geoserver',
          transition: 0
        })
      })
    },
    createLabelLayer() {
      const vectorSource = new VectorSource({
        features: []
      })
      this.districtList.forEach((value) => {
        const pointDistrict = new Feature({
          geometry: new Point(fromLonLat([value.longitude, value.latitude])),
          name: value.name,
          id: 'district'
        })
        const stylePoint = new Style({
          text: new Text({
            font: 'bold 16px Roboto, sans-serif',
            fill: new Fill({ color: '#000' }),
            stroke: new Stroke({
              color: '#fff',
              width: 1
            }),
            textAlign: 'center',
            textBaseline: 'middle',
            overflow: true,
            text: value.name.toLocaleUpperCase()
          })
        })
        pointDistrict.setStyle(stylePoint)
        vectorSource.addFeature(pointDistrict)
      })
      return new VectorLayer({
        source: vectorSource,
        id: 'DistrictLabel',
        zIndex: 1
      })
    },
    createEntranceLayer() {
      const vectorSource = new VectorSource({
        features: []
      })
      EntranceExitList.forEach((value) => {
        const pointDistrict = new Feature({
          geometry: new Point(fromLonLat([value.longitude, value.latitude])),
          id: 'entranceExit'
        })
        const icon = require('@/assets/images/entranceExit/' + value.img)
        const stylePoint = new Style({
          image: new Icon({
            anchor: [0.5, 0.5],
            src: icon,
            scale: value.scale ?? 1
          })
        })
        pointDistrict.setStyle(stylePoint)
        vectorSource.addFeature(pointDistrict)
      })
      return new VectorLayer({
        source: vectorSource,
        id: 'entranceExit',
        zIndex: 0
      })
    },
    createEventLayer(events) {
      const vectorSource = new VectorSource({
        features: []
      })
      vectorSource.clear()
      if (events.length > 0) {
        events.forEach((event) => {
          if (this.isValidLngLat(event.longitude, event.latitude)) {
            const iconFeature = new Feature({
              geometry: new Point(
                fromLonLat([event.longitude, event.latitude])
              ),
              id: 'EventLayer',
              name: event.id,
              data: event
            })
            const speedLimit = event.speedLimit ? event.speedLimit : 0
            const srcIcon = event.eventCode
              ? this.getIconEvent(event.eventCode, speedLimit)
              : nothingIcon
            const iconStyle = new Style({
              image: new Icon({
                anchor: [0.5, 0.5],
                src: srcIcon,
                scale: 0.7
              })
            })
            iconFeature.setStyle(iconStyle)
            vectorSource.addFeature(iconFeature)
          }
        })
      }
      return new VectorLayer({
        source: vectorSource,
        zIndex: 500,
        id: 'EventLayer'
      })
    },
    createReplaceLayer(data, icon = null) {
      const vectorSource = new VectorSource({
        features: []
      })
      vectorSource.clear()
      if (data.length > 0) {
        data.forEach((event) => {
          if (this.isValidLngLat(event.point.longitude, event.point.latitude)) {
            const iconFeature = new Feature({
              geometry: new Point(
                fromLonLat([event.point.longitude, event.point.latitude])
              ),
              id: 'ReplaceLayer',
              data: event.nearby,
              point: event.point
            })
            const numbGroup = event.nearby ? event.nearby.length.toString() : ''
            let iconSrc = icon || replaceIcon
            let offsetY = 1
            let scale = 0.8
            let font = '15px sans-serif'
            if (this.listDeviceFilter.includes(event.point.type)) {
              offsetY = 8
              scale = 0.75
              font = '12px sans-serif'
            }
            if (event.nearby.find((val) => val.eventStatus == 'NOT_SEEN')) {
              iconSrc = warningIcon
            } else if (event.nearby.find((val) => val.cameraStatus == 'STOP')) {
              iconSrc = replaceCamOffIcon
            }
            iconFeature.setStyle(
              new Style({
                image: new Icon({
                  anchor: [0.5, 0.5],
                  src: iconSrc,
                  scale
                }),
                text: new Text({
                  font,
                  offsetY,
                  textAlign: 'center',
                  justify: 'center',
                  text: numbGroup,
                  fill: new Fill({
                    color: '#FFFFFF'
                  }),
                  stroke: new Stroke({
                    color: '#000000',
                    width: 1.5
                  }),
                  padding: [2, 2, 2, 2]
                })
              })
            )
            vectorSource.addFeature(iconFeature)
          }
        })
      }
      return new VectorLayer({
        source: vectorSource,
        zIndex: 501,
        id: 'ReplaceLayer'
      })
    },
    distanceBetweenPoints(latlng1, latlng2) {
      const R = 6371e3
      const φ1 = (latlng1[1] * Math.PI) / 180
      const φ2 = (latlng2[1] * Math.PI) / 180
      const Δφ = ((latlng2[1] - latlng1[1]) * Math.PI) / 180
      const Δλ = ((latlng2[0] - latlng1[0]) * Math.PI) / 180
      const a =
        Math.sin(Δφ / 2) * Math.sin(Δφ / 2) +
        Math.cos(φ1) * Math.cos(φ2) * Math.sin(Δλ / 2) * Math.sin(Δλ / 2)
      const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a))
      return R * c
    },
    resetCameraIcon() {
      let listPoint = this.iconLayer.getSource().getFeatures()
      listPoint = listPoint.filter((val) => val.get('type') !== 4)
      if (listPoint.length < 1) {
        return
      }
      const self = this
      const temp = []
      const checked = new Set()
      for (let i = 0; i < listPoint.length; i++) {
        if (checked.has(i)) continue
        const point = listPoint[i]
        const inRange = []
        for (let j = i + 1; j < listPoint.length; j++) {
          if (checked.has(j)) continue
          const insidePoint = listPoint[j]
          const lonlat1 = toLonLat(insidePoint.getGeometry().getCoordinates())
          const lonlat2 = toLonLat(point.getGeometry().getCoordinates())
          if (this.distanceBetweenPoints(lonlat1, lonlat2) < this.rangeGroup) {
            checked.add(i)
            checked.add(j)
            inRange.push(insidePoint)
          }
        }
        if (inRange.length > 0) {
          inRange.push(point)
          inRange.forEach((inside) => {
            self.iconLayer.getSource().removeFeature(inside)
          })
          temp.push({
            point: point.values_.data,
            nearby: inRange.map((val) => {
              return val.values_.data
            })
          })
        }
      }
      if (this.groupCameraLayer) {
        this.mapInstance.removeLayer(this.groupCameraLayer)
      }
      this.groupCameraLayer = this.createReplaceLayer(temp, replaceCamOnIcon)
      this.mapInstance.addLayer(this.groupCameraLayer)
    },
    resetIconLayer(type = '') {
      let listPoint = []
      if (this.eventsData && this.jobsData) {
        const sourceEvent = this.eventsData
        const sourceJob = this.jobsData
        listPoint = _.concat(sourceEvent, sourceJob)
      }
      if (listPoint.length < 1) {
        return
      }
      const result = []
      const checked = new Set()
      for (let i = 0; i < listPoint.length; i++) {
        const point = listPoint[i]
        if (checked.has(i)) continue
        const inRange = []
        for (let j = i + 1; j < listPoint.length; j++) {
          const insidePoint = listPoint[j]
          if (checked.has(j)) continue
          const lonlat1 = [insidePoint.latitude, insidePoint.longitude]
          const lonlat2 = [point.latitude, point.longitude]
          if (this.distanceBetweenPoints(lonlat1, lonlat2) < this.rangeGroup) {
            checked.add(i)
            checked.add(j)
            inRange.push(insidePoint)
          }
        }
        if (inRange.length > 0) {
          inRange.push(point)
          if (!type) {
            result.push({
              point: point,
              nearby: inRange
            })
          }
        } else {
          if (type == 'event' && 'jobId' in point == false) {
            result.push(point)
          } else if (type == 'job' && 'jobId' in point == true) {
            result.push(point)
          }
        }
      }
      return result
    },
    getIconEvent(eventCode, speed = 0, eventStatus = '') {
      if (eventStatus == 'NOT_SEEN') {
        return warningIcon
      }
      let srcIcon = replaceIcon
      switch (eventCode) {
        case 'ACCIDENT':
          srcIcon = tainanIcon
          break
        case 'BROKENVEHICLE':
          srcIcon = xehongIcon
          break
        case 'BARRIER':
          srcIcon = vatcanIcon
          break
        case 'FIRE':
          srcIcon = fireIcon
          break
        case 'RAIN':
          srcIcon = rainIcon
          break
        case 'LANDSLIDE':
          srcIcon = landslideIcon
          break
        case 'SNOW':
          srcIcon = snowIcon
          break
        case 'FOG':
          srcIcon = fogIcon
          break
        case 'CONSTRUCTION':
          srcIcon = contructionIcon
          break
        case 'CROWD':
          srcIcon = crowdIcon
          break
        case 'HIGH_TEMPERATURE':
          srcIcon = nhietdotangIcon
          break
        case 'DAMAGED_ROAD':
          srcIcon = duonghongIcon
          break
        case 'DESTRUCTION':
          srcIcon = phahoaiIcon
          break
        case 'CROWDING':
          srcIcon = xehongIcon
          break
        case 'BLOCKLANE':
          srcIcon = blockLaneIcon
          break
        case 'CLOSE_LANE':
          srcIcon = blockLaneIcon
          break
        case 'BLOCKROAD':
          srcIcon = roadBlockIcon
          break
        case 'FORBIDDEN_WAY':
          srcIcon = roadBlockIcon
          break
        case 'LIMIT_SPEED':
          srcIcon = this.getJobSpeedIcon(speed)
          break
        case 'CLOSE_OPEN_ENTRANCE_EXIT':
          srcIcon = dongloiravaoBlockIcon
          break
      }
      return srcIcon
    },
    getIconCam(icon) {
      let srcIcon = bienBaoVmsIcon
      switch (icon.type) {
        case 1:
          if (icon.statusDevice == 'not') {
            srcIcon = cameraQuanSatNotIcon
          } else if (icon.statusDevice == 'offline') {
            srcIcon = cameraQuanSatOfflineIcon
          } else {
            srcIcon = cameraQuanSatIcon
          }
          break
        case 2:
          if (icon.statusDevice == 'not') {
            srcIcon = cameraGiamSatNotIcon
          } else if (icon.statusDevice == 'offline') {
            srcIcon = cameraGiamSatOfflineIcon
          } else {
            srcIcon = cameraGiamSatIcon
          }
          break
        case 4:
          if (icon.statusDevice == 'not') {
            srcIcon = bienBaoVmsNotIcon
          } else if (icon.statusDevice == 'offline') {
            srcIcon = bienBaoVmsOfflineIcon
          } else {
            srcIcon = bienBaoVmsIcon
          }
          break
        case 6:
          srcIcon = tramThuPhiIcon
          break
        case 7:
          srcIcon = cabinetIcon
          break
        case 8:
          if (icon.statusDevice == 'not') {
            srcIcon = cameraDoXeNotIcon
          } else if (icon.statusDevice == 'offline') {
            srcIcon = cameraDoXeOfflineIcon
          } else {
            srcIcon = cameraDoXeIcon
          }
          break
      }
      return srcIcon
    },
    getListStages() {
      listStageResource
        .list()
        .then((res) => {
          if (res.status == 200) {
            const { data } = res
            this.listSiteStage = data
            const statusSite = []
            if (data.length > 0) {
              this.milepostList = data.map((stage) => {
                stage.listSite.forEach((site, i) => {
                  if (i + 1 < stage.listSite.length) {
                    statusSite.push({
                      siteId: stage.listSite[i + 1].siteId + '-top',
                      statusCode:
                        stage.listSite[i + 1].trafficStatusTop?.statusCode ??
                        'CLEAR'
                    })
                  }
                  statusSite.push({
                    siteId: site.siteId + '-bottom',
                    statusCode: site.trafficStatusBottom?.statusCode ?? 'CLEAR'
                  })
                })
                return {
                  stageName:
                    stage.stageTop.processName ?? stage.stageBottom.processName,
                  stageTop: stage.stageTop,
                  stageBottom: stage.stageBottom,
                  listSite: stage.listSite.map((site) => {
                    return {
                      mid: fromLonLat([site.longitude, site.latitude]),
                      top: fromLonLat([site.longitudeTop, site.latitudeTop]),
                      bottom: fromLonLat([
                        site.longitudeBottom,
                        site.latitudeBottom
                      ])
                    }
                  })
                }
              })
              this.polygonLayer = this.createPolygonLayer()
              this.mapInstance.addLayer(this.polygonLayer)
              const featuresPolygon = this.polygonLayer
                .getSource()
                .getFeatures()
              featuresPolygon.forEach((feature) => {
                const stageDirection = this.createOverLayStageDirection({
                  longitude: getCenter(feature.getGeometry().getExtent())[0],
                  latitude: getCenter(feature.getGeometry().getExtent())[1],
                  ...feature.get('data')
                })
                this.overlayDirections.push(stageDirection)
                this.mapInstance.addOverlay(stageDirection)
              })
              this.siteLayer = this.createSiteLayer()
              this.mapInstance.addLayer(this.siteLayer)
              this.listSiteStatus = statusSite
              statusSite.forEach((value) => {
                this.setSiteStatus(value)
              })
            }
          }
        })
        .catch((err) => {
          console.error(err)
        })
    },
    getListDevice() {
      listDeviceStageResource
        .list()
        .then((response) => {
          if (response.status == 200) {
            const { data } = response
            const self = this
            const listCameras = data.listCameras.map((val) => {
              return {
                ...val,
                statusDevice: self.getCameraStatus(val.cameraStatus),
                type: self.getCameraType(val.cameraType)
              }
            })
            let listVmsBoards = []
            const listTollStation = []
            if (this.typePage === 'vms') {
              listVmsBoards = data.listVmsBoards.map((val) => {
                return {
                  ...val,
                  statusDevice: self.getCameraStatus(val.boardStatus),
                  type: 4
                }
              })
            }
            this.devicesList = _.concat(
              listCameras,
              listVmsBoards,
              listTollStation
            )
            this.iconLayer = this.createIconLayer(this.devicesList)
            this.mapInstance.addLayer(this.iconLayer)
            this.resetCameraIcon()
          }
        })
        .catch((error) => {
          console.error(error)
        })
    },
    getCameraType(camType, res = 'CODE') {
      if (res == 'CODE') {
        switch (camType) {
          case 'SURVEILLANCE':
            return 2
          case 'PANORAMIC':
            return 1
          case 'LICENSEPLATE':
            return 8
          default:
            return 1
        }
      }
      switch (camType) {
        case 'SURVEILLANCE':
          return 'Camera giám sát'
        case 'PANORAMIC':
          return 'Camera quan sát'
        case 'LICENSEPLATE':
          return 'Camera dò xe'
        default:
          return 'Camera giám sát'
      }
    },
    getCameraStatus(status) {
      if (status == 'RUNNING' || status == 2 || status == 'ENABLE') {
        return 'online'
      } else {
        return 'not'
      }
    },
    closePopup() {
      this.widthPopup = '900px'
    },
    hiddenPopup() {
      if (this.popupGroupEvents) {
        this.popupGroupEvents.setPosition(undefined)
        const tooltip = document.querySelectorAll('[id^=el-tooltip]')
        if (tooltip.length > 0) {
          tooltip.forEach((val) => val.remove())
        }
      }
      this.showPopup = false
    },
    hiddenOverlayPopup() {
      if (this.popup) {
        this.popup.setPosition(undefined)
      }
      if (this.stageNameOverlay) {
        this.stageNameOverlay.setPosition(undefined)
      }
    },
    isValidLngLat(lng, lat) {
      if (lng < -180 || lng > 180) {
        return false
      } else if (lat < -90 || lat > 90) {
        return false
      }
      return true
    },
    createOverLay(data, show = true) {
      const element = document.createElement('div')
      let className = 'pulsate'
      if (!show) {
        className += ' hide'
      }
      element.setAttribute('class', className)
      element.setAttribute('id', data.id)
      element.addEventListener('click', (e) => {
        this.eventClick(data)
      })
      const coordinate = fromLonLat([data.longitude, data.latitude])
      return new Overlay({
        id: data.id,
        element: element,
        position: coordinate,
        autoPan: false,
        offset: [0, 0],
        zIndex: 999
      })
    },
    createOverLayStation(data) {
      const element = document.createElement('div')
      element.setAttribute('class', 'overlay-station')
      element.setAttribute('id', data.id)
      element.addEventListener('click', (e) => {
        const itemInfo = this.devicesList.find((x) => x.id == data.id)
        this.showPopup = true
        this.itemInfo = itemInfo
        this.typeItem = itemInfo.type
      })
      const coordinate = fromLonLat([data.longitude, data.latitude])
      return new Overlay({
        id: data.id,
        element: element,
        position: coordinate,
        autoPan: true,
        offset: [0, 0],
        autoPanAnimation: {
          duration: 250
        },
        zIndex: 999
      })
    },
    createOverLayStageDirection(data) {
      const element = document.createElement('div')
      element.setAttribute('class', 'overlay-direction')
      element.setAttribute('id', data.id)
      const offset = [-20, -15]
      const mapMid = this.mapMidList.filter(function(val) {
        return val.stageCode == data.stageCode
      })
      if (data.derectionCode == this.$directionSubMain.value) {
        element.innerHTML = '<img src="' + nbhnIcon + '" />'
        if (mapMid.length > 0) {
          data.longitude = mapMid[0].middleTopPoint.longitude
          data.latitude = mapMid[0].middleTopPoint.latitude
        }
      } else {
        element.innerHTML = '<img src="' + hnnbIcon + '" />'
        if (mapMid.length > 0) {
          data.longitude = mapMid[0].middleBottomPoint.longitude
          data.latitude = mapMid[0].middleBottomPoint.latitude
        }
      }
      return new Overlay({
        id: data.id,
        element: element,
        offset: offset,
        stopEvent: false,
        position: fromLonLat([data.longitude, data.latitude]),
        zIndex: 0
      })
    },
    handleMoveEnd(event) {
    },
    handlePointerMove(event) {
      const pixel = this.mapInstance.getEventPixel(event.originalEvent)
      const hit = this.mapInstance.hasFeatureAtPixel(pixel)
      const hover = this.mapInstance.forEachFeatureAtPixel(
        event.pixel,
        (feature) => feature
      )
      this.mapInstance.getTargetElement().style.cursor = ''
      const stageIdHover = null
      const ignorePointer = ['district', 'entranceExit']
      if (hover) {
        if (!ignorePointer.includes(hover.get('id')) && hit) {
          this.mapInstance.getTargetElement().style.cursor = 'pointer'
        }
        const typeGeometry = hover.getGeometry().getType()
        if (
          typeGeometry == 'MultiPolygon' &&
          hover.get('name') == 'siteFeature'
        ) {
          if (this.polygonLayer) {
            const data = hover.get('data')
            const featuresPolygon = this.polygonLayer.getSource().getFeatures()
            const stageHover = featuresPolygon.find((feature) => {
              return data.id == feature.get('id')
            })
            const centerStage = getCenter(stageHover.getGeometry().getExtent())
            this.stageName = data.processName
            this.stageNameOverlay.setPosition(centerStage)
          }
        } else if (typeGeometry == 'Point' && hover.get('id') == 'IconLayer') {
          const data = hover.get('data')
          const typeDivice = [1, 2, 3, 4, 5, 6, 7, 8, 14]
          if (
            typeDivice.includes(data.type) &&
            !this.popupGroupEvents.getPosition()
          ) {
            this.deviceDetail = data
            this.popup.setPosition(fromLonLat([data.longitude, data.latitude]))
          }
        } else {
          this.hiddenOverlayPopup()
        }
      } else {
        this.hiddenOverlayPopup()
      }
      if (!hit) {
        this.hiddenOverlayPopup()
        if (this.checkFilterEvent.includes(11)) {
          if (!stageIdHover) {
            this.changeColorId(this.colorDefault, this.listStageId)
          }
        } else {
          this.changeColorId(this.colorDefault, this.listStageId)
        }
      }
    },
    handleClickIcon(event) {
      const selectedIcon = this.mapInstance.forEachFeatureAtPixel(
        event.pixel,
        (feature) => feature
      )
      if (selectedIcon) {
        const coordinates = selectedIcon.getGeometry().getCoordinates()
        const typeSelectedFeature = selectedIcon.getGeometry().getType()
        const { values_: pointData } = selectedIcon
        if (typeSelectedFeature == 'Point') {
          if (selectedIcon.get('id') == 'ditrict') {
            return
          }
          const selectedId = selectedIcon.get('name')
          if (selectedIcon.get('id') == 'EventLayer') {
            this.eventClick(selectedIcon.get('data'))
          } else if (selectedIcon.get('id') == 'ReplaceLayer') {
            this.popupGroupEvents.setPosition(coordinates)
            this.groupEvents = selectedIcon.get('data')
          } else {
            this.hiddenPopup()
            const deviceType = selectedIcon.get('type')
            this.showDialogDevice(selectedId, deviceType)
          }
        } else if (typeSelectedFeature == 'MultiPolygon') {
          const { listStages } = this.$store.state.user
          const {
            id,
            stageCode,
            processName,
            derectionCode: directionCode
          } = selectedIcon.get('data')
          const isAccess = true
        } else {
          this.hiddenPopup()
        }
      } else {
        this.hiddenPopup()
      }
    },
    handleChangeFilter() {
      let showSecurityEvent = false
      if (this.checkFilterEvent.includes(20)) {
        showSecurityEvent = true
      }
      localStorage.setItem('alertSecurityEvent', showSecurityEvent)
      const filterDevices = this.devicesList.filter((item) => {
        return this.checkFilterEvent.includes(item.type)
      })
      let isDefaultStatus = false
      if (!this.checkFilterEvent.includes(11)) {
        isDefaultStatus = true
      }
      this.listSiteStatus.forEach((value) => {
        this.setSiteStatus(value, false, isDefaultStatus)
      })
      this.filterIcon = filterDevices
      if (this.iconLayer) {
        this.mapInstance.removeLayer(this.iconLayer)
      }
      this.iconLayer = this.createIconLayer(filterDevices)
      if (this.checkFilterEvent.includes(10)) {
        this.eventIconShow = true
        const mapEvents = JSON.parse(localStorage.getItem('mapEvents'))
        const mapJobs = JSON.parse(localStorage.getItem('mapJobs'))
        if (Array.isArray(mapEvents)) {
          this.eventsData = _.cloneDeep(mapEvents)
        }
        if (Array.isArray(mapJobs)) {
          this.jobsData = _.cloneDeep(mapJobs)
        }
      } else {
        this.eventIconShow = false
        this.mapInstance.getOverlays().clear()
        if (this.overlayDirections.length > 0) {
          this.overlayDirections.forEach((v) => this.mapInstance.addOverlay(v))
        }
        this.mapInstance.addOverlay(this.popup)
        this.mapInstance.addOverlay(this.popupGroupEvents)
      }
      this.changeEventIcon()
      this.changeJobIcon()
      this.mapInstance.addLayer(this.iconLayer)
      this.resetCameraIcon()
    },
    changeColorId(color = '#61F69D', id) {
      const newStyle = new Style({
        stroke: new Stroke({
          color,
          width: 1
        }),
        fill: new Fill({
          color
        })
      })
      const ids = Array.isArray(id) ? id : [id]
      if (this.siteLayer) {
        const featuresPolygon = this.siteLayer.getSource().getFeatures()
        featuresPolygon.forEach((feature) => {
          if (ids.includes(feature.get('id'))) {
            feature.setStyle(newStyle)
          }
        })
      }
    },
    delay(ms) {
      return new Promise((res) => setTimeout(res, ms))
    },
    async showDialogDevice(deviceId, deviceType) {
      if (deviceType) {
        switch (deviceType) {
          case 6:
            this.showPopupStation(deviceId)
            return
          case 7:
            this.showPopupCabinet(deviceId)
            return
        }
      }
      let itemInfo = null
      if (this.devicesList && this.devicesList.length > 0) {
        itemInfo = this.devicesList.find((x) => x.id == deviceId)
      } else {
        const resourceDevice = new ResourceNew('system/systemconfig/camera/' + deviceId)
        resourceDevice
          .list()
          .then((res) => {
            if (res.status == 200 || res.status == 201) {
              const { data } = res
              itemInfo = data
              itemInfo.type = this.getCameraType(data.cameraType)
            }
          })
          .catch((err) => {
            this.$message({
              message:
                err.message || 'Có lỗi xảy ra! showDialogDevice resourceDevice',
              type: 'warning'
            })
            console.error(err)
          })
      }
      if (itemInfo) {
        if (
          itemInfo.type == 1 ||
          itemInfo.type == 2 ||
          itemInfo.type == 3 ||
          itemInfo.type == 8
        ) {
          await new Resource('vds/camera/' + itemInfo.id)
            .list()
            .then((res) => {
              if (res.status == 200 || res.status == 201) {
                const { data } = res
                itemInfo.vdsId = data ? data.id : ''
              }
            })
            .catch((err) => {
              console.error(err)
            })
        } else if (itemInfo.type == 4) {
          await new Resource('vmsboard/displayscript/board')
            .get(itemInfo.id)
            .then((res) => {
              if (res.status == 200 || res.status == 201) {
                const { data } = res
                itemInfo.startTime = data.startTime
                itemInfo.endTime = data.endTime
                itemInfo.newsName = data.name
                itemInfo.preview = data.preview.split(',')
              }
            })
            .catch((err) => {
              console.error(err)
            })
        }
        this.itemInfo = _.cloneDeep(itemInfo)
        this.flagMileStone = Math.floor(Math.random() * 9999999)
        this.typeItem = itemInfo.type
        this.showPopup = true
      }
    },
    clickOther(eventData) {
      this.priority = null
      switch (eventData.type) {
        case 10:
          this.showDetailAccident(eventData)
          break
        case 11:
          this.showDetailTrafficJam(eventData)
          break
        case 12:
          this.showDetailViolent(eventData)
          break
        case 13:
          this.showDetailViolent(eventData)
          break
        default:
          this.showDetailTrafficJam(eventData)
          break
      }
    },
    async eventClick(event) {
      const eventData = {
        id: event.id,
        jobId: event.jobId,
        eventCode: event.eventCode,
        startTime: event.startTime,
        icon: event.icon,
        url: event.url ?? ''
      }
      if (config.eventCodeMapJob.includes(eventData.eventCode)) {
        this.eventData = eventData
        await this.getDetailJob(event.jobId)
        if (eventData.eventCode == 'LIMIT_SPEED') {
          this.widthPopup = '600px'
        } else {
          this.widthPopup = '350px'
        }
      } else {
        this.$store.dispatch('app/setShowNotify', {
          data: eventData,
          isShow: true
        })
      }
    },
    notifyEvent(eventData, iconSrc) {
      const h = this.$createElement
      this.$notify({
        position: 'top-right',
        dangerouslyUseHTMLString: true,
        customClass: 'noti-event',
        message: h('div', { class: 'noti-content' }, [
          h(
            'div',
            {
              class: 'row info',
              on: { click: this.clickOther.bind(this, eventData) }
            },
            [
              h('div', { class: 'col-4 noti-icon' }, [
                h('img', {
                  class: 'icon img-fluid',
                  attrs: { src: iconSrc }
                })
              ]),
              h('div', { class: 'col-8' }, [
                h('h5', { class: 'title' }, eventData.name),
                h(
                  'div',
                  { class: 'date-time' },
                  moment().format('DD/MM/YYYY HH:mm')
                ),
                h('div', { class: 'content' }, eventData.position)
              ])
            ]
          )
        ])
      })
    },
    redirectEvent(path, data, type = '') {
      localStorage.removeItem('newEvent')
      if (!path) {
        return
      }
      if (type == 'event') {
        localStorage.setItem('detailEventId', JSON.stringify(data.id))
        localStorage.setItem('showTab', 'info')
      } else if (type == 'vds') {
        localStorage.setItem('listNearbyCameras', JSON.stringify(data))
      } else if (type == 'newEvent') {
        localStorage.setItem('newEvent', JSON.stringify(data))
      }
      const route = this.$router.resolve({ path: path })
      window.open(route.href, '_blank')
    },
    handleAccident() {
      console.log('handleAccident')
      this.showPopupAccident = false
      this.confirmAccident = true
      this.iconLayer.getSource().clear()
      this.iconLayer = this.createIconLayer(this.getListIcon(), 'accident')
      this.mapInstance.addLayer(this.iconLayer)
      const overlaySelected = this.mapInstance.getOverlayById(this.eventData.id)
      this.mapInstance.removeOverlay(overlaySelected)
      this.redirectEvent('/inspect-event', 'accident')
    },
    handleViolent() {
      console.log('handleViolent')
      this.showPopupViolent = false
      this.confirmViolent = true
      this.iconLayer.getSource().clear()
      this.iconLayer = this.createIconLayer(this.getListIcon(), 'violent')
      this.mapInstance.addLayer(this.iconLayer)
      const overlaySelected = this.mapInstance.getOverlayById(this.eventData.id)
      this.mapInstance.removeOverlay(overlaySelected)
      this.redirectEvent('/inspect-event', 'violent')
    },
    getListIcon() {
      let listIcons = [...this.devicesList]
      if (this.confirmAccident) {
        listIcons = listIcons.concat(this.accidentData)
      }
      if (this.confirmViolent) {
        listIcons = listIcons.concat(this.violentData)
      }
      if (this.confirmFireEvent) {
        listIcons = listIcons.concat(this.fireEventData)
      }
      if (this.confirmPatrolman) {
        const patrol = this.getEventData(14)
        listIcons = listIcons.concat(patrol)
      }
      return listIcons
    },
    async showPopupStation(stationId) {
      const itemInfo = this.listStations.find((x) => x.id == stationId)
      const listMediaStation = [itemInfo]
      this.activeMediaStation = itemInfo
      await new ResourceNew('system/tollstation/camera')
        .list({ stationId })
        .then((res) => {
          if (res.status == 200 || res.status == 201) {
            const { data } = res
            if (data.length > 0) {
              data.forEach((v) => {
                v.type = this.getCameraType(v.cameraType)
                v.statusDevice = this.getCameraStatus(v.cameraStatus)
                listMediaStation.push(v)
              })
            }
          }
        })
        .catch((err) => {
          console.error(err)
        })
      const sumTotal = (arr) => arr.reduce((sum, { value }) => sum + value, 0)
      await new ResourceNew('system/tollstation/traffic')
        .list({
          tollStationId: stationId,
          levelFilterByTime: 'day',
          startDate: moment().format('YYYY-01-01 00:00:00'),
          endDate: moment().format('YYYY-MM-DD HH:mm:ss')
        })
        .then((res) => {
          if (res.status == 200 || res.status == 201) {
            const { data } = res
            itemInfo.totalTraffic = numberWithCommas(
              sumTotal(data.in) + sumTotal(data.out),
              '.'
            )
          }
        })
        .catch((err) => {
          console.error(err)
        })
      itemInfo.listMediaStation = listMediaStation
      this.itemInfo = itemInfo
      this.typeItem = 6
      this.showStationVideo = false
      this.showPopup = true
    },
    showPopupCabinet(id) {
      const itemInfo = this.devicesList.find((x) => x.id == id)
      console.log('itemInfo', itemInfo)
      this.itemInfo = itemInfo
      this.typeItem = 7
      this.showPopup = true
    },
    getJobSpeedIcon(speed, size = 'sm') {
      const speeds = Array(24)
        .fill(5)
        .map((v, i) => v * (i + 1))
      if (speed && speeds.includes(speed)) {
        if (size !== 'sm') {
          return require('@/assets/images/event/speed/speed' + speed + '.png')
        }
        return require('@/assets/images/event/jobs/speed' + speed + '.png')
      }
      return require('@/assets/images/event/jobs/speed0.png')
    },
    removeOverlayStation() {
      if (this.overLayStation) {
        this.mapInstance.removeOverlay(this.overLayStation)
      }
    },
    setSiteStatus(site, isNotify = false, isDefault = false) {
      return
    },
    setDeviceStatus(device) {
      if (this.iconLayer) {
        const self = this
        const filterStatusDevices = this.devicesList.map((val) => {
          if (val.id == device.deviceId) {
            val.statusDevice = self.getCameraStatus(device.status)
          }
          return val
        })
        this.devicesList = _.cloneDeep(filterStatusDevices)
        this.handleChangeFilter()
      }
    },
    changeEventIcon() {
      const hiddenEvent = [
        'PROCESSED',
        'INCORRECT',
        'UNKNOW'
      ]
      if (this.eventsData.length > 0) {
        const self = this
        const eventSeen = []
        this.resetIconLayer('event').forEach((event) => {
          if (event.eventStatus == 'NOT_SEEN' && self.eventIconShow) {
            self.mapInstance.addOverlay(self.createOverLay(event))
          } else if (hiddenEvent.includes(event.eventStatus)) {
            return
          } else {
            const overlay = self.mapInstance.getOverlayById(event.id)
            if (overlay) {
              self.mapInstance.removeOverlay(overlay)
            }
            eventSeen.push(event)
          }
        })
        if (this.eventLayer) {
          this.mapInstance.removeLayer(this.eventLayer)
        }
        if (this.groupEventLayer) {
          this.mapInstance.removeLayer(this.groupEventLayer)
        }
        if (this.eventIconShow) {
          this.eventLayer = this.createEventLayer(eventSeen)
          this.mapInstance.addLayer(this.eventLayer)
          this.groupEventLayer = this.createReplaceLayer(this.resetIconLayer())
          this.mapInstance.addLayer(this.groupEventLayer)
        }
      } else {
        if (this.eventLayer) {
          this.mapInstance.removeLayer(this.eventLayer)
        }
      }
    },
    async getEventsMap() {
      const resource = new ResourceNew('system/management/center/event/map')
      const param = {
        fromDate: moment().add(-7, 'days').format('YYYY-MM-DD HH:mm:ss'),
        toDate: moment().format('YYYY-MM-DD HH:mm:ss')
      }
      await resource
        .list(param)
        .then((res) => {
          if (res.status == 200) {
            const { data } = res
            if (data.length > 0) {
              const eventList = data
                .filter((val) => val.display == 1)
                .map((event) => {
                  return {
                    id: event.eventIdString,
                    name: event.eventName,
                    eventCode: event.eventCode,
                    eventStatus: event.eventStatus,
                    sourceId: event.sourceId,
                    sourceName: event.sourceName,
                    siteId: event.site.site_id,
                    siteName: event.site.site_name,
                    startTime: moment(event.startTime).format(
                      'YYYY-MM-DD HH:mm:ss'
                    ),
                    longitude: event.longitude,
                    latitude: event.latitude
                  }
                })
              localStorage.setItem('mapEvents', JSON.stringify(eventList))
            }
          }
        })
        .catch((err) => {
          console.error('getEventsMap', err)
        })
    },
    async getJobsMap() {
      const resource = new ResourceNew('system/management/common/job/map-notification')
      await resource
        .list()
        .then((res) => {
          if (res.status == 200 || res.status == 201) {
            const { data } = res
            if (data) {
              const listJobs = []
              data.map((val) => {
                const { job, startPoint, endPoint } = val
                if (startPoint) {
                  listJobs.push({
                    id: job.eventId,
                    jobId: job.id,
                    eventCode: job.jobType,
                    startTime: moment(job.eventStartTime).format(
                      'YYYY-MM-DD HH:mm:ss'
                    ),
                    longitude: startPoint.longitude,
                    latitude: startPoint.latitude,
                    speedLimit: job.limitSpeed ? job.limitSpeed : null
                  })
                }
                if (endPoint) {
                  listJobs.push({
                    id: job.eventId,
                    jobId: job.id,
                    eventCode: job.jobType,
                    startTime: moment(job.eventStartTime).format(
                      'YYYY-MM-DD HH:mm:ss'
                    ),
                    longitude: endPoint.longitude,
                    latitude: endPoint.latitude,
                    speedLimit: job.limitSpeed ? job.limitSpeed : null
                  })
                }
              })
              localStorage.setItem('mapJobs', JSON.stringify(listJobs))
            }
          }
        })
        .catch((err) => {
          console.error('getJobsMap', err)
        })
    },
    changeJobIcon() {
      if (this.jobsData.length > 0) {
        if (this.jobLayer) {
          this.mapInstance.removeLayer(this.jobLayer)
        }
        if (this.groupEventLayer) {
          this.mapInstance.removeLayer(this.groupEventLayer)
        }
        if (this.eventIconShow) {
          this.jobLayer = this.createEventLayer(this.resetIconLayer('job'))
          this.mapInstance.addLayer(this.jobLayer)
          this.groupEventLayer = this.createReplaceLayer(this.resetIconLayer())
          this.mapInstance.addLayer(this.groupEventLayer)
        }
      } else {
        if (this.jobLayer) {
          this.mapInstance.removeLayer(this.jobLayer)
        }
      }
    },
    getDetailJob(id) {
      if (id) {
        detailJobResource
          .get(id)
          .then((res) => {
            if (res.status == 200) {
              if (res.data) {
                this.detailJob = res.data
                this.showPopup = true
                this.typeItem = 19
                this.getNearbyCameras(res.data.sourceId)
              } else {
                this.$message.error('Không có thông tin công việc')
              }
            }
          })
          .catch((err) => {
            this.$message.error('Không có thông tin công việc')
            console.error(err)
          })
      }
    },
    async getLane() {
      await new ResourceNew('system/systemconfig/lanes?page=0&size=200')
        .list()
        .then((res) => {
          this.laneList = []
          if (res.status == 200) {
            if (res.data.content) {
              this.laneList = res.data.content
            }
          }
        })
        .catch((err) => console.error('getLane', err))
    },
    getJobName(code) {
      return config.listJobMap[code]
    },
    getNearbyCameras(camId) {
      if (camId) {
        const resource = new Resource('map/nearby-camera')
        const param = {
          cameraId: camId
        }
        resource
          .list(param)
          .then((res) => {
            if (res.status == 200 || res.status == 201) {
              this.listNearbyCam = res.data.filter(Boolean)
            }
          })
          .catch((err) => console.error(err))
      }
    },
    getLaneImage(status) {
      return getLaneImage(status)
    }
  }
}
</script>
<style lang="scss" scoped>
$widthPopupDevice: 250px;
$widthEvent: 235px;
$widthThumbVMS: 480px;
.loading-dasboard-traffic {
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  svg {
    animation: rotate 1.5s linear infinite;
    font-size: 40px;
  }
}
.svg-icon-loading {
  width: 1em;
  height: 1em;
  fill: currentColor;
  overflow: hidden;
  stroke: #7C7E81;
  stroke-width: 4;
  stroke-dasharray: 90, 188;
}
@mixin text-shadow {
  text-shadow: 1px 0 0 #ffffff, -1px 0 0 #ffffff, 0 1px 0 #ffffff,
    0 -1px 0 #ffffff, 1px 1px #ffffff, -1px -1px 0 #ffffff, 1px -1px 0 #ffffff,
    -1px 1px 0 #ffffff;
}
.filter-violation-map {
  width: 375px;
  ::v-deep .el-collapse-item__wrap {
    border-bottom: initial !important;
  }
  ::v-deep .el-collapse-item__header {
    height: 30px;
    font-size: 16px;
  }
}
.svg-icon-map {
  width: 22px;
  height: 22px;
}
.list-icon-note {
  max-height: 50vh;
  overflow-y: auto;
  padding: 5px 0;
  .last-item {
    width: 100% !important;
  }
  h6 {
    clear: both;
    font-size: 14px;
    border-bottom: 1px solid #ddd;
    margin-bottom: 5px;
    padding-bottom: 5px;
    padding-top: 5px;
  }
  .item {
    margin-bottom: 4px;
    width: 50%;
    float: left;
    display: flex;
    justify-content: flex-start;
    align-items: center;
    .img {
      float: left;
      margin-right: 10px;
    }
  }
  .color {
    height: 25px;
    width: 25px;
  }
}
.image-vmss {
  clear: both;
  padding-top: 10px;
  margin-bottom: 10px;
  img {
    width: 100%;
  }
}
.detail-info-vms {
  .item-icon {
    width: 36px;
    float: left;
  }
  .item {
    width: calc(100% - 36px) !important;
    padding-top: 0 !important;
  }
  .detail-btn {
    text-align: center;
  }
}
.isScreenfull #map {
  height: 100vh !important;
}
#map {
  .ol-popup-device {
    position: absolute;
    min-width: 150px;
    width: $widthPopupDevice;
    background-color: white;
    -webkit-filter: drop-shadow(0 1px 4px rgba(0, 0, 0, 0.2));
    filter: drop-shadow(0 1px 4px rgba(0, 0, 0, 0.2));
    border-radius: 6px;
    border: 1px solid #ccc;
    bottom: 25px;
    left: $widthPopupDevice/2 * -1;
    .ol-popup-events {
      max-height: 200px;
      overflow-y: auto;
      padding: 15px 15px 5px 10px;
      display: grid;
      grid-template-columns: repeat(5, max-content);
      justify-content: center;
      justify-items: center;
      div {
        padding: 0 3px;
      }
      img {
        max-width: 35px;
      }
    }
    @for $i from 1 through 5 {
      &.fit-content-#{$i} {
        max-width: fit-content;
        width: $widthEvent;
        min-width: unset;
        left: #{(40 * $i + 25) / 2 * -1}px;
      }
    }
  }
  .ol-popup-device:after,
  .ol-popup-device:before {
    top: 100%;
    border: solid transparent;
    content: '';
    height: 0;
    width: 0;
    position: absolute;
    pointer-events: none;
    left: $widthPopupDevice/2;
  }
  @for $i from 0 through 5 {
    .fit-content-#{$i}:after,
    .fit-content-#{$i}:before {
      left: #{(40 * $i + 25) / 2}px;
    }
  }
  .ol-popup-device:after {
    border-top-color: white;
    border-width: 10px;
    margin-left: -10px;
  }
  .ol-popup-device:before {
    border-top-color: #cccccc;
    border-width: 11px;
    margin-left: -11px;
  }
  .close-popup {
    position: absolute;
    right: 5px;
    top: 1px;
    cursor: pointer;
  }
}
#stage-name {
  position: absolute;
  min-width: 200px;
  width: fit-content;
  top: 0px;
  bottom: 0px;
  font-size: 16px;
  left: $widthPopupDevice/2 * -1;
  pointer-events: none;
  font-weight: 700;
  @include text-shadow;
}
.flag-hide-btn {
  .transform-map-0,
  .transform-map-1 {
    display: none;
  }
  .transform-map {
    display: none !important;
  }
  .transform-map-6 {
    display: block !important;
    right: 15px !important;
  }
}
.map-monitor {
  height: 100%;
  .map-title {
    color: #292663;
    font-family: Roboto;
    font-size: 22px;
    font-weight: 700;
    line-height: 42px;
    position: absolute;
    z-index: 99;
    left: 31%;
    top: 15px;
    pointer-events: none;
    @include text-shadow;
  }
  position: relative;
  .search-block {
    position: absolute;
    z-index: 99;
    top: 20px;
    left: 20px;
    width: 300px;
    .el-icon-search {
      color: white;
      font-size: 15px;
      position: relative;
    }
    input {
      height: 38px !important;
      line-height: 36px !important;
      border: 1px solid #403e74 !important;
    }
  }
}
.dropdown-custom {
  padding: 8px 10px !important;
  .el-checkbox-group {
    display: flex;
    flex-wrap: wrap;
    flex-direction: column;
    align-content: center;
    align-items: flex-start;
    justify-content: space-around;
  }
  .img-checkbox {
    float: left;
    width: 18px;
    img {
      width: 100%;
    }
  }
  .txt-checkbox {
    margin-left: 10px;
    color: #3d3d3d;
    font-size: 14px;
    line-height: 20px;
  }
}
.el-drawer {
  &__wrapper {
    overflow: unset !important;
    width: 100%;
    top: 70%;
  }
}
#btn-drawer {
  background: #292663;
  color: #ffffff;
  position: fixed;
  right: 0.36%;
  bottom: 1%;
  padding: 5px 8px;
  transition: 0.3s;
  &.active {
    bottom: 30%;
    i {
      transform: rotate(180deg);
    }
  }
}
.media-info {
  .cam-detail {
    height: 380px;
  }
  .list-img-cam {
    margin-top: 15px;
    display: flex;
    justify-content: flex-start;
    .img-cam {
      width: 31%;
      margin-left: 15px;
      height: 100px;
      &:nth-child(1) {
        margin-left: 0px;
      }
      .active {
        border: 2px solid #409eff;
      }
    }
  }
  .image-vms {
    border: 1px solid #707070;
    padding: 15px;
    display: flex;
    align-items: center;
    background-image: url('~@/assets/images/vms/vms_bg.png');
    border-radius: 4px;
    background-repeat: no-repeat;
    background-size: cover;
    background-position: center;
    position: relative;
    img {
      position: absolute;
      width: $widthThumbVMS !important;
      height: auto !important;
      top: 37%;
      right: calc(50% - #{$widthThumbVMS/2});
    }
    #btn-reset {
      position: absolute;
      top: 10px;
      right: 10px;
      background: transparent;
      border: none;
    }
    .vms-content {
      width: 100%;
      background-color: #ffffff;
      padding: 10px;
      display: flex;
      align-items: center;
      flex-direction: column;
    }
  }
}
#wrong-direction {
  position: fixed;
  top: 45px;
  z-index: 9999;
  background: transparent;
  border: 0px;
  color: #8b85ff26;
}
#event-fire {
  position: fixed;
  top: 0px;
  z-index: 9999;
  background: transparent;
  border: 0px;
  color: #8b85ff26;
}
#event-patrolman {
  position: fixed;
  top: 0px;
  left: 180px;
  z-index: 9999;
  background: transparent;
  border: 0px;
  color: #292663;
}
#event-accident {
  position: fixed;
  top: 45px;
  left: 180px;
  z-index: 9999;
  background: transparent;
  border: 0px;
  color: #292663;
}
#event-traffic-jam {
  position: fixed;
  top: 45px;
  left: 280px;
  z-index: 9999;
  background: transparent;
  border: 0px;
  color: #292663;
}
.sw-media {
  position: absolute;
  right: 5px;
  top: 5px;
  z-index: 99;
}
.item-carousel {
  position: relative;
  height: 100%;
  &__image {
    width: 100%;
    height: 100%;
    object-fit: contain;
  }
}
.full-screen-map {
  position: absolute;
  top: 15px;
  z-index: 999;
  right: 15px;
}
.station-lane {
  height: 430px;
}
.list-media-station {
  display: flex;
  justify-content: flex-start;
  align-items: center;
  .item-media {
    width: 74px;
    padding: 7px;
    border: 2px solid #00000000;
    border-radius: 5px;
    img {
      width: 100%;
    }
    &.active {
      border-color: #403e74;
    }
  }
}
</style>
<style lang="scss">
.map .ol-rotate {
  top: 40px;
}
.map .ol-zoom {
  top: auto;
  left: auto;
  right: 10px;
  bottom: 10px;
}
.map-monitor {
  .el-input-group__prepend {
    background-color: #403e74;
    padding: 0 13px;
  }
  .el-input__inner {
    border: 1px solid #dadae1;
  }
  .el-button--medium.is-circle {
    padding: 10px;
  }
  .el-dialog__title {
    font-size: 15px;
  }
  .el-dialog__body {
    width: 100%;
    padding: 15px;
    .el-col {
      border-radius: 4px;
      height: 100%;
      padding: 0 10px;
    }
    .detail-info {
      height: 430px;
      border: 1px solid #707070;
      position: relative;
      &>div.mb-1 {
        padding-left: 5px;
        min-height: 50px;
      }
      .detail-btn {
        margin-bottom: 10px;
        position: absolute;
        bottom: 10px;
        left: 0px;
        width: 100%;
        text-align: center;
        display: grid;
        justify-content: center;
        grid-template-columns: repeat(2, auto);
        row-gap: 15px;
        padding: 0 15px;
        &.new-event {
          button {
            width: 100px;
          }
        }
        button {
          width: 150px;
          &:nth-child(2n) {
            margin-left: 15px;
          }
          &:nth-child(2n-1) {
            margin-left: 0px;
          }
        }
        &-job {
          margin-top: 15px;
          display: grid;
          justify-content: center;
          grid-template-columns: repeat(2, auto);
        }
      }
      .item-icon {
        width: 36px;
        float: left;
        display: block;
        padding-top: 10px;
        text-align: center;
      }
      &-job {
        border: 0px;
        height: auto;
      }
      &-speed {
        max-height: 300px;
      }
    }
    .media-info {
      padding: 0px;
      .cam-detail .video-js {
        height: 380px;
      }
      #vds-cam.cam-detail {
        height: 430px;
        .video-js {
          height: 430px;
        }
      }
    }
    .fill {
      padding-left: 15px;
      overflow: hidden;
      &>img {
        width: 100%;
        height: 100%;
      }
    }
    span.item {
      width: 85%;
      padding: 5px 0px 0px 10px;
      display: -webkit-box;
      &__hidden {
        overflow: hidden;
        text-overflow: ellipsis;
        -webkit-line-clamp: 3;
        -webkit-box-orient: vertical;
      }
      span {
        word-break: break-word;
      }
      img {
        margin-bottom: 5px;
        vertical-align: baseline;
      }
      .title {
        font-weight: 700;
        color: #3d3d3d;
      }
    }
  }
  .filter-layers {
    position: absolute;
    z-index: 99;
    top: 15px;
    right: 75px;
    .el-icon-place {
      color: #ccd2e3;
      font-size: 25px;
      position: relative;
    }
  }
  .transform-map {
    position: absolute;
    z-index: 99;
    top: 15px;
    &>.el-icon-place {
      color: #ccd2e3;
      font-size: 25px;
      position: relative;
    }
  }
  @for $i from 0 through 6 {
    .transform-map-#{$i} {
      right: #{(45 * $i + (15 * $i) + 15)}px;
    }
  }
  .dialog-device {
    .speed-img {
      display: flex;
      align-items: center;
      border: 1px solid #000000;
      border-radius: 4px;
      padding: 25px;
      height: 300px;
    }
  }
}
.el-drawer {
  &__open .el-drawer.btt {
    height: 100% !important;
    .el-drawer__body {
      height: 280px;
      overflow: hidden;
    }
  }
}
#block-drawer {
  display: grid;
  width: 100%;
  height: 100%;
  grid-template-columns: repeat(7, 1fr);
  grid-auto-flow: row;
  .item-drawer {
    border: 1px solid #3f3b85;
  }
  .el-select {
    width: 100%;
    .el-input {
      input {
        border-radius: 0px;
        background: #292663 !important;
        color: #ffffff;
        border: none;
      }
    }
  }
}
.filter-violation {
  min-width: 300px;
  .el-checkbox {
    margin-right: 15px;
    width: 100%;
    display: flex;
    justify-content: space-between;
    flex-flow: row-reverse;
    &__label {
      font-weight: normal;
      display: flex;
      align-items: center;
    }
    &__input {
      display: flex;
      align-self: center;
    }
  }
}
.overlay-direction {
  pointer-events: none;
}
.pulsate {
  -webkit-animation: pulse 1s ease-out;
  -moz-animation: pulse 1s ease-out;
  animation: pulse 1s ease-out;
  -webkit-animation-iteration-count: infinite;
  -moz-animation-iteration-count: infinite;
  animation-iteration-count: infinite;
  background-image: url('../../assets/images/icons_map/warning.png');
  background-position: center;
  background-repeat: no-repeat;
  height: 32px;
  width: 32px;
  position: absolute;
  top: -16px;
  left: -16px;
  z-index: 10;
  opacity: 0;
  &.hide {
    background: #00000000;
    border-radius: 10px;
  }
}
.overlay-station {
  height: 30px;
  width: 30px;
  position: absolute;
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
  display: inline-block;
  &:before,
  &:after {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: #045cff;
    border-radius: 50%;
    border: 2px solid #005eb6;
    opacity: 0;
  }
  &:before {
    transform: scale(1);
    animation: blink 1s infinite linear;
  }
  &:after {
    animation: blink 1s 1.2s infinite linear;
  }
}
@keyframes blink {
  0% {
    opacity: 0;
    transform: scale(0.4);
  }
  33.33333% {
    opacity: 1;
    transform: scale(1);
  }
  100% {
    opacity: 0;
    background: rgba(#ffc400, 0.6);
    transform: scale(2);
    border-radius: 0;
  }
}
@keyframes pulse {
  0% {
    opacity: 0;
  }
  100% {
    opacity: 1;
  }
}
#container-map {
  height: 100%;
}
</style>
