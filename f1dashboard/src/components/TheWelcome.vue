<template>
  <canvas ref="canvasRef" width="800" height="600"></canvas>
  <div class="button-container">
    <button @click="startAnimation" :disabled="isAnimating || isLoading">
      {{ isLoading ? "データ取得中..." : isAnimating ? "アニメーション中..." : "アニメーション開始" }}
    </button>
    <select name="course-select" id="course-select" @change="changeCircuite">
      <option value="Monaco">Monaco</option>
      <option value="Spain">Spain</option>
      <option value="Canada">Canada</option>
    </select>
  </div>
  <div v-if="isLoading" class="spinner-overlay">
    <div class="spinner"></div>
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";
import Monacocircuit from "@/assets/Monacocircuit.png";
import Spaincircuit from "@/assets/Spaincircuit.png";
import Canadacircuit from "@/assets/Canadacircuit.png";

const canvasRef = ref(null);
const ctx = ref(null);
const points = ref([]);
let index = 0;
const isAnimating = ref(false);
const isLoading = ref(false);
let circuitImage = new Image();

onMounted(() => {
  const canvas = canvasRef.value;
  ctx.value = canvas.getContext("2d");

  circuitImage.src = Monacocircuit;
  circuitImage.onload = () => {
    console.log("画像読み込み完了");
    drawCircuit(); // 任意：背景に描く処理（必要なら）
  };
});

function changeCircuite(){
  const selectElement = document.getElementById("course-select");
  const selectedValue = selectElement.value;
  console.log("選択されたコース:", selectedValue);

  if(selectedValue === "Monaco"){
    circuitImage.src = Monacocircuit;
  }else if(selectedValue === "Spain"){
    circuitImage.src = Spaincircuit;
  }else if(selectedValue === "Canada"){
    circuitImage.src = Canadacircuit;
  }
  drawCircuit();
}


async function fetchData() {
  isLoading.value = true;

  const selectElement = document.getElementById("course-select");
  const selectedValue = selectElement.value;
  console.log("選択されたコース:", selectedValue);

  try {
    // ① session_key の取得
    const sessionRes = await fetch(
      `https://api.openf1.org/v1/sessions?country_name=${selectedValue}&year=2025`
    );

    if (!sessionRes.ok) {
      throw new Error(`Session API エラー: ${sessionRes.status}`);
    }

    const sessionData = await sessionRes.json();

    if (!sessionData.length) {
      throw new Error("指定された条件に一致するセッションが見つかりません。");
    }

    const sessionKey = sessionData[0].session_key;
    console.log("取得した session_key:", sessionKey);

    // ② 位置情報の取得
    const locationRes = await fetch(
      `https://api.openf1.org/v1/location?session_key=${sessionKey}&driver_number=81`
    );

    if (!locationRes.ok) {
      throw new Error(`Location API エラー: ${locationRes.status}`);
    }

    const locationData = await locationRes.json();
    console.log("取得したデータ:", locationData);

    points.value = locationData.map((p) => ({ x: p.x, y: p.y }));
    normalizePoints();
  } catch (error) {
    console.error("データの取得に失敗しました:", error);
    alert("データの取得に失敗しました。ページを再読み込みしてください。");
  } finally {
    isLoading.value = false;
  }
}


async function startAnimation() {
  console.log("アニメーション開始");
  await fetchData(); // 画像と無関係にAPIだけ実行
  if (points.value.length === 0) return;
  index = 0;
  isAnimating.value = true;
  animate();
}

// 背景画像を描画
function drawCircuit() {
  const canvasWidth = 800;
  const canvasHeight = 600;

  // コース線のバウンディングボックスを取得
  const xs = points.value.map(p => p.x);
  const ys = points.value.map(p => p.y);
  const minX = Math.min(...xs);
  const maxX = Math.max(...xs);
  const minY = Math.min(...ys);
  const maxY = Math.max(...ys);

  const trackWidth = maxX - minX;
  const trackHeight = maxY - minY;

  // 背景画像サイズ
  const imgWidth = 1200;
  const imgHeight = 750;

  // 背景画像をコース線に合わせてスケーリング
  const scaleX = trackWidth / imgWidth;
  const scaleY = trackHeight / imgHeight;
  const scale = Math.max(scaleX, scaleY); // 全体が覆えるようにスケーリング

  const drawWidth = imgWidth * scale;
  const drawHeight = imgHeight * scale;

  // 背景の左上座標（線のminX, minYに合わせて描画開始）
  const offsetX = minX - (drawWidth - trackWidth) / 2;
  const offsetY = minY - (drawHeight - trackHeight) / 2;

  // 背景画像を描画
  ctx.value.clearRect(0, 0, canvasWidth, canvasHeight);
  ctx.value.drawImage(circuitImage, offsetX, offsetY, drawWidth, drawHeight);
}

function animate() {
  drawCircuit(); 
  if (index > 0) {
    ctx.value.beginPath();
    ctx.value.moveTo(points.value[0].x, points.value[0].y);
    for (let i = 1; i <= index; i++) {
      ctx.value.lineTo(points.value[i].x, points.value[i].y);
    }
    ctx.value.strokeStyle = "blue";
    ctx.value.stroke();
  }

  const current = points.value[index];
  ctx.value.beginPath();
  ctx.value.arc(current.x, current.y, 4, 0, 2 * Math.PI);
  ctx.value.fillStyle = "orange";
  ctx.value.fill();

  index++;
  if (index % 100 === 0) {
    console.log(`現在のインデックス: ${index}, ポイント数: ${points.value.length}`);
  }

  if (index < points.value.length) {
    setTimeout(animate, 0.0001);
  } else {
    isAnimating.value = false;
  }
}

function normalizePoints() {
  const canvasWidth = 800;
  const canvasHeight = 600;

  const xs = points.value.map((p) => p.x);
  const ys = points.value.map((p) => p.y);
  const minX = Math.min(...xs), maxX = Math.max(...xs);
  const minY = Math.min(...ys), maxY = Math.max(...ys);

  const trackWidth = maxX - minX;
  const trackHeight = maxY - minY;

  // 背景画像（＝キャンバス）に合わせたスケーリング
  const scale = Math.min(canvasWidth / trackWidth, canvasHeight / trackHeight);

  // トラック中心（元データ）
  const centerX = (minX + maxX) / 2;
  const centerY = (minY + maxY) / 2;

  // 背景画像中心（キャンバス中心）
  const canvasCenterX = canvasWidth / 2;
  const canvasCenterY = canvasHeight / 2;

  // 調整した回転角（必要に応じて ± 調整）
  const angle = (-45 * Math.PI) / 180;

  // 正規化・回転・スケーリング・中央寄せ処理
  points.value = points.value.map((p) => {
    const x = (p.x - centerX);
    const y = (p.y - centerY);

    const rotatedX = x * Math.cos(angle) - y * Math.sin(angle);
    const rotatedY = x * Math.sin(angle) + y * Math.cos(angle);

    return {
      x: canvasCenterX + rotatedX * scale,
      y: canvasCenterY - rotatedY * scale, // Y反転
    };
  });
}




</script>

<style scoped>
canvas {
    border: 1px solid #ccc;
}
button {
  margin-top: 10px;
  padding: 8px 16px;
  font-size: 16px;
}

/* スピナー表示 */
.spinner-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(255, 255, 255, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
}

.spinner {
  width: 40px;
  height: 40px;
  border: 5px solid #ccc;
  border-top-color: #3e8ed0;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}
</style>
