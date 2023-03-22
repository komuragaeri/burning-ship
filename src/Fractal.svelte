<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';

  let container;
  let scene, camera, renderer;
  let isDragging = false;
  let previousMousePosition = { x: 0, y: 0 };

  // 頂点シェーダー
  const vertexShader = `
    varying vec2 vUv;
  
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `;
  
  // フラグメントシェーダー
  const fragmentShader = `
    precision highp float;
    varying vec2 vUv;
  
    vec2 burningShip(vec2 c) {
      vec2 z = vec2(0.0);
      for (int i = 0; i < 100; i++) {
        z = vec2(abs(z.x), abs(z.y));
        z = vec2(z.x * z.x - z.y * z.y, 2.0 * z.x * z.y) + c;
        if (length(z) > 4.0) {
          return vec2(float(i) / 100.0);
        }
      }
      return vec2(0.0);
    }
  
    void main() {
      vec2 c = (vec2(vUv.x, 1.0 - vUv.y) * 2.0 - 1.0) * vec2(2.0, 2.0);
      vec2 color = burningShip(c);
      gl_FragColor = vec4(color, 0.0, 1.0);
    }
  `;

  onMount(() => {
    init();
  });

  function handleWheel(event) {
    const zoomFactor = 1.1;
    const delta = event.deltaY < 0 ? 1 / zoomFactor : zoomFactor;
  
    // マウスカーソルの位置を正規化座標系に変換
    const mouseX = (event.clientX / window.innerWidth) * 2 - 1;
    const mouseY = -(event.clientY / window.innerHeight) * 2 + 1;
  
    // カメラの範囲を変更する前のマウスカーソルのワールド座標を計算
    const preZoomX = mouseX * (camera.right - camera.left) / 2 + (camera.right + camera.left) / 2;
    const preZoomY = mouseY * (camera.top - camera.bottom) / 2 + (camera.top + camera.bottom) / 2;
  
    // カメラの範囲を変更
    camera.left *= delta;
    camera.right *= delta;
    camera.top *= delta;
    camera.bottom *= delta;
  
    // カメラの範囲を変更した後のマウスカーソルのワールド座標を計算
    const postZoomX = mouseX * (camera.right - camera.left) / 2 + (camera.right + camera.left) / 2;
    const postZoomY = mouseY * (camera.top - camera.bottom) / 2 + (camera.top + camera.bottom) / 2;
  
    // マウスカーソルの位置を基準にズームするため、カメラの範囲に差分を適用
    camera.left += preZoomX - postZoomX;
    camera.right += preZoomX - postZoomX;
    camera.top += preZoomY - postZoomY;
    camera.bottom += preZoomY - postZoomY;
  
    camera.updateProjectionMatrix();
    renderer.render(scene, camera);
  }
  function handleMouseDown(event) {
    isDragging = true;
    previousMousePosition = { x: event.clientX, y: event.clientY };
  }
  
  function handleMouseUp() {
    isDragging = false;
  }
  
  function handleMouseMove(event) {
    if (!isDragging) return;
  
    const dx = event.clientX - previousMousePosition.x;
    const dy = event.clientY - previousMousePosition.y;
  
    const scaleFactor = (camera.right - camera.left) / window.innerWidth;
  
    camera.left -= dx * scaleFactor;
    camera.right -= dx * scaleFactor;
    camera.top += dy * scaleFactor;
    camera.bottom += dy * scaleFactor;
  
    camera.updateProjectionMatrix();
    renderer.render(scene, camera);
  
    previousMousePosition = { x: event.clientX, y: event.clientY };
  }

  function init() {
    // シーンの作成
    scene = new THREE.Scene();
  
    // カメラの作成
    const aspectRatio = window.innerWidth / window.innerHeight;
    camera = new THREE.OrthographicCamera(-aspectRatio, aspectRatio, 1, -1, 1, 1000);
    camera.position.z = 1;
  
    // レンダラーの作成
    renderer = new THREE.WebGLRenderer();
    renderer.setSize(window.innerWidth, window.innerHeight);
  
    // シェーダーマテリアルの作成
    const shaderMaterial = new THREE.ShaderMaterial({
      vertexShader: vertexShader,
      fragmentShader: fragmentShader,
    });
  
    // メッシュの作成
    const geometry = new THREE.PlaneGeometry(2, 2);
    const mesh = new THREE.Mesh(geometry, shaderMaterial);
    scene.add(mesh);
  
    // レンダラーをコンテナに追加
    container.appendChild(renderer.domElement);

    // イベントリスナの追加
    container.addEventListener("wheel", handleWheel);
    container.addEventListener("mousedown", handleMouseDown);
    container.addEventListener("mouseup", handleMouseUp);
    container.addEventListener("mousemove", handleMouseMove);
  
    // 描画
    renderer.render(scene, camera);
  }
</script>

<svelte:head>
  <style>
    .container {
      width: 100%;
      height: 100%;
      overflow: hidden;
      position: absolute;
    }
  </style>
</svelte:head>

<div class="container" bind:this="{container}"></div>
