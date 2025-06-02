<template>
  <canvas ref="canvasRef" width="800" height="600"></canvas>
  <div class="button-container">
    <button @click="StartAnimation" :disabled="isAnimating">
      Start Animation
    </button>
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


onMounted(async () => {
    const canvas = canvasRef.value;
    ctx.value = canvas.getContext("2d");

    circuitImage.src = circuitPath; // 画像パス（public/img/circuit.png などに配置）

    circuitImage.onload = async () => {
        isLoading.value = true; // ← API前にtrue
        try {
          const res = await fetch(
            "https://api.openf1.org/v1/location?session_key=latest&driver_number=81"
          );

          if (!res.ok) {
            // レスポンスがエラー（例: 404, 500）
            throw new Error(`HTTPエラー: ${res.status}`);
          }

          const data = await res.json();
          console.log("取得したデータ:", data);
          points.value = data.map((p) => ({ x: p.x, y: p.y }));

          normalizePoints();
        } catch (error) {
          console.error("データの取得に失敗しました:", error);
          alert("位置情報の取得に失敗しました。ページを再読み込みしてください。");
        } finally {
          isLoading.value = false;
        }
    };
});

  function normalizePoints() {
      const xs = points.map((p) => p.x);
      const ys = points.map((p) => p.y);
      const minX = Math.min(...xs),
          maxX = Math.max(...xs);
      const minY = Math.min(...ys),
          maxY = Math.max(...ys);

      const scaleX = 800 / (maxX - minX);
      const scaleY = 600 / (maxY - minY);

      points.value = points.map((p) => ({
          x: (p.x - minX) * scaleX,
          y: 600 - (p.y - minY) * scaleY, // Y反転
      }));
  }

  function animate() {
    // 背景画像を描画
    ctx.clearRect(0, 0, 800, 600);
    ctx.drawImage(circuitImage, 0, 0, 800, 600);

    if (index > 0) {
        ctx.beginPath();
        ctx.moveTo(points[0].x, points[0].y);
        for (let i = 1; i <= index; i++) {
            ctx.lineTo(points[i].x, points[i].y);
        }
        ctx.strokeStyle = "blue";
        ctx.stroke();
    }

    const current = points[index];
    ctx.beginPath();
    ctx.arc(current.x, current.y, 4, 0, 2 * Math.PI);
    ctx.fillStyle = "orange";
    ctx.fill();

    index++;
    if (index % 100 === 0) {
        // 100ポイントごとにログ出力
        console.log(`現在のインデックス: ${index}, ポイント数: ${points.length}`);
    }

    if (index < points.length) {
        setTimeout(animate,0.001);
    }else{
      isAnimating.value = false; // 終了時ボタンを再度使えるように
    }
  }
  function startAnimation() {
    if (points.length === 0) return;
    index = 0;
    isAnimating.value = true;
    animate();
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
  width: 800px;
  height: 600px;
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
