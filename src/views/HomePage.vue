<template>
  <div class="background">
    <div class="home-container">
      <h1 class="title">织夜工具箱</h1>

      <div class="button-group">
        <router-link to="chouka" class="btn chouka-btn">
          <span>✨ 抽卡模拟器 ✨</span>
        </router-link>

        <router-link to="fenxi" class="btn fenxi-btn">
          <span>📊 抽卡数据分析 📈</span>
        </router-link>

        <router-link to="kejin" class="btn kejin-btn">
          <span>🎁 我该不该氪？ 💰</span>
        </router-link>

        <button @click="handleComingSoon" :disabled="isComingSoonClicked" class="btn coming-soon-btn">
          <span>{{ comingSoonText }}</span>
        </button>
      </div>

      <div class="info-footer">
        <a href="https://github.com/Thisiseanxu/gacha-party" target="_blank" class="footer-link">
          <github-one theme="outline" size="20" />
          <span>开源地址</span>
        </a>

        <a href="https://qm.qq.com/cgi-bin/qm/qr?k=PD3VWuDfxO_hAVZQBreK1CjvWORTkNN2&jump_from=webapi&authKey=c4Sos3R4opf3VqerCwpPX+IOmwZUDm4hqkyT7qDGhta2fAhdUETlxFZ9wDrcRu1z"
          target="_blank" class="footer-link">
          <tencent-qq theme="outline" size="20" />
          <span>加入Q群</span>
        </a>

        <a v-if="deferredPrompt" @click="handleInstallClick" class="footer-link">
          <install theme="outline" size="20" />
          <span>安装应用</span>
        </a>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';
import { colors } from '@/styles/colors.js';
import { GithubOne, TencentQq, Install } from '@icon-park/vue-next';


// 创建一个 ref 保存 'beforeinstallprompt' 事件
const deferredPrompt = ref(null);
const captureInstallPrompt = (e) => {
  // 阻止浏览器默认的、自动弹出的安装提示
  e.preventDefault();
  // 保存事件对象，以便后续手动触发
  deferredPrompt.value = e;
  console.log('PWA 安装提示已被捕获，等待用户手动触发。');
};

// 设置一个监听器来捕获 'beforeinstallprompt' 事件
onMounted(() => {
  window.addEventListener('beforeinstallprompt', captureInstallPrompt);
});
onUnmounted(() => {
  window.removeEventListener('beforeinstallprompt', captureInstallPrompt);
});

const handleInstallClick = async () => {
  if (!deferredPrompt.value) {
    return;
  }
  // 调用保存的事件对象的 prompt() 方法，会弹出浏览器标准的安装窗口
  deferredPrompt.value.prompt();

  // 等待用户做出选择 (接受或拒绝)
  const { outcome } = await deferredPrompt.value.userChoice;
  console.log(`PWA 安装提示的用户选择: ${outcome}`);

  // 无论用户选择什么，这个事件都已经用过，无法再次使用。
  // 清空 ref，我们的安装按钮也会因此被 v-if 隐藏。
  deferredPrompt.value = null;
};

// --- 开发中按钮控制逻辑 ---
const originalComingSoonText = '🛠️ 伤害计算器 (即将推出)';
const comingSoonText = ref(originalComingSoonText);
const isComingSoonClicked = ref(false);

const handleComingSoon = () => {
  // 如果按钮已经被点击，则不执行任何操作
  if (isComingSoonClicked.value) return;

  // 更新文本并禁用按钮
  comingSoonText.value = '正在努力更新，不要戳我了！';
  isComingSoonClicked.value = true;

  // 3秒后恢复按钮
  setTimeout(() => {
    comingSoonText.value = originalComingSoonText;
    isComingSoonClicked.value = false;
  }, 1000);
};

const colorTextPrimary = colors.text.primary;
const colorTextHighlight = colors.text.highlight;
</script>

<style scoped>
.background {
  position: relative;
  min-height: 100vh;
  background-color: #000;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden;
}

.background::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-image: url('/images/homepage_bg.webp');
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  opacity: 0.2;
  z-index: 1;
}

.home-container {
  position: relative;
  z-index: 2;
  max-width: 600px;
  width: 100%;
  padding: 2rem;
  background-color: rgba(26, 27, 32, 0.8);
  border-radius: 12px;
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
}

.title {
  font-size: 3rem;
  font-weight: bold;
  color: v-bind(colorTextPrimary);
  margin-bottom: 2rem;
}

.button-group {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
  width: 100%;
}

.btn {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 1.2rem;
  border-radius: 8px;
  text-decoration: none;
  font-weight: 600;
  transition: all 0.3s ease;
  border: none;
  color: white;
  font-size: 1.2rem;
  cursor: pointer;
  /* 为所有按钮添加指针手势 */
}

.btn:hover {
  transform: translateY(-2px);
  filter: brightness(1.1);
}

.chouka-btn {
  background: linear-gradient(145deg, #8B5CF6, #6D28D9);
}

.fenxi-btn {
  background: linear-gradient(145deg, #F9A8D4, #EC4899);
}

.kejin-btn {
  background: linear-gradient(145deg, #10B981, #059669);
}

/* --- 开发中功能的按钮 --- */
.coming-soon-btn {
  background: linear-gradient(145deg, #6B7280, #4B5563);
  color: #D1D5DB;
}

/* 按钮被禁用时的样式 */
.coming-soon-btn:disabled {
  background: linear-gradient(145deg, #4B5563, #374151);
  color: #9CA3AF;
  cursor: not-allowed;
  /* 禁用时显示“不可用”光标 */
  transform: none;
  /* 禁用时移除悬浮效果 */
  filter: none;
}

.info-footer {
  display: flex;
  justify-content: center;
  gap: 2rem;
  margin-top: 2.5rem;
  padding-top: 1.5rem;
  border-top: 1px solid rgba(58, 59, 64, 0.5);
  width: 100%;
}

.footer-link {
  display: flex;
  align-items: center;
  gap: 0.8rem;
  color: v-bind(colorTextHighlight);
  text-decoration: none;
  font-size: 1rem;
  transition: all 0.3s ease;
}

.footer-link:hover {
  filter: brightness(1.2);
}
</style>
