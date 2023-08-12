<template>
  <div id="container"></div>
  <a-space direction="vertical">
    <a-button type="primary" class="listBtn" @click="showNearbyModal">
      <bars-outlined style="font-size: 18px;position: relative;top: -2px;" />
      <span class="listBtn-span">查看附近停车场</span>
    </a-button>
  </a-space>
  <a-space direction="vertical">
    <a-input-search class="search-input" placeholder="请输入停车场名称" enter-button @search="onSearch" />
  </a-space>
  <a-modal v-model:open="open_nearby" title="附近停车场" width="100%" wrap-class-name="full-modal" @ok="handleNearbyOk"
    :footer="null">
    <a-list class="demo-loadmore-list" :loading="initNearbyLoading" item-layout="horizontal"
      :data-source="dataSource_nearby" :pagination=false :locale="{ emptyText: '暂无数据' }">
      <template #renderItem="{ item }">
        <a-list-item>
          <template #actions>
            <a v-if="dataSource_nearby.length" @click="onPosition([item.lng, item.lat])">
              <a>查看停车场</a>
            </a>
          </template>
          <a-list-item-meta>
            <template #title>
              <div>{{ item.parking_name }}</div>
            </template>
            <template #description>
              <div>地址：{{ item.address }}</div>
              <div>距您{{ parseInt(item.distance) }}米</div>
            </template>
          </a-list-item-meta>
        </a-list-item>
      </template>
    </a-list>
  </a-modal>
  <a-modal v-model:open="open_search" title="搜索停车场" width="100%" wrap-class-name="full-modal" @ok="handleSearchOk"
    :footer="null">
    <a-list class="demo-loadmore-list" :loading="initSearchLoading" item-layout="horizontal"
      :data-source="dataSource_search" :pagination=false :locale="{ emptyText: '暂无数据' }">
      <template #renderItem="{ item }">
        <a-list-item>
          <template #actions>
            <a v-if="dataSource_search.length" @click="onSearchPosition([item.lng, item.lat])">
              <a>查看停车场</a>
            </a>
          </template>
          <a-list-item-meta>
            <template #title>
              <div>{{ item.parking_name }}</div>
            </template>
            <template #description>
              <div>地址：{{ item.address }}</div>
              <div>距您{{ parseInt(item.distance) }}米</div>
            </template>
          </a-list-item-meta>
        </a-list-item>
      </template>
    </a-list>
  </a-modal>
</template>

<script setup>
import AMapLoader from '@amap/amap-jsapi-loader';
import { shallowRef, onMounted, ref } from 'vue'
import {
  BarsOutlined,
} from '@ant-design/icons-vue';
import { message } from 'ant-design-vue';
const parking_lot_nearby = require('../Data/parking_lot.json')

const map = shallowRef(null)
let parking_nearby_info = ref([])
let parking_search_info = ref([])
let currentAMap = null
let currentPosition = ref('')
let searchValue = ref('')
const dataSource_nearby = ref([])
const dataSource_search = ref([])
const open_nearby = ref(false);
const open_search = ref(false);
const initNearbyLoading = ref(true);
const initSearchLoading = ref(true);
// const loading = ref(false);
// 模糊搜索
let filterMsg = ref([])

const showNearbyModal = () => {
  getParkingLotNearbyInfo()
  open_nearby.value = true;
};

const handleNearbyOk = () => {
  open_nearby.value = false;
};

const showSearchModal = () => {
  open_search.value = true;
};

const handleSearchOk = () => {
  open_search.value = false;
};

const initMap = () => {
  AMapLoader.load({
    key: "0bd30b96d926277e6ea7e74e5f8bf37f",             // 申请好的Web端开发者Key，首次调用 load 时必填
    version: "2.0",      // 指定要加载的 JSAPI 的版本，缺省时默认为 1.4.15
  }).then((AMap) => {
    currentAMap = AMap
    map.value = new AMap.Map("container", {  //设置地图容器id
      zoom: 14,
      viewMode: "2D",
      center: [116.1, 31.1]
    });

    AMap.plugin('AMap.ToolBar', function () {//异步加载插件
      var toolbar = new AMap.ToolBar({
        position: {
          top: '20px',
          left: '10px'
        }
      });
      map.value && map.value.addControl(toolbar);
    });

    AMap.plugin('AMap.Scale', function () {
      var scale = new AMap.Scale({
        position: {
          bottom: '20px',
          left: '10px'
        }
      });
      map.value && map.value.addControl(scale);
    });

    AMap.plugin('AMap.MapType', function () {
      var mapType = new AMap.MapType({
        position: {
          top: '20px',
          right: '10px'
        }
      });
      map.value && map.value.addControl(mapType);
    });

    position_self()

  }).catch(e => {
    console.log(e);
  })
}
// 页面挂载完成
onMounted(() => {
  initMap()
  initNearbyLoading.value = false;
})

// 自身定位
const position_self = () => {
  currentAMap.plugin('AMap.Geolocation', function () {//异步加载插件
    var geolocation = new currentAMap.Geolocation({
      position: {
        enableHighAccuracy: true, // 是否使用高精度定位，默认：true
        timeout: 10000, // 设置定位超时时间，默认：无穷大
        offset: [10, 20],  // 定位按钮的停靠位置的偏移量
        position: 'RB', //  定位按钮的排放位置,  RB表示右下
      },
      showCircle: false,
    });
    map.value && map.value.addControl(geolocation);
    geolocation.getCurrentPosition((status, result) => {
      if (status == 'complete') {
        onComplete(result)
      } else {
        onError(result)
      }
    });
  });
}

// 定位成功 画圆
const onComplete = (result) => {
  currentPosition.value = {
    lng: result.position.lng,
    lat: result.position.lat
  }
  var circle = new currentAMap.Circle({
    center: [currentPosition.value.lng, currentPosition.value.lat],
    radius: 500, //半径
    borderWeight: 3,
    fillColor: '#1791fc',
    zIndex: 50,
  })
  map.value && map.value.add(circle);
  // 缩放地图到合适的视野级别
  map.value.setFitView()
  getParkingLotNearbyInfo()
  const elA = document.querySelector('.amap-geolocation')
  elA?.addEventListener('click', () => {
    getParkingLotNearbyInfo()
  })
}

const onError = (err) => {
  // 定位出错
  console.log('定位出错', err)
}

// 获取停车场数据
const getParkingLotNearbyInfo = () => {
  map.value && map.value.clearMap();
  parking_nearby_info.value = parking_lot_nearby.parking_lot_info
  dataSource_nearby.value = []
  parking_nearby_info.value.forEach((item) => {
    let distance = currentAMap.GeometryUtil.distance(currentPosition.value, [item.lng, item.lat])
    // if (distance <= 500) {
    dataSource_nearby.value.push({
      parking_name: item.parking_name,
      address: item.address,
      lng: item.lng,
      lat: item.lat,
      distance,
    })
    const marker = new currentAMap.Marker({
      position: [item.lng, item.lat],
      title: item.parking_name
    });
    map.value && map.value.add(marker)
    marker.on('click', function (e) {
      openInfoWindow(e, item, distance)
    })
    // }
  })
  dataSource_nearby.value.sort(compare('distance', true))
}

// 搜索停车场数据
const getParkingLotSearchInfo = () => {
  if (!searchValue.value) {
    message.info('请输入关键词', [2])
    initSearchLoading.value = false;
    return
  }
  handleSearch()
  map.value && map.value.clearMap();
  dataSource_search.value = []
  parking_search_info.value.forEach((item) => {
    let distance = currentAMap.GeometryUtil.distance(currentPosition.value, [item.lng, item.lat])
    dataSource_search.value.push({
      parking_name: item.parking_name,
      address: item.address,
      lng: item.lng,
      lat: item.lat,
      distance,
    })
    const marker = new currentAMap.Marker({
      position: [item.lng, item.lat],
      title: item.parking_name
    });
    map.value && map.value.add(marker)
    marker.on('click', function (e) {
      openInfoWindow(e, item, distance)
    })
  })
  initSearchLoading.value = false;
  showSearchModal()
}

const handleSearch = () => {
  parking_search_info.value = []
  let queryStringArr = searchValue.value.split("");
  let str = "(.*?)";
  filterMsg.value = [];
  let regStr = str + queryStringArr.join(str) + str;
  let reg = RegExp(regStr, "i"); // 以mh为例生成的正则表达式为/(.*?)m(.*?)h(.*?)/i
  parking_nearby_info.value.forEach(item => {
    if (reg.test(item.parking_name || item.address)) {
      parking_search_info.value.push(item)
    }
  });
}

// 排序   desc=true: 正序
const compare = (property, desc) => {
  return function (a, b) {
    var value1 = a[property];
    var value2 = b[property];
    if (desc == true) {
      // 升序排列
      return value1 - value2;
    } else {
      // 降序排列
      return value2 - value1;
    }
  }
}

//构建自定义信息窗体
const createInfoWindow = (title, content) => {
  var info = document.createElement("div");
  info.className = "custom-info input-card content-window-card";
  //可以通过下面的方式修改自定义窗体的宽高
  info.style.width = "200px";
  // 定义顶部标题
  var top = document.createElement("div");
  var titleD = document.createElement("div");
  var closeX = document.createElement("img");
  top.className = "info-top";
  titleD.innerHTML = title;
  closeX.src = "https://webapi.amap.com/images/close2.gif";
  closeX.onclick = closeInfoWindow;

  top.appendChild(closeX);
  top.appendChild(titleD);
  info.appendChild(top);

  // 定义中部内容
  var middle = document.createElement("div");
  middle.className = "info-middle";
  middle.style.backgroundColor = 'white';
  middle.innerHTML = content;
  info.appendChild(middle);

  // 定义底部内容
  var bottom = document.createElement("div");
  bottom.className = "info-bottom";
  bottom.style.position = 'relative';
  bottom.style.top = '0px';
  bottom.style.margin = '0 auto';
  var sharp = document.createElement("img");
  sharp.src = "https://webapi.amap.com/images/sharp.png";
  bottom.appendChild(sharp);
  info.appendChild(bottom);
  return info;
}

//关闭信息窗体
const closeInfoWindow = () => {
  map.value.clearInfoWindow();
}

const openInfoWindow = (e, item, distance) => {
  console.log('item', item)
  // //实例化信息窗体
  var content = [];
  content.push(`
  <div style="font-size:15px;">${item.address}</div>
  <div style="font-size:15px;">距您${parseInt(distance)}米</div>
  <button class="btn-nav"><a class="a-nav" href='${navigation(item)}' style="font: 20px black;">到这去</a></button>`);
  var title = '<div style="font-size:20px;color:black;font-weight:bold;">' + item.parking_name + '</div>'

  // 创建 infoWindow 实例
  var infoWindow = new currentAMap.InfoWindow({
    isCustom: true,  //使用自定义窗体
    content: createInfoWindow(title, content.join("<br>")),  //传入 dom 对象，或者 html 字符串
    offset: new currentAMap.Pixel(15, -50)
  });

  infoWindow.open(map.value, e.target.getPosition());
}

const navigation = (item) => {
  return `https://uri.amap.com/navigation?from=${currentPosition.value.lng},${currentPosition.value.lat},startpoint&to=${item.lng},${item.lat},endpoint&mode=car&policy=0&src=mypage&coordinate=gaode&callnative=1`
}

const onSearch = (v) => {
  searchValue.value = v && v.trim()
  getParkingLotSearchInfo()
}

const onPosition = (position) => {
  open_nearby.value = false;
  map.value.setCenter(position)
}

const onSearchPosition = (position) => {
  open_search.value = false;
  map.value.setCenter(position)
}

</script>

<style>
#container {
  position: relative;
  padding: 0;
  margin: 0;
  width: 100%;
  height: 600px;
}

.content-window-card {
  position: relative;
  box-shadow: none;
  bottom: 0;
  left: 0;
  width: 200px;
  padding: 0;
}

.content-window-card p {
  height: 2rem;
}

.custom-info {
  border: solid 1px silver;
}

div.info-top {
  position: relative;
  background: none repeat scroll 0 0 #F9F9F9;
  border-bottom: 1px solid #CCC;
  border-radius: 5px 5px 0 0;
}

div.info-top div {
  display: inline-block;
  color: #333333;
  font-size: 14px;
  font-weight: bold;
  line-height: 31px;
  padding: 0 10px;
}

div.info-top img {
  position: absolute;
  top: 10px;
  right: 10px;
  transition-duration: 0.25s;
}

div.info-top img:hover {
  box-shadow: 0px 0px 5px #000;
}

div.info-middle {
  font-size: 12px;
  padding: 10px 6px;
  line-height: 20px;
}

div.info-bottom {
  height: 0px;
  width: 100%;
  clear: both;
  text-align: center;
}

div.info-bottom img {
  position: relative;
  z-index: 104;
}

span {
  margin-left: 5px;
  font-size: 11px;
}

.info-middle img {
  float: left;
  margin-right: 6px;
}

.ant-space {
  width: 100%;
  padding: 10px;
}

.full-modal {
  .ant-modal {
    max-width: 100%;
    top: 0;
    padding-bottom: 0;
    margin: 0;
  }

  .ant-modal-content {
    display: flex;
    flex-direction: column;
    height: calc(100vh);
  }

  .ant-modal-body {
    flex: 1;
  }
}

.listBtn {
  padding: 0 3px;
  font-size: 20px !important;
  font-weight: bold;
  position: relative;
  top: 30px;
  width: 100%;
  z-index: 100;

  .listBtn-span {
    font-size: 16px;
    position: relative;
    top: -2px;
  }
}

.search-input {
  position: relative;
  right: 5px;
  top: 50px;
}

.btn-nav {
  margin-top: 10px;
  margin-left: 40px;
  background-color: #1677ff;
  /* Green */
  border: none;
  border-radius: 12px;
  color: white;
  padding: 6px 16px;
  text-align: center;
  text-decoration: none;
  display: inline-bloc
}

.a-nav {
  color: black
}
</style>

