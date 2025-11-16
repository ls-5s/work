<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';

// 生成1000个图片数据
const totalItems = 1000;
const items = ref([]);

// 虚拟滚动相关配置
const containerRef = ref(null);
const itemHeight = 240; // 增加项目高度，提供更好的视觉体验
const visibleCount = ref(10); // 可见的项目数量
const bufferCount = 2; // 缓冲区项目数量
const scrollTop = ref(0); // 滚动位置

// 计算开始索引和结束索引
const startIndex = computed(() => {
  const start = Math.floor(scrollTop.value / itemHeight);
  return Math.max(0, start - bufferCount);
});

const endIndex = computed(() => {
  const end = startIndex.value + visibleCount.value + bufferCount * 2;
  return Math.min(totalItems - 1, end);
});

// 计算可见项目
const visibleItems = computed(() => {
  return items.value.slice(startIndex.value, endIndex.value + 1);
});

// 计算偏移量
const offsetY = computed(() => {
  return startIndex.value * itemHeight;
});

// 计算总高度
const totalHeight = computed(() => {
  return totalItems * itemHeight;
});

// 处理滚动事件
const handleScroll = () => {
  if (containerRef.value) {
    scrollTop.value = containerRef.value.scrollTop;
  }
};

// 组件挂载时初始化数据和事件监听
onMounted(() => {
  // 生成模拟数据
  for (let i = 0; i < totalItems; i++) {
    // 使用picsum.photos提供的随机图片
    items.value.push({
      id: i,
      url: `https://picsum.photos/600/400?random=${i}`,
      alt: `图片 ${i + 1}`
    });
  }

  // 计算可见项目数量
  if (containerRef.value) {
    visibleCount.value = Math.ceil(containerRef.value.clientHeight / itemHeight) + 1;
  }

  // 添加滚动事件监听
  containerRef.value?.addEventListener('scroll', handleScroll);
});

// 组件卸载时移除事件监听
onUnmounted(() => {
  containerRef.value?.removeEventListener('scroll', handleScroll);
});
</script>

<template>
  <div class="app-container">
    <header class="header">
      <h1 class="title">虚拟滚动图片列表</h1>
      <p class="subtitle">流畅滚动1000张图片的高性能实现</p>
    </header>

    <main class="main-content">
      <div class="virtual-scroll-wrapper">
        <div class="virtual-scroll-container" ref="containerRef">
          <div
            class="virtual-scroll-content"
            :style="{ height: totalHeight + 'px' }"
          >
            <div
              class="virtual-scroll-items"
              :style="{ transform: `translateY(${offsetY}px)` }"
            >
              <div
                v-for="item in visibleItems"
                :key="item.id"
                class="image-item"
              >
                <div class="image-wrapper">
                  <img
                    :src="item.url"
                    :alt="item.alt"
                    class="image"
                    loading="lazy"
                  />
                </div>
                <div class="item-info">
                  <h3 class="item-title">{{ item.alt }}</h3>
                  <p class="item-id">ID: {{ item.id }}</p>
                </div>
              </div>
            </div>
          </div>
        </div>

        <div class="stats-panel">
          <div class="stat-item">
            <span class="stat-label">总图片数:</span>
            <span class="stat-value">{{ totalItems }}</span>
          </div>
          <div class="stat-item">
            <span class="stat-label">渲染DOM数:</span>
            <span class="stat-value">{{ visibleItems.length }}</span>
          </div>
          <div class="stat-item">
            <span class="stat-label">当前位置:</span>
            <span class="stat-value">{{ startIndex + 1 }} - {{ endIndex + 1 }}</span>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<style>
/* 全局样式重置 */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  background-color: #f5f7fa;
  color: #333;
  line-height: 1.6;
}
</style>

<style scoped>
.app-container {
  min-height: 100vh;
  padding: 20px;
}

.header {
  text-align: center;
  margin-bottom: 40px;
  padding: 30px 0;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border-radius: 12px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.1);
}

.title {
  font-size: 2.5rem;
  font-weight: 700;
  margin-bottom: 10px;
  letter-spacing: -0.5px;
}

.subtitle {
  font-size: 1.1rem;
  opacity: 0.9;
  font-weight: 300;
}

.main-content {
  max-width: 1200px;
  margin: 0 auto;
}

.virtual-scroll-wrapper {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.virtual-scroll-container {
  height: 700px;
  overflow-y: auto;
  background: white;
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
  position: relative;
  scroll-behavior: smooth;
}

/* 自定义滚动条 */
.virtual-scroll-container::-webkit-scrollbar {
  width: 8px;
}

.virtual-scroll-container::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 4px;
}

.virtual-scroll-container::-webkit-scrollbar-thumb {
  background: #c1c1c1;
  border-radius: 4px;
  transition: background 0.3s;
}

.virtual-scroll-container::-webkit-scrollbar-thumb:hover {
  background: #a8a8a8;
}

.virtual-scroll-content {
  position: relative;
  width: 100%;
}

.virtual-scroll-items {
  width: 100%;
}

.image-item {
  height: 240px;
  padding: 16px;
  border-bottom: 1px solid #f0f0f0;
  display: flex;
  align-items: center;
  gap: 20px;
  transition: background-color 0.2s ease, transform 0.1s ease;
  background: white;
}

.image-item:hover {
  background-color: #fafafa;
  transform: translateY(-1px);
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.05);
}

.image-wrapper {
  flex-shrink: 0;
  width: 180px;
  height: 140px;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
}

.image {
  width: 100%;
  height: 100%;
  object-fit: cover;
  transition: transform 0.3s ease;
}

.image-item:hover .image {
  transform: scale(1.05);
}

.item-info {
  flex: 1;
  min-width: 0;
}

.item-title {
  font-size: 1.2rem;
  font-weight: 600;
  color: #2c3e50;
  margin-bottom: 8px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.item-id {
  font-size: 0.9rem;
  color: #7f8c8d;
  font-weight: 400;
}

.stats-panel {
  background: white;
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
  display: flex;
  justify-content: space-around;
  flex-wrap: wrap;
  gap: 20px;
}

.stat-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 5px;
}

.stat-label {
  font-size: 0.9rem;
  color: #7f8c8d;
  font-weight: 500;
}

.stat-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #667eea;
}

/* 响应式设计 */
@media (max-width: 768px) {
  .app-container {
    padding: 10px;
  }

  .header {
    padding: 20px 0;
  }

  .title {
    font-size: 2rem;
  }

  .virtual-scroll-container {
    height: 600px;
  }

  .image-item {
    flex-direction: column;
    height: auto;
    min-height: 280px;
    text-align: center;
    padding: 20px;
  }

  .image-wrapper {
    width: 100%;
    max-width: 300px;
    height: 180px;
  }

  .stats-panel {
    flex-direction: column;
    align-items: center;
  }
}
</style>
