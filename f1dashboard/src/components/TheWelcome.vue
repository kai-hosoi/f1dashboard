<template>
  <canvas ref="canvasRef" width="800" height="600"></canvas>
  <div class="button-container">
    <button @click="startAnimation" :disabled="isAnimating || isLoading">
      {{ isLoading ? "データ取得中..." : isAnimating ? "アニメーション中..." : "アニメーション開始" }}
    </button>
    <select name="course-select" id="course-select">
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
import circuitPath from "@/assets/circuit.png";

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

  // circuitImage.src = circuitPath;
  circuitImage.onload = () => {
    console.log("画像読み込み完了");
    drawCircuit(); // 任意：背景に描く処理（必要なら）
  };
});


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
  ctx.value.clearRect(0, 0, 800, 600);
  ctx.value.drawImage(circuitImage, 0, 0, 800, 600);
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
  const xs = points.value.map((p) => p.x);
  const ys = points.value.map((p) => p.y);
  const minX = Math.min(...xs),
    maxX = Math.max(...xs);
  const minY = Math.min(...ys),
    maxY = Math.max(...ys);

  const scaleX = 800 / (maxX - minX);
  const scaleY = 600 / (maxY - minY);

  points.value = points.value.map((p) => ({
    x: (p.x - minX) * scaleX,
    y: 600 - (p.y - minY) * scaleY,
  }));
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
