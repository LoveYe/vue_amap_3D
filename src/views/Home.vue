<template>
  <div id="container" style="width: 100%; height: 100%; resize: both"></div>
</template>


<script>
import remoteLoad from "./amap.js";
import { OBJLoader, MTLLoader } from "three-obj-mtl-loader";
export default {
  data() {
    return {
      map: null,
      sobject3Dlayer: null,
    };
  },
  async created() {
    await remoteLoad(
      "https://webapi.amap.com/maps?v=1.4.15&key=8b6a690023d1b935dd6710f5a9bd784d&plugin=AMap.ToolBar,AMap.ControlBar"
    );
    this.initMap();
  },

  methods: {
    //初始化地图
    initMap() {
      let AMap = window.AMap;
      let _this = this;
      // 地图的初始化
      _this.map = new AMap.Map("container", {
        resizeEnable: true,
        rotateEnable: true,
        pitchEnable: true,
        zoom: 19, //18.6,
        pitch: 75,
        rotation: 25,
        skyColor: "#ffffff",
        viewMode: "3D", //��Ǵ3D˓ͼ,ĬɏΪ�رӍ
        buildingAnimation: true, //¥�鴶Жˇ�񵸶���
        mapStyle: "amap://styles/6cda541a7167495c66a7971e109e7451",
        expandZoomRange: true,
        zooms: [17.5, 19.9],
        center: [106.323597, 29.516401], //106.323238, lat: 29.515816
        //bounds: new AMap.Bounds([106.319848, 29.51133], [106.334276, 29.519892]),
        // layers: [
        //   new AMap.TileLayer(),
        //   new AMap.ImageLayer({
        //     url: "./parkMap.png",
        //     zooms: [15, 20],
        //     bounds: new AMap.Bounds(
        //       [106.322304, 29.51505], //29.51509 515034 106.322204  zooms: [15, 28] 106.322104, 29.514434
        //       [106.329201, 29.51742] //29.51740 106.329401, 29.51814
        //     ),
        //   }),
        // ],
      });

      //初始化3D
      _this.init3DPlayer();

      //点击地图事件  寻找点击的3D模型触发一系列事件
      _this.map.on("click", function (ev) {
        var pixel = ev.pixel;
        var px = new AMap.Pixel(pixel.x, pixel.y);
        var obj =
          _this.map.getObject3DByContainerPos(
            px,
            [_this.sobject3Dlayer],
            true
          ) || {};

        // 选中的 face 所在的索引
        // var index = obj.index;
        // 选中的 object3D 对象，这里为当前 Mesh
        //console.log('click map', ev.lnglat);
        // 点击如果没有选中灯杆
        if (!obj[0]) {
          // lat: 29.516997 lng: 106.323903
          // let lnglatclicked = ev.lnglat;
          // let center = [lnglatclicked.lng, lnglatclicked.lat];
          // map.setZoomAndCenter(20, center);
          // map.setRotation(25);
          // map.setPitch(60);
          return;
        }
        // var object = obj[0].object;
        //console.log('点击灯杆', obj[0].object);
        // 被拾取到的对象和拾取射线的交叉点的3D坐标
        // var point = obj.point;
        // 交叉点距透视原点的距离
        // var distance = obj.distance;

        console.log("我点击了第几个", obj[0].object);
        //alert(obj[0].object.id);
        // updateMesh(object);
        //updateInfo(obj);
        //updateMesh(object);
        // obj[0] ? (window.parent.postMessage({
        //   flag: 0,
        //   carrierId: obj[0].object.id,
        //   lights: result
        // })) : null
      });
    },
    //初始化3D模型
    init3DPlayer() {
      let AMap = window.AMap;
      let _this = this;
      let mtlLoader = new MTLLoader();
      _this.sobject3Dlayer = new AMap.Object3DLayer();
      _this.map
        ? mtlLoader.load(`static/elight.mtl`, function (materials) {
            materials.preload();
            // 初始化obj解析器
            let objLoader = new OBJLoader();

            // 开始引入3D对象文件进行解析
            objLoader.load(`static/elight.obj`, function (obj) {
              // 注意这个回调的参数obj就是我们传入的3D对象
              // 对象的children就是我们要的mesh对象 如下
              console.log("mesh", obj);
              let meshes = obj.children.length == 0 ? [] : obj.children;
              let colorBg = [0.1, 0.93, 0.72];
              // 接下来很重要
              // 需要就是我们我们把这个mesh对象转变成高德地图可以认识的mesh对象
              //
              meshes.forEach((meshItem) => {
                let vecticesF3 = meshItem.geometry.attributes.position;
                let vecticesNormal3 = meshItem.geometry.attributes.normal;
                let vecticesUV2 = meshItem.geometry.attributes.uv;
                let vectexCount = vecticesF3.count;
                let material = meshItem.material[0] || meshItem.material;

                // 注意这里是高德地图的3d解析器
                let mesh = new AMap.Object3D.MeshAcceptLights();
                let geometry = mesh.geometry;

                // 以下就是改变3D模型的颜色
                for (var j = 0; j < vectexCount; j += 1) {
                  var s = j * 3;
                  geometry.vertices.push(
                    vecticesF3.array[s],
                    vecticesF3.array[s + 2],
                    -vecticesF3.array[s + 1]
                  );

                  if (vecticesNormal3) {
                    geometry.vertexNormals.push(
                      vecticesNormal3.array[s],
                      vecticesNormal3.array[s + 2],
                      -vecticesNormal3.array[s + 1]
                    );
                  }
                  if (vecticesUV2) {
                    geometry.vertexUVs.push(
                      vecticesUV2.array[j * 2],
                      1 - vecticesUV2.array[j * 2 + 1]
                    );
                  }
                  // 真实颜色 0.81, 0.73, 0.45
                  geometry.vertexColors.push(...colorBg, 0.9); //0.87,0.93,0.72
                }

                // 以下是根据自己项目需求对这个3d对象调整颜色，旋转角度等属性的  可添加自定义属性
                mesh.DEPTH_TEST = material.depthTest;
                mesh.transparent = true; //opacity<1;
                mesh.scale(24, 18, 9); //11
                // mesh.rotateZ(20);
                mesh.position(new AMap.LngLat(106.323597, 29.516401));
                // mesh.id = nid; //'light'+lightId;
                mesh.colorFlag = colorBg[0] == 0.81 ? "00" : "11";
                console.log("地图mesh", mesh);
                // _this.meshes.push(mesh);

                // 最后需要把这个3D对象添加到地图的3D解析器中
                _this.sobject3Dlayer.add(mesh);
              });

              // 然后需要把所有的3D对象添加到地图上  就可以看到地图上的3D模型了
              _this.map.add(_this.sobject3Dlayer);
            });
          })
        : null;
    },
  },
};
</script>

<style>
#container {
  height: 100%;
}
</style>