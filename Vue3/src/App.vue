<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';

// 生成1000个图片数据
const totalItems = 1000;
const items = ref([]);

// 虚拟滚动相关配置
const containerRef = ref(null);
const itemHeight = 200; // 每个项目的固定高度
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
      url: `https://picsum.photos/400/300?random=${i}`,
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
          <img :src="item.url" :alt="item.alt" />
          <p>{{ item.alt }}</p>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.virtual-scroll-container {
  height: 600px;
  overflow-y: auto;
  border: 1px solid #ccc;
  position: relative;
  width: 100%;
  max-width: 800px;
  margin: 0 auto;
}

.virtual-scroll-content {
  position: relative;
  width: 100%;
}

.virtual-scroll-items {
  width: 100%;
}

.image-item {
  height: 200px;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 10px;
  border-bottom: 1px solid #eee;
  box-sizing: border-box;
}

.image-item img {
  max-width: 100%;
  max-height: 160px;
  object-fit: contain;
}

.image-item p {
  margin: 10px 0 0;
  font-size: 14px;
  color: #333;
}
</style>
