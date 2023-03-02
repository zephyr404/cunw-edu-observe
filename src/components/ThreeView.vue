<template>
  <div class="wrap">
    <div class="main">
      <div class="main-top">
        <div class="preview">
          <div class="item" v-for="(item, index) in angle" :key="index">
            <img v-show="item.show" :src="item.url" alt="" />
            <div class="explanation">
              <span>{{ item.text }}</span>
            </div>
          </div>
        </div>

        <div class="canvas">
          <canvas class="threejs" id="threejs" ref="threejs"></canvas>
          <div class="btn-group">
            <div
              :class="item.active ? 'item active' : 'item'"
              v-for="(item, index) in angle"
              :key="index"
              @click="handleChangeView(item)"
            >
              <span>{{ item.text }}</span>
            </div>
          </div>
        </div>
      </div>

      <div class="main-bottom">
        <div class="thumb">
          <div
            :class="activeThumb === item.id ? 'item active' : 'item'"
            v-for="(item, index) in thumbs"
            :key="index"
            @click="handleSetThumb(item)"
          >
            <img :src="item.url" alt="" />
            <span
              ><strong>{{ item.text }}</strong></span
            >
          </div>
        </div>
      </div>
    </div>

    <div class="footer">
      <div class="item item-reset" @click="handleReset">
        <img :src="tools[0].url" alt="" />
        <span>重置</span>
      </div>
      <div class="item item-intro" @click="showDialog = true">
        <img :src="tools[1].url" alt="" />
        <span>操作说明</span>
      </div>
    </div>

    <div class="dialog" v-show="showDialog">
      <div class="explanation">
        <div class="header">
          <h3>操作说明</h3>
        </div>
        <div class="body">
          <ul>
            <li>
              1.观察物体，长按几何体可进行任意角度旋转，观察不同角度物体的形状；
            </li>
            <li>2.点击右侧按钮，会出现几何体对应视图的形状；</li>
            <li>3.点击“重置”，恢复至初始状态。</li>
          </ul>
        </div>

        <div class="foot">
          <button @click="showDialog = false">关闭</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { watch, ref, onMounted } from "vue";
import * as THREE from "three";
import WebGL from "three/examples/jsm/capabilities/WebGL";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader";
import { Line2 } from "three/examples/jsm/lines/Line2";
import { LineGeometry } from "three/examples/jsm/lines/LineGeometry";
import { LineMaterial } from "three/examples/jsm/lines/LineMaterial";
import TWEEN from "@tweenjs/tween.js";

interface IThumb {
  id: number;
  text: string;
  url: any;
}

let activeThumb = ref<number>(1);

watch(activeThumb, (activeThumb: any, newActiveThumb: any) => {
  console.log(activeThumb, newActiveThumb);
});

// watch(
//   camera,
//   (val: any, newVal: any) => {
//     console.log(val, newVal);
//   },
//   {
//     deep: true,
//     immediate: true,
//   }
// );

let thumbs = ref<Array<IThumb>>([
  {
    id: 1,
    text: "模型1",
    url: require("@/assets/images/matrix/1.png"),
  },
  {
    id: 2,
    text: "模型2",
    url: require("@/assets/images/matrix/2.png"),
  },
  {
    id: 3,
    text: "模型3",
    url: require("@/assets/images/matrix/3.png"),
  },
  {
    id: 4,
    text: "模型4",
    url: require("@/assets/images/matrix/4.png"),
  },
  {
    id: 5,
    text: "模型5",
    url: require("@/assets/images/matrix/5.png"),
  },
]);

interface IControl {
  id: number;
  direction: string;
  text: string;
  coordinate: Array<number>;
  url: string;
  active: boolean;
  show: boolean;
}

let angle = ref<Array<IControl>>([
  {
    id: 1,
    direction: "front",
    text: "从前面看",
    coordinate: [0, 0, 10],
    url: require(`@/assets/images/matrix/1-1.png`),
    active: false,
    show: false,
  },
  {
    id: 2,
    direction: "top",
    text: "从上面看",
    coordinate: [0, 10, 0],
    url: require(`@/assets/images/matrix/1-2.png`),
    active: false,
    show: false,
  },
  {
    id: 3,
    direction: "left",
    text: "从左面看",
    coordinate: [-10, 0, 0],
    url: require(`@/assets/images/matrix/1-3.png`),
    active: false,
    show: false,
  },
]);

interface ITool {
  id: number;
  text: string;
  url: any;
}
let tools = ref<Array<ITool>>([
  {
    id: 1,
    text: "重置",
    url: require("@/assets/images/icons/reset.png"),
  },
  {
    id: 2,
    text: "操作说明",
    url: require("@/assets/images/icons/intro.png"),
  },
]);

let showDialog = ref<boolean>(false);

const handleSetThumb = (item: any) => {
  activeThumb.value = item.id;
  handleRemoveAllGroup();
  angle.value.forEach((el, index) => {
    el.show = false;
    el.active = false;
    el.url = require(`@/assets/images/matrix/${item.id}-${index + 1}.png`);
  });
  init();
};

// THREEJS
// 创建场景
const scene: any = new THREE.Scene();

// 场景背景色
// scene.background = new THREE.Color("#000");

// 辅助坐标系
// const axesHelper = new THREE.AxesHelper(500);
// scene.add(axesHelper);

// 创建相机
let camera: any = null;

// 创建相机控制器
let controls: any = null;

// renderer
let renderer: any = null;

// 初始化
const init = () => {
  let width: number = (document.querySelector(".canvas") as HTMLElement)
    .offsetWidth;
  let height: number = (document.querySelector(".canvas") as HTMLElement)
    .offsetHeight;
  let k = width / height;
  //三维场景显示范围控制系数，系数越大，显示的范围越大
  let s = 400;
  camera = new THREE.OrthographicCamera(-s * k, s * k, s, -s, 1, 1000);
  camera.position.set(-200, 100, 500);
  camera.lookAt(0, 0, 0);
  scene.add(camera);

  // 绘制图形
  draw();

  // 获取canvas元素并渲染到canvas上
  const canvas: any = document.querySelector("#threejs");
  renderer = new THREE.WebGLRenderer({
    canvas,
    antialias: true,
    alpha: true,
  });
  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.setSize(
    (document.querySelector(".canvas") as HTMLElement).offsetWidth,
    (document.querySelector(".canvas") as HTMLElement).offsetHeight
  );

  // 控制相机
  controls = new OrbitControls(camera, renderer.domElement);

  // 阻尼惯性
  controls.enableDamping = true;
  controls.dampingFactor = 0.1;

  // 摄像机平移
  controls.enablePan = false;

  // 缩放
  controls.enableZoom = true;
  controls.maxZoom = 1.5;
  controls.minZoom = 0.8;

  // 自动旋转
  controls.autoRotate = false;

  // 渲染
  function animate() {
    requestAnimationFrame(animate);
    TWEEN.update();
    controls.update();
    renderer.render(scene, camera);
  }

  if (WebGL.isWebGLAvailable()) {
    animate();
  } else {
    const warning = WebGL.getWebGLErrorMessage();
    alert(warning);
  }

  window.onresize = function () {
    let width: number = (document.querySelector(".canvas") as HTMLElement)
      .offsetWidth;
    let height: number = (document.querySelector(".canvas") as HTMLElement)
      .offsetHeight;
    // 重置渲染器输出画布canvas尺寸
    renderer.setSize(width, height);
    // 重置相机投影的相关参数
    k = width / height; //窗口宽高比
    camera.left = -s * k;
    camera.right = s * k;
    camera.top = s;
    camera.bottom = -s;
    // 渲染器执行render方法的时候会读取相机对象的投影矩阵属性projectionMatrix
    // 但是不会每渲染一帧，就通过相机的属性计算投影矩阵(节约计算资源)
    // 如果相机的一些属性发生了变化，需要执行updateProjectionMatrix ()方法更新相机的投影矩阵
    camera.updateProjectionMatrix();
  };
};

const handleRemoveAllGroup = () => {
  scene.traverse(function (obj: any) {
    if (obj.type === "Mesh") {
      // console.log("Mesh", obj);
      obj.geometry.dispose();
      obj.material.dispose();
    }
    if (obj.type === "Group") {
      if (obj.name === "box") {
        // console.log("Group", obj);
        scene.remove(obj);
      }
    }
  });
  // console.log("Group", scene);
};

const handleCameraMove = (position: any, time: any) => {
  const tween = new TWEEN.Tween(camera.position)
    .to(position, time)
    .easing(TWEEN.Easing.Quadratic.InOut);
  tween.onUpdate((pos) => {
    console.log(camera.position, pos);
    // camera.position.set(pos);
    // camera.lookAt(new THREE.Vector3(0, 0, 0));
    // camera.updateProjectionMatrix();
  });
  return tween;
};

const handleChangeView = (item: any) => {
  item.active = !item.active;
  item.show = true;
  let p: any = {};
  switch (item.direction) {
    case "front":
      p = { x: 0, y: 0, z: 500 };
      break;
    case "top":
      p = { x: 0, y: 500, z: 0 };
      break;
    case "left":
      p = { x: -500, y: 0, z: 0 };
      break;
    default:
      break;
  }

  const tween = handleCameraMove(p, 1000);
  tween.start();

  function animate() {
    requestAnimationFrame(animate);
    TWEEN.update();
    renderer.render(scene, camera);
  }
  animate();
};

const handleReset = () => {
  handleRemoveAllGroup();
  activeThumb.value = 1;
  angle.value.forEach((el, index) => {
    el.show = false;
    el.active = false;
    el.url = require(`@/assets/images/matrix/1-${index + 1}.png`);
  });

  init();
};

onMounted(() => {
  init();
});

let draw = () => {
  if (activeThumb.value === 1) {
    let group = new THREE.Group();
    group.name = "box";
    // 第1个立方体
    const geometry = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges = new THREE.EdgesGeometry(geometry);
    let edgesMtl = new THREE.LineBasicMaterial({
      color: 0x000000,
      linewidth: 15,
    });
    let cubeLine = new THREE.LineSegments(cubeEdges, edgesMtl);
    geometry.computeBoundingSphere();
    cubeLine.frustumCulled = false;
    const material = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube = new THREE.Mesh(geometry, material);
    cube.add(cubeLine);
    group.add(cube);

    // 第2个立方体
    const geometry2 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges2 = new THREE.EdgesGeometry(geometry2, 10);
    let edgesMtl2 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine2 = new THREE.LineSegments(cubeEdges2, edgesMtl2);
    const material2 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube2 = new THREE.Mesh(geometry2, material2);
    cube2.translateX(100);
    cube2.add(cubeLine2);
    group.add(cube2);
    // 第3个立方体
    const geometry3 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges3 = new THREE.EdgesGeometry(geometry3);
    let edgesMtl3 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine3 = new THREE.LineSegments(cubeEdges3, edgesMtl3);
    const material3 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube3 = new THREE.Mesh(geometry3, material3);
    cube3.translateX(200);
    cube3.add(cubeLine3);
    group.add(cube3);
    // 第4个立方体
    const geometry4 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges4 = new THREE.EdgesGeometry(geometry4);
    let edgesMtl4 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine4 = new THREE.LineSegments(cubeEdges4, edgesMtl4);
    const material4 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube4 = new THREE.Mesh(geometry4, material4);
    cube4.translateX(-100);
    cube4.add(cubeLine4);
    group.add(cube4);
    // 第5个立方体
    const geometry5 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges5 = new THREE.EdgesGeometry(geometry5);
    let edgesMtl5 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine5 = new THREE.LineSegments(cubeEdges5, edgesMtl5);
    const material5 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube5 = new THREE.Mesh(geometry5, material5);
    cube5.translateZ(100);
    cube5.add(cubeLine5);
    group.add(cube5);
    // 第6个立方体
    const geometry6 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges6 = new THREE.EdgesGeometry(geometry6);
    let edgesMtl6 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine6 = new THREE.LineSegments(cubeEdges6, edgesMtl6);
    const material6 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube6 = new THREE.Mesh(geometry6, material6);
    cube6.translateY(100);
    cube6.add(cubeLine6);
    group.add(cube6);
    // 柱体
    const geometry8 = new THREE.CylinderGeometry(2, 2, 300, 40, 40);
    let cubeEdges8 = new THREE.EdgesGeometry(geometry8);
    let edgesMtl8 = new THREE.LineBasicMaterial({ color: 0xffffff });
    let cubeLine8 = new THREE.LineSegments(cubeEdges8, edgesMtl8);
    const material8 = new THREE.MeshBasicMaterial({
      color: 0xffffff,
      wireframe: false,
    });
    const cube8 = new THREE.Mesh(geometry8, material8);
    cube8.add(cubeLine8);
    cube8.translateX(350);
    cube8.translateY(-50);
    cube8.translateZ(50);
    cube8.rotateX(Math.PI / 2);
    group.add(cube8);
    // 锥体
    const geometry9 = new THREE.ConeGeometry(5, 30, 32);
    const material9 = new THREE.MeshBasicMaterial({ color: 0xffffff });
    const cone = new THREE.Mesh(geometry9, material9);
    cone.translateX(350);
    cone.translateY(-50);
    cone.translateZ(200);
    cone.rotateX(Math.PI / 2);
    group.add(cone);
    group.name = "box";
    scene.add(group);
    // console.log(group.rotation);
  } else if (activeThumb.value === 2) {
    let group = new THREE.Group();
    // 第1个立方体
    const geometry = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges = new THREE.EdgesGeometry(geometry);
    let edgesMtl = new THREE.LineBasicMaterial({
      color: 0x000000,
    });
    let cubeLine = new THREE.LineSegments(cubeEdges, edgesMtl);
    const material = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube = new THREE.Mesh(geometry, material);
    cube.add(cubeLine);
    group.add(cube);
    // 第2个立方体
    const geometry2 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges2 = new THREE.EdgesGeometry(geometry2, 10);
    let edgesMtl2 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine2 = new THREE.LineSegments(cubeEdges2, edgesMtl2);
    const material2 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube2 = new THREE.Mesh(geometry2, material2);
    cube2.translateX(100);
    cube2.add(cubeLine2);
    group.add(cube2);
    // 第3个立方体
    const geometry3 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges3 = new THREE.EdgesGeometry(geometry3);
    let edgesMtl3 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine3 = new THREE.LineSegments(cubeEdges3, edgesMtl3);
    const material3 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube3 = new THREE.Mesh(geometry3, material3);
    cube3.translateZ(100);
    cube3.add(cubeLine3);
    group.add(cube3);
    // 第4个立方体
    const geometry4 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges4 = new THREE.EdgesGeometry(geometry4);
    let edgesMtl4 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine4 = new THREE.LineSegments(cubeEdges4, edgesMtl4);
    const material4 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube4 = new THREE.Mesh(geometry4, material4);
    cube4.translateX(-100);
    cube4.add(cubeLine4);
    group.add(cube4);
    // 第5个立方体
    const geometry5 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges5 = new THREE.EdgesGeometry(geometry5);
    let edgesMtl5 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine5 = new THREE.LineSegments(cubeEdges5, edgesMtl5);
    const material5 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube5 = new THREE.Mesh(geometry5, material5);
    cube5.translateZ(100);
    cube5.add(cubeLine5);
    group.add(cube5);
    // 第6个立方体
    const geometry6 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges6 = new THREE.EdgesGeometry(geometry6);
    let edgesMtl6 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine6 = new THREE.LineSegments(cubeEdges6, edgesMtl6);
    const material6 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube6 = new THREE.Mesh(geometry6, material6);
    cube6.translateY(100);
    cube6.add(cubeLine6);
    group.add(cube6);
    // 柱体
    const geometry8 = new THREE.CylinderGeometry(2, 2, 300, 40, 40);
    let cubeEdges8 = new THREE.EdgesGeometry(geometry8);
    let edgesMtl8 = new THREE.LineBasicMaterial({ color: 0xffffff });
    let cubeLine8 = new THREE.LineSegments(cubeEdges8, edgesMtl8);
    const material8 = new THREE.MeshBasicMaterial({
      color: 0xffffff,
      wireframe: false,
    });
    const cube8 = new THREE.Mesh(geometry8, material8);
    cube8.add(cubeLine8);
    cube8.translateX(250);
    cube8.translateY(-50);
    cube8.translateZ(50);
    cube8.rotateX(Math.PI / 2);
    group.add(cube8);
    // 锥体
    const geometry9 = new THREE.ConeGeometry(5, 30, 32);
    const material9 = new THREE.MeshBasicMaterial({ color: 0xffffff });
    const cone = new THREE.Mesh(geometry9, material9);
    cone.translateX(250);
    cone.translateY(-50);
    cone.translateZ(200);
    cone.rotateX(Math.PI / 2);
    group.add(cone);
    group.name = "box";
    scene.add(group);
  } else if (activeThumb.value === 3) {
    let group = new THREE.Group();
    // 第1个立方体
    const geometry = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges = new THREE.EdgesGeometry(geometry);
    let edgesMtl = new THREE.LineBasicMaterial({
      color: 0x000000,
    });
    let cubeLine = new THREE.LineSegments(cubeEdges, edgesMtl);
    const material = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube = new THREE.Mesh(geometry, material);
    cube.add(cubeLine);
    group.add(cube);
    // 第2个立方体
    const geometry2 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges2 = new THREE.EdgesGeometry(geometry2, 10);
    let edgesMtl2 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine2 = new THREE.LineSegments(cubeEdges2, edgesMtl2);
    const material2 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube2 = new THREE.Mesh(geometry2, material2);
    cube2.translateX(100);
    cube2.add(cubeLine2);
    group.add(cube2);
    // 第3个立方体
    const geometry3 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges3 = new THREE.EdgesGeometry(geometry3);
    let edgesMtl3 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine3 = new THREE.LineSegments(cubeEdges3, edgesMtl3);
    const material3 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube3 = new THREE.Mesh(geometry3, material3);
    cube3.translateZ(100);
    cube3.add(cubeLine3);
    group.add(cube3);
    // 第5个立方体
    const geometry5 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges5 = new THREE.EdgesGeometry(geometry5);
    let edgesMtl5 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine5 = new THREE.LineSegments(cubeEdges5, edgesMtl5);
    const material5 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube5 = new THREE.Mesh(geometry5, material5);
    cube5.translateX(100);
    cube5.translateZ(100);
    cube5.add(cubeLine5);
    group.add(cube5);
    // 第6个立方体
    const geometry6 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges6 = new THREE.EdgesGeometry(geometry6);
    let edgesMtl6 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine6 = new THREE.LineSegments(cubeEdges6, edgesMtl6);
    const material6 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube6 = new THREE.Mesh(geometry6, material6);
    cube6.translateY(100);
    cube6.add(cubeLine6);
    group.add(cube6);
    // 柱体
    const geometry8 = new THREE.CylinderGeometry(2, 2, 300, 40, 40);
    let cubeEdges8 = new THREE.EdgesGeometry(geometry8);
    let edgesMtl8 = new THREE.LineBasicMaterial({ color: 0xffffff });
    let cubeLine8 = new THREE.LineSegments(cubeEdges8, edgesMtl8);
    const material8 = new THREE.MeshBasicMaterial({
      color: 0xffffff,
      wireframe: false,
    });
    const cube8 = new THREE.Mesh(geometry8, material8);
    cube8.add(cubeLine8);
    cube8.translateX(250);
    cube8.translateY(-50);
    cube8.translateZ(50);
    cube8.rotateX(Math.PI / 2);
    group.add(cube8);
    // 锥体
    const geometry9 = new THREE.ConeGeometry(5, 30, 32);
    const material9 = new THREE.MeshBasicMaterial({ color: 0xffffff });
    const cone = new THREE.Mesh(geometry9, material9);
    cone.translateX(250);
    cone.translateY(-50);
    cone.translateZ(200);
    cone.rotateX(Math.PI / 2);
    group.add(cone);
    group.name = "box";
    scene.add(group);
  } else if (activeThumb.value === 4) {
    let group = new THREE.Group();
    // 第1个立方体
    const geometry = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges = new THREE.EdgesGeometry(geometry);
    let edgesMtl = new THREE.LineBasicMaterial({
      color: 0x000000,
    });
    let cubeLine = new THREE.LineSegments(cubeEdges, edgesMtl);
    const material = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube = new THREE.Mesh(geometry, material);
    cube.add(cubeLine);
    group.add(cube);
    // 第2个立方体
    const geometry2 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges2 = new THREE.EdgesGeometry(geometry2, 10);
    let edgesMtl2 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine2 = new THREE.LineSegments(cubeEdges2, edgesMtl2);
    const material2 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube2 = new THREE.Mesh(geometry2, material2);
    cube2.translateX(100);
    cube2.add(cubeLine2);
    group.add(cube2);
    // 第3个立方体
    const geometry3 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges3 = new THREE.EdgesGeometry(geometry3);
    let edgesMtl3 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine3 = new THREE.LineSegments(cubeEdges3, edgesMtl3);
    const material3 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube3 = new THREE.Mesh(geometry3, material3);
    cube3.translateZ(100);
    cube3.add(cubeLine3);
    group.add(cube3);
    // 第4个立方体
    const geometry4 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges4 = new THREE.EdgesGeometry(geometry4);
    let edgesMtl4 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine4 = new THREE.LineSegments(cubeEdges4, edgesMtl4);
    const material4 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube4 = new THREE.Mesh(geometry4, material4);
    cube4.translateX(100);
    cube4.translateY(100);
    cube4.add(cubeLine4);
    group.add(cube4);
    // 第5个立方体
    const geometry5 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges5 = new THREE.EdgesGeometry(geometry5);
    let edgesMtl5 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine5 = new THREE.LineSegments(cubeEdges5, edgesMtl5);
    const material5 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube5 = new THREE.Mesh(geometry5, material5);
    cube5.translateZ(100);
    cube5.add(cubeLine5);
    group.add(cube5);
    // 第6个立方体
    const geometry6 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges6 = new THREE.EdgesGeometry(geometry6);
    let edgesMtl6 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine6 = new THREE.LineSegments(cubeEdges6, edgesMtl6);
    const material6 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube6 = new THREE.Mesh(geometry6, material6);
    cube6.translateY(100);
    cube6.add(cubeLine6);
    group.add(cube6);
    // 柱体
    const geometry8 = new THREE.CylinderGeometry(2, 2, 300, 40, 40);
    let cubeEdges8 = new THREE.EdgesGeometry(geometry8);
    let edgesMtl8 = new THREE.LineBasicMaterial({ color: 0xffffff });
    let cubeLine8 = new THREE.LineSegments(cubeEdges8, edgesMtl8);
    const material8 = new THREE.MeshBasicMaterial({
      color: 0xffffff,
      wireframe: false,
    });
    const cube8 = new THREE.Mesh(geometry8, material8);
    cube8.add(cubeLine8);
    cube8.translateX(250);
    cube8.translateY(-50);
    cube8.translateZ(50);
    cube8.rotateX(Math.PI / 2);
    group.add(cube8);
    // 锥体
    const geometry9 = new THREE.ConeGeometry(5, 30, 32);
    const material9 = new THREE.MeshBasicMaterial({ color: 0xffffff });
    const cone = new THREE.Mesh(geometry9, material9);
    cone.translateX(250);
    cone.translateY(-50);
    cone.translateZ(200);
    cone.rotateX(Math.PI / 2);
    group.add(cone);
    group.name = "box";
    scene.add(group);
  } else if (activeThumb.value === 5) {
    let group = new THREE.Group();
    // 第1个立方体
    const geometry = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges = new THREE.EdgesGeometry(geometry);
    let edgesMtl = new THREE.LineBasicMaterial({
      color: 0x000000,
    });
    let cubeLine = new THREE.LineSegments(cubeEdges, edgesMtl);
    const material = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube = new THREE.Mesh(geometry, material);
    cube.add(cubeLine);
    group.add(cube);
    // 第2个立方体
    const geometry2 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges2 = new THREE.EdgesGeometry(geometry2, 10);
    let edgesMtl2 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine2 = new THREE.LineSegments(cubeEdges2, edgesMtl2);
    const material2 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube2 = new THREE.Mesh(geometry2, material2);
    cube2.translateX(100);
    cube2.add(cubeLine2);
    group.add(cube2);
    // 第3个立方体
    const geometry3 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges3 = new THREE.EdgesGeometry(geometry3);
    let edgesMtl3 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine3 = new THREE.LineSegments(cubeEdges3, edgesMtl3);
    const material3 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube3 = new THREE.Mesh(geometry3, material3);
    cube3.translateZ(100);
    cube3.add(cubeLine3);
    group.add(cube3);
    // 第4个立方体
    const geometry4 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges4 = new THREE.EdgesGeometry(geometry4);
    let edgesMtl4 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine4 = new THREE.LineSegments(cubeEdges4, edgesMtl4);
    const material4 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube4 = new THREE.Mesh(geometry4, material4);
    cube4.translateX(-100);
    cube4.add(cubeLine4);
    group.add(cube4);
    // 第5个立方体
    const geometry5 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges5 = new THREE.EdgesGeometry(geometry5);
    let edgesMtl5 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine5 = new THREE.LineSegments(cubeEdges5, edgesMtl5);
    const material5 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube5 = new THREE.Mesh(geometry5, material5);
    cube5.translateZ(100);
    cube5.add(cubeLine5);
    group.add(cube5);
    // 第6个立方体
    const geometry6 = new THREE.BoxGeometry(100, 100, 100);
    let cubeEdges6 = new THREE.EdgesGeometry(geometry6);
    let edgesMtl6 = new THREE.LineBasicMaterial({ color: 0x000000 });
    let cubeLine6 = new THREE.LineSegments(cubeEdges6, edgesMtl6);
    const material6 = new THREE.MeshBasicMaterial({
      color: 0xbfeeeb,
      wireframe: false,
    });
    const cube6 = new THREE.Mesh(geometry6, material6);
    cube6.translateX(100);
    cube6.translateY(100);
    cube6.add(cubeLine6);
    group.add(cube6);
    // 柱体
    const geometry8 = new THREE.CylinderGeometry(2, 2, 300, 40, 40);
    let cubeEdges8 = new THREE.EdgesGeometry(geometry8);
    let edgesMtl8 = new THREE.LineBasicMaterial({ color: 0xffffff });
    let cubeLine8 = new THREE.LineSegments(cubeEdges8, edgesMtl8);
    const material8 = new THREE.MeshBasicMaterial({
      color: 0xffffff,
      wireframe: false,
    });
    const cube8 = new THREE.Mesh(geometry8, material8);
    cube8.add(cubeLine8);
    cube8.translateX(250);
    cube8.translateY(-50);
    cube8.translateZ(50);
    cube8.rotateX(Math.PI / 2);
    group.add(cube8);
    // 锥体
    const geometry9 = new THREE.ConeGeometry(5, 30, 32);
    const material9 = new THREE.MeshBasicMaterial({ color: 0xffffff });
    const cone = new THREE.Mesh(geometry9, material9);
    cone.translateX(250);
    cone.translateY(-50);
    cone.translateZ(200);
    cone.rotateX(Math.PI / 2);
    group.add(cone);
    group.name = "box";
    scene.add(group);
  } else {
    console.log("none");
  }
};
</script>

<style scoped lang="scss">
.wrap {
  height: 100vh;
  width: 100vw;
  overflow: hidden;
  position: relative;

  .main {
    height: calc(100vh - 10em);
    padding: 5em 7em 0 7em;
    box-sizing: border-box;

    .main-top {
      width: 100%;
      height: calc(100% - 20em);
      display: flex;
      flex-flow: row nowrap;
      justify-content: space-between;
      align-items: flex-start;
      overflow: hidden;
      .preview {
        height: 100%;
        margin-right: 2em;
        display: flex;
        flex-flow: column nowrap;
        justify-content: space-between;
        align-items: center;

        .item {
          position: relative;
          background-color: #e0e5eb;
          width: 34em;
          min-width: 34em;
          height: calc((100% - 4em) / 3);
          // flex-grow: 1;
          margin-bottom: 2em;
          box-sizing: border-box;

          &:last-child {
            margin-bottom: 0;
          }

          img {
            width: 100%;
            height: 100%;
            -webkit-animation: fadeinout 3s linear forwards;
            animation: fadeinout 3s linear forwards;
          }

          @-webkit-keyframes fadeinout {
            0% {
              opacity: 1;
            }
            50% {
              opacity: 0.5;
            }
            100% {
              opacity: 0;
            }
          }

          @keyframes fadeinout {
            0% {
              opacity: 1;
            }
            50% {
              opacity: 0.5;
            }
            100% {
              opacity: 0;
            }
          }

          @-webkit-keyframes fadeinout {
            0% {
              opacity: 0;
            }
            50% {
              opacity: 0.5;
            }
            100% {
              opacity: 1;
            }
          }

          @keyframes fadeinout {
            0% {
              opacity: 0;
            }
            50% {
              opacity: 0.5;
            }
            100% {
              opacity: 1;
            }
          }

          .explanation {
            position: absolute;
            top: 0;
            left: 0;
            width: 8rem;
            height: 2.8rem;
            line-height: 2.8rem;
            text-align: center;
            font-size: 1.6rem;
            background-color: #ebc197;
          }
        }
      }

      .canvas {
        flex-grow: 1;
        height: 100%;
        width: 100%;
        background-color: #88969c;
        position: relative;
        overflow: hidden;

        #threejs {
          position: absolute;
          top: 0;
          left: 0;
          width: 100%;
          height: 100%;
          overflow: hidden;
        }

        .btn-group {
          position: absolute;
          right: 3rem;
          bottom: 3rem;

          .item {
            width: 14rem;
            height: 4.6rem;
            line-height: 4.6rem;
            text-align: center;
            font-size: 2rem;
            background-color: #fff;
            border: 0.1rem solid #88969c;
            opacity: 0.8;
            border-radius: 0.8rem;
            margin-bottom: 1.6rem;
            transition: background-color 0.2s, color 0.2s;

            &:last-child {
              margin-bottom: 0;
            }

            &.active {
              background-color: #84ceca;
              border: 0.1rem solid #009890;
            }
          }
        }
      }
    }

    .main-bottom {
      width: 100%;
      height: 16em;
      margin: 2em 0;
      border: 1px dashed #2f847f;
      border-radius: 2em;

      .thumb {
        width: 100%;
        height: 100%;
        display: flex;
        flex-flow: row nowrap;
        justify-content: flex-start;
        align-items: center;
        overflow: scroll hidden;

        &::-webkit-scrollbar {
          background-color: transparent;
        }
        &::-webkit-scrollbar-track-piece {
          background-color: transparent;
        }

        .item {
          display: flex;
          flex-flow: column nowrap;
          justify-content: center;
          align-items: center;
          width: 20em;
          min-width: 20em;
          overflow: hidden;

          img {
            width: 14em;
          }

          span {
            font-size: 1.6em;
          }
        }
      }
    }
  }

  .footer {
    height: 10em;
    background-color: #f3f6fc;
    padding: 0 7em;
    box-sizing: border-box;
    display: flex;
    flex-flow: row nowrap;
    justify-content: right;
    align-items: center;

    .item {
      margin: 1em;
      height: 60%;
      width: 10em;
      display: flex;
      flex-flow: column nowrap;
      justify-content: space-around;
      align-items: center;

      img {
        width: 3.2em;
      }

      span {
        font-size: 1.6em;
      }
    }
  }

  .dialog {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    overflow: hidden;
    background-color: rgba($color: #000000, $alpha: 0.5);

    .explanation {
      width: 40%;
      height: 60%;
      margin: 10% 30%;
      background-color: #fff;
      border-radius: 0.5rem;
      display: inline-block;
      ::-webkit-scrollbar {
        display: none;
      }
      scrollbar-width: none;
      -ms-overflow-style: none;

      .header {
        font-size: 2em;
        padding: 0 2em;
      }

      .body {
        text-align: left;
        height: 75%;
        max-height: 75%;
        overflow: hidden scroll;
        padding: 0 4em;

        ul {
          list-style: none;
          margin: 0;
          padding: 0;

          li {
            font-size: 2rem;
          }
        }
      }

      .foot {
        text-align: center;

        button {
          font-size: 2em;
          margin-top: 1em;
        }
      }
    }
  }
}
</style>
