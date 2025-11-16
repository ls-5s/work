<script setup>
import { ref, computed, onMounted } from 'vue';

// 商品数据状态管理
const productData = ref({
  title: '',
  highlights: [''],
  specifications: [
    { name: '', value: '' }
  ],
  description: ''
});

// 图片状态管理
const uploadedImage = ref(null);
const mainImages = ref([]);
const detailImages = ref([]);
const isGenerating = ref(false);
const generationStep = ref(0);
const generationProgress = ref(0);

// 错误处理状态
const errors = ref({
  image: '',
  title: '',
  api: ''
});



// 修改后
const DASHSCOPE_API_KEY = import.meta.env.VITE_DASHSCOPE_API_KEY ||'';

// console.log('DASHSCOPE_API_KEY:', DASHSCOPE_API_KEY);
// 阿里云DASHSCOPE API配置（使用相对路径以便通过代理）
const dashscopeConfig = {
  apiKey: DASHSCOPE_API_KEY,
  apiUrl: '/api/v1/services/aigc/text2image/image-synthesis',
  taskPollUrlPrefix: '/api/v1/tasks'
};

// 模型配置
const MODEL_NAME = 'wan2.5-t2i-preview';

// 存储任务ID和参数的临时内存存储
const taskCreationTimes = new Map();
const taskParameters = new Map();



// 计算属性：检查是否可以开始生成
const canGenerate = computed(() => {
  return uploadedImage.value &&
         productData.value.title.trim() &&
         !errors.value.image &&
         !errors.value.title
});

// 计算属性：生成进度百分比
const progressPercent = computed(() => {
  return generationProgress.value * 100;
});

// 计算属性：当前生成阶段的文本描述
const currentStepText = computed(() => {
  const steps = [
    '正在分析商品信息...',
    '正在生成主图...',
    '正在生成详情图...',
    '生成完成！'
  ];
  return steps[generationStep.value] || '准备中...';
});

// 处理图片上传
const handleImageUpload = (event) => {
  const file = event.target.files[0];
  if (!file) return;

  // 检查文件类型
  if (!file.type.match('image.*')) {
    errors.value.image = '请上传有效的图片文件';
    return;
  }

  // 检查图片尺寸（这里简化处理，实际应该在服务端验证）
  const reader = new FileReader();
  reader.onload = (e) => {
    const img = new Image();
    img.onload = () => {
      if (img.width < 800 || img.height < 800) {
        errors.value.image = '建议上传至少800×800px的白底图以保证质量';
      } else {
        errors.value.image = '';
        uploadedImage.value = e.target.result;
      }
    };
    img.src = e.target.result;
  };
  reader.readAsDataURL(file);
};

// 添加卖点
const addHighlight = () => {
  productData.value.highlights.push('');
};

// 移除卖点
const removeHighlight = (index) => {
  if (productData.value.highlights.length > 1) {
    productData.value.highlights.splice(index, 1);
  }
};

// 更新卖点
const updateHighlight = (index, value) => {
  productData.value.highlights[index] = value;
};

// 添加规格参数
const addSpecification = () => {
  productData.value.specifications.push({ name: '', value: '' });
};

// 移除规格参数
const removeSpecification = (index) => {
  if (productData.value.specifications.length > 1) {
    productData.value.specifications.splice(index, 1);
  }
};

// 验证表单
const validateForm = () => {
  let isValid = true;

  if (!productData.value.title.trim()) {
    errors.value.title = '请输入商品标题';
    isValid = false;
  } else {
    errors.value.title = '';
  }

  if (!uploadedImage.value) {
    errors.value.image = '请上传商品白底图';
    isValid = false;
  }


  return isValid;
};

// 创建异步图像生成任务
const createImageGenerationTask = async (prompt, size = "1024*1024") => {
  try {
    const headers = {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${dashscopeConfig.apiKey}`,
      'X-DashScope-Async': 'enable'
    };

    const requestBody = {
      model: MODEL_NAME,
      input: {
        prompt: prompt
      },
      parameters: {
        size: size,
        n: 1
      }
    };

    const response = await fetch(dashscopeConfig.apiUrl, {
      method: 'POST',
      headers: headers,
      body: JSON.stringify(requestBody)
    });

    if (!response.ok) {
      const errorData = await response.json().catch(() => ({}));
      throw new Error(`API调用失败: ${response.status} ${errorData.message || response.statusText}`);
    }

    const data = await response.json();

    if (data.output?.task_id) {
      // 存储任务信息
      taskCreationTimes.set(data.output.task_id, {
        createTime: Date.now(),
        retryCount: 0,
        maxRetry: 5
      });
      taskParameters.set(data.output.task_id, {
        prompt: prompt,
        size: size
      });

      console.log(`创建图像生成任务: ${data.output.task_id}`);
      return data.output.task_id;
    } else {
      throw new Error('API返回不包含task_id');
    }
  } catch (error) {
    console.error('创建图像生成任务失败:', error);
    // 提供更具体的错误信息给用户
    if (error.message.includes('401')) {
      errors.value.api = 'API Key无效或已过期，请检查配置';
    } else if (error.message.includes('Network')) {
      errors.value.api = '网络连接失败，请检查您的网络设置';
    } else {
      errors.value.api = `创建图像生成任务失败: ${error.message}`;
    }
    throw error;
  }
};

// 查询异步任务状态
const pollTaskStatus = async (taskId) => {
  try {
    const headers = {
      'Authorization': `Bearer ${dashscopeConfig.apiKey}`
    };

    const response = await fetch(`${dashscopeConfig.taskPollUrlPrefix}/${taskId}`, {
      method: 'GET',
      headers: headers
    });

    if (!response.ok) {
      throw new Error(`查询任务状态失败: ${response.status} ${response.statusText}`);
    }

    return await response.json();
  } catch (error) {
    console.error('查询任务状态失败:', error);
    const taskInfo = taskCreationTimes.get(taskId);
    if (taskInfo) {
      taskInfo.retryCount++;
      if (taskInfo.retryCount > taskInfo.maxRetry) {
        throw new Error('任务查询超过最大重试次数');
      }
    }
    throw error;
  }
};

// 等待异步任务完成
const waitForTaskCompletion = async (taskId, checkInterval = 3000) => {
  return new Promise((resolve, reject) => {
    const checkStatus = async () => {
      try {
        const taskStatus = await pollTaskStatus(taskId);

        if (taskStatus.output && taskStatus.output.task_status === 'SUCCEEDED') {
          resolve(taskStatus.output.results);
        } else if (taskStatus.output && taskStatus.output.task_status === 'FAILED') {
          reject(new Error(`任务失败: ${taskStatus.output.failure_details || '未知错误'}`));
        } else {
          // 任务仍在进行中，继续轮询
          setTimeout(checkStatus, checkInterval);
        }
      } catch (error) {
        reject(error);
      }
    };

    // 开始轮询
    setTimeout(checkStatus, checkInterval);
  });
};

// 生成图片（使用阿里云DASHSCOPE API，异步模式）
const generateImages = async () => {
  if (!validateForm()) return;

  isGenerating.value = true;
  generationStep.value = 0;
  generationProgress.value = 0;
  mainImages.value = [];
  detailImages.value = [];
  errors.value.api = '';

  try {
    // 构建商品信息
    const productInfo = `
商品标题: ${productData.value.title}
商品卖点: ${productData.value.highlights.filter(h => h.trim()).join(', ')}
规格参数: ${productData.value.specifications
      .filter(s => s.name.trim() && s.value.trim())
      .map(s => `${s.name}: ${s.value}`).join(', ')}
商品描述: ${productData.value.description}
    `.trim();

    // 第一阶段：分析商品信息
    generationStep.value = 0;
    generationProgress.value = 0.1;

    // 构建主图提示词数组（3张不同风格）
    const mainImagePrompts = [
      `${productInfo}，高清商品白底图转为专业电商主图，明亮清晰的白色背景，商品居中展示，细节丰富，专业商业摄影风格，800×800px`,
      `${productInfo}，高清商品白底图转为现代简约风格主图，柔和自然光线，极简美学构图，突出商品质感，800×800px`,
      `${productInfo}，高清商品白底图转为营销风格主图，突出商品卖点，适当添加简约装饰元素，吸引眼球的色彩搭配，800×800px`
    ];

    // 第二阶段：生成主图
    generationStep.value = 1;
    generationProgress.value = 0.2;

    console.log('开始生成主图...');

    // 创建3个主图生成任务
    const mainImageTaskIds = [];
    for (const prompt of mainImagePrompts) {
      const taskId = await createImageGenerationTask(prompt, "800*800");
      mainImageTaskIds.push(taskId);
    }

    // 更新进度
    generationProgress.value = 0.3;

    // 等待所有主图任务完成
    const mainImageResults = [];
    for (let i = 0; i < mainImageTaskIds.length; i++) {
      try {
        const results = await waitForTaskCompletion(mainImageTaskIds[i]);
        if (results && results.length > 0 && results[0].url) {
          mainImageResults.push(results[0].url);
        }
        // 更新进度
        generationProgress.value = 0.3 + (i + 1) * 0.1; // 0.3-0.6
      } catch (error) {
        console.error(`主图生成失败: ${error.message}`);
        // 失败时使用占位图
        mainImageResults.push(`https://picsum.photos/seed/${Date.now() + i}/800/800`);
      }
    }

    mainImages.value = mainImageResults;

    // 第三阶段：生成详情图
    generationStep.value = 2;
    generationProgress.value = 0.6;

    console.log('开始生成详情图...');

    // 构建详情图提示词数组（5张不同角度/卖点）
    const detailImagePrompts = [
      `${productInfo}，商品细节特写详情图，展示商品材质和工艺细节，高清清晰，专业光影，宽度750px`,
      `${productInfo}，商品功能展示详情图，突出商品主要功能和使用方式，场景化展示，宽度750px`,
      `${productInfo}，商品规格参数可视化详情图，清晰展示商品尺寸、材质等关键参数，信息图表风格，宽度750px`,
      `${productInfo}，商品使用场景详情图，展示商品在实际使用环境中的效果，提升购买欲望，宽度750px`,
      `${productInfo}，商品优势对比详情图，突出商品与同类产品的核心优势，简洁明了的对比设计，宽度750px`
    ];

    // 创建5个详情图生成任务
    const detailImageTaskIds = [];
    for (const prompt of detailImagePrompts) {
      const taskId = await createImageGenerationTask(prompt, "750*1000");
      detailImageTaskIds.push(taskId);
    }

    // 更新进度
    generationProgress.value = 0.7;

    // 等待所有详情图任务完成
    const detailImageResults = [];
    for (let i = 0; i < detailImageTaskIds.length; i++) {
      try {
        const results = await waitForTaskCompletion(detailImageTaskIds[i]);
        if (results && results.length > 0 && results[0].url) {
          detailImageResults.push(results[0].url);
        }
        // 更新进度
        generationProgress.value = 0.7 + (i + 1) * 0.06; // 0.7-1.0
      } catch (error) {
        console.error(`详情图生成失败: ${error.message}`);
        // 失败时使用占位图
        detailImageResults.push(`https://picsum.photos/seed/${Date.now() + 10 + i}/750/1000`);
      }
    }

    detailImages.value = detailImageResults;

    // 完成生成
    generationStep.value = 3;
    generationProgress.value = 1;

    console.log('图片生成完成！');

    setTimeout(() => {
      isGenerating.value = false;
    }, 500);

  } catch (error) {
    isGenerating.value = false;
    errors.value.api = `生成过程失败: ${error.message}`;
    console.error('生成过程失败:', error);
  }
};

// 下载图片
const downloadImage = (imageUrl, filename) => {
  const link = document.createElement('a');
  link.href = imageUrl;
  link.download = filename;
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
};

// 下载所有图片
const downloadAllImages = () => {
  // 下载主图
  mainImages.value.forEach((img, index) => {
    downloadImage(img, `主图_${index + 1}.png`);
  });

  // 下载详情图
  detailImages.value.forEach((img, index) => {
    downloadImage(img, `详情图_${index + 1}.png`);
  });
};


</script>

<template>
  <div class="app-container">
    <header class="app-header">
      <h1>商品图像智能生成系统</h1>
      <p class="subtitle">上传商品白底图和描述信息，自动生成电商平台所需的高质量主图和详情图</p>
    </header>

    <main class="app-main">
      <!-- 输入区域 -->
      <section class="input-section" v-if="!isGenerating && mainImages.length === 0">
        <div class="content-wrapper">
          <!-- 左侧：图片上传 -->
          <div class="image-upload-section">
            <h2 class="section-title">上传商品白底图</h2>
            <div class="upload-container">
              <input
                type="file"
                id="image-upload"
                accept="image/*"
                @change="handleImageUpload"
                class="file-input"
              />
              <label for="image-upload" class="upload-label">
                <div v-if="!uploadedImage" class="upload-placeholder">
                  <div class="upload-icon">+</div>
                  <p>点击上传或拖拽文件至此处</p>
                  <p class="upload-hint">建议上传800×800px以上的白底图</p>
                </div>
                <img v-else :src="uploadedImage" alt="已上传的商品图" class="preview-image" />
              </label>
              <p v-if="errors.image" class="error-message">{{ errors.image }}</p>
            </div>
          </div>

          <!-- 右侧：商品信息 -->
          <div class="product-info-section">
            <h2 class="section-title">商品信息</h2>

            <!-- 商品标题 -->
            <div class="form-group">
              <label for="product-title">商品标题 <span class="required">*</span></label>
              <input
                type="text"
                id="product-title"
                v-model="productData.title"
                placeholder="请输入商品标题（如：2023新款夏季连衣裙女士碎花长裙）"
                class="form-input"
              />
              <p v-if="errors.title" class="error-message">{{ errors.title }}</p>
            </div>

            <!-- 商品卖点 -->
            <div class="form-group">
              <div class="label-with-add">
                <label>商品卖点</label>
                <button type="button" @click="addHighlight" class="add-btn">+ 添加卖点</button>
              </div>
              <div v-for="(highlight, index) in productData.highlights" :key="index" class="highlight-input-group">
                <input
                  type="text"
                  v-model="productData.highlights[index]"
                  :placeholder="`卖点 ${index + 1}`"
                  class="form-input"
                />
                <button
                  type="button"
                  @click="removeHighlight(index)"
                  class="remove-btn"
                  :disabled="productData.highlights.length <= 1"
                >×</button>
              </div>
            </div>

            <!-- 规格参数 -->
            <div class="form-group">
              <div class="label-with-add">
                <label>规格参数</label>
                <button type="button" @click="addSpecification" class="add-btn">+ 添加参数</button>
              </div>
              <div v-for="(spec, index) in productData.specifications" :key="index" class="specification-row">
                <input
                  type="text"
                  v-model="spec.name"
                  placeholder="参数名称（如：材质）"
                  class="form-input spec-input"
                />
                <input
                  type="text"
                  v-model="spec.value"
                  placeholder="参数值（如：100%纯棉）"
                  class="form-input spec-input"
                />
                <button
                  type="button"
                  @click="removeSpecification(index)"
                  class="remove-btn"
                  :disabled="productData.specifications.length <= 1"
                >×</button>
              </div>
            </div>

            <!-- 商品描述 -->
            <div class="form-group">
              <label for="product-description">商品描述</label>
              <textarea
                id="product-description"
                v-model="productData.description"
                placeholder="请输入详细的商品描述信息..."
                class="form-textarea"
                rows="4"
              ></textarea>
            </div>
          </div>
        </div>

        <!-- API错误信息 -->
        <p v-if="errors.api" class="api-error-message">{{ errors.api }}</p>

        <!-- API Key配置状态提示 -->


        <!-- 生成按钮 -->
        <div class="generate-button-container">
          <button
            @click="generateImages"
            :disabled="!canGenerate"
            class="generate-button"
          >
            生成商品图像
          </button>
        </div>
      </section>

      <!-- 生成进度区域 -->
      <section class="generation-section" v-if="isGenerating">
        <div class="progress-container">
          <h2 class="section-title">{{ currentStepText }}</h2>
          <div class="progress-bar">
            <div class="progress-fill" :style="{ width: progressPercent + '%' }"></div>
          </div>
          <p class="progress-percent">{{ Math.round(progressPercent) }}%</p>
          <p v-if="errors.api" class="api-error-message">{{ errors.api }}</p>
        </div>
      </section>

      <!-- 结果展示区域 -->
      <section class="results-section" v-if="mainImages.length > 0">
        <!-- 主图结果 -->
        <div class="result-group">
          <div class="result-header">
            <h2 class="section-title">生成的主图（800×800px）</h2>
            <p class="result-description">以下是根据您的输入生成的3张不同风格主图，可用于电商平台展示</p>
          </div>
          <div class="images-grid main-images-grid">
            <div v-for="(image, index) in mainImages" :key="index" class="image-item">
              <img :src="image" :alt="`主图 ${index + 1}`" class="result-image" />
              <div class="image-actions">
                <button @click="downloadImage(image, `主图_${index + 1}.png`)" class="download-btn">
                  下载
                </button>
              </div>
            </div>
          </div>
        </div>

        <!-- 详情图结果 -->
        <div class="result-group">
          <div class="result-header">
            <h2 class="section-title">生成的详情图（宽度750px）</h2>
            <p class="result-description">以下是根据您的输入生成的5张详情图，展示商品的不同角度和卖点</p>
          </div>
          <div class="images-list detail-images-list">
            <div v-for="(image, index) in detailImages" :key="index" class="detail-image-item">
              <img :src="image" :alt="`详情图 ${index + 1}`" class="detail-image" />
              <div class="image-actions">
                <button @click="downloadImage(image, `详情图_${index + 1}.png`)" class="download-btn">
                  下载
                </button>
              </div>
            </div>
          </div>
        </div>

        <!-- 批量下载按钮 -->
        <div class="download-all-container">
          <button @click="downloadAllImages" class="download-all-btn">
            下载所有图片
          </button>
        </div>

        <!-- 重新生成按钮 -->
        <div class="regenerate-container">
          <button @click="mainImages = []" class="regenerate-btn">
            重新生成
          </button>
        </div>
      </section>
    </main>

    <footer class="app-footer">
      <p>© 2023 商品图像智能生成系统 - 基于AI技术的电商图像自动生成解决方案</p>
    </footer>
  </div>
</template>

<style>
/* 全局样式 */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  line-height: 1.6;
  color: #333;
  background-color: #f5f7fa;
}
</style>

<style scoped>
/* 应用容器 */
.app-container {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

/* 页头 */
.app-header {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 2rem 0;
  text-align: center;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
}

.app-header h1 {
  font-size: 2.5rem;
  margin-bottom: 0.5rem;
  font-weight: 700;
}

.subtitle {
  font-size: 1.1rem;
  opacity: 0.9;
  max-width: 800px;
  margin: 0 auto;
}

/* 主内容区域 */
.app-main {
  flex: 1;
  padding: 2rem;
  max-width: 1400px;
  margin: 0 auto;
  width: 100%;
}

/* 输入区域 */
.input-section {
  background: white;
  border-radius: 12px;
  padding: 2rem;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
}

.content-wrapper {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 2rem;
  margin-bottom: 2rem;
}

/* 区块标题 */
.section-title {
  font-size: 1.5rem;
  font-weight: 600;
  color: #2c3e50;
  margin-bottom: 1.5rem;
  border-bottom: 3px solid #667eea;
  padding-bottom: 0.5rem;
  display: inline-block;
}

/* 图片上传区域 */
.image-upload-section {
  display: flex;
  flex-direction: column;
}

.upload-container {
  position: relative;
  border: 2px dashed #ddd;
  border-radius: 8px;
  overflow: hidden;
  transition: all 0.3s ease;
  background-color: #fafafa;
}

.upload-container:hover {
  border-color: #667eea;
  background-color: #f0f3ff;
}

.file-input {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  opacity: 0;
  cursor: pointer;
  z-index: 2;
}

.upload-label {
  display: block;
  width: 100%;
  min-height: 300px;
  cursor: pointer;
  position: relative;
}

.upload-placeholder {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  min-height: 300px;
  text-align: center;
  padding: 2rem;
}

.upload-icon {
  font-size: 3rem;
  color: #667eea;
  margin-bottom: 1rem;
  opacity: 0.7;
}

.upload-placeholder p {
  margin-bottom: 0.5rem;
  color: #666;
}

.upload-hint {
  font-size: 0.9rem;
  color: #999;
}

.preview-image {
  width: 100%;
  height: auto;
  display: block;
  object-fit: contain;
  max-height: 400px;
  margin: 0 auto;
}

/* 商品信息区域 */
.product-info-section {
  display: flex;
  flex-direction: column;
}

/* 表单样式 */
.form-group {
  margin-bottom: 1.5rem;
}

.form-group label {
  display: block;
  font-weight: 500;
  margin-bottom: 0.5rem;
  color: #333;
}

.required {
  color: #e74c3c;
}

.label-with-add {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.5rem;
}

.form-input,
.form-textarea {
  width: 100%;
  padding: 0.75rem 1rem;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 1rem;
  transition: border-color 0.3s ease;
}

.form-input:focus,
.form-textarea:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.form-textarea {
  resize: vertical;
  min-height: 100px;
}

.highlight-input-group,
.specification-row {
  display: flex;
  align-items: center;
  margin-bottom: 0.75rem;
}

.highlight-input-group .form-input {
  flex: 1;
  margin-right: 0.5rem;
}

.specification-row .spec-input {
  flex: 1;
  margin-right: 0.5rem;
}

.add-btn,
.remove-btn {
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.9rem;
  transition: all 0.3s ease;
}

.add-btn {
  background-color: #667eea;
  color: white;
  font-weight: 500;
}

.add-btn:hover {
  background-color: #5a67d8;
  transform: translateY(-1px);
}

.remove-btn {
  background-color: #f56565;
  color: white;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.2rem;
  padding: 0;
}

.remove-btn:hover:not(:disabled) {
  background-color: #e53e3e;
}

.remove-btn:disabled {
  background-color: #cbd5e0;
  cursor: not-allowed;
}

/* 错误消息样式 */
.error-message {
  color: #e53e3e;
  font-size: 0.875rem;
  margin-top: 0.5rem;
}

.api-error-message {
  background-color: #fed7d7;
  color: #c53030;
  padding: 1rem;
  border-radius: 6px;
  margin-top: 1rem;
  text-align: center;
  border-left: 4px solid #c53030;
}

.config-warning {
  background-color: #feebc8;
  color: #c05621;
  padding: 1rem;
  border-radius: 6px;
  margin-top: 1rem;
  text-align: center;
  border-left: 4px solid #c05621;
}

/* 生成按钮样式 */
.generate-button-container {
  text-align: center;
  margin-top: 2rem;
}

.generate-button {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  padding: 1rem 3rem;
  border-radius: 30px;
  font-size: 1.25rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 4px 15px rgba(102, 126, 234, 0.4);
}

.generate-button:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(102, 126, 234, 0.5);
}

.generate-button:disabled {
  background: #a0aec0;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

/* 生成进度区域样式 */
.generation-section {
  background: white;
  border-radius: 12px;
  padding: 3rem;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
  text-align: center;
}

.progress-container {
  max-width: 600px;
  margin: 0 auto;
}

.progress-bar {
  width: 100%;
  height: 10px;
  background-color: #e2e8f0;
  border-radius: 5px;
  overflow: hidden;
  margin: 2rem 0;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #667eea, #764ba2);
  transition: width 0.3s ease;
  border-radius: 5px;
}

.progress-percent {
  font-size: 2rem;
  font-weight: 700;
  color: #667eea;
  margin-bottom: 1rem;
}

/* 结果展示区域样式 */
.results-section {
  background: white;
  border-radius: 12px;
  padding: 2rem;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
}

.result-group {
  margin-bottom: 3rem;
}

.result-header {
  text-align: center;
  margin-bottom: 2rem;
}

.result-description {
  color: #64748b;
  font-size: 1rem;
  max-width: 800px;
  margin: 0 auto;
}

/* 主图网格布局 */
.images-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 2rem;
  margin-top: 2rem;
}

.main-images-grid {
  grid-template-columns: repeat(3, 1fr);
}

.image-item {
  position: relative;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
  background-color: #f8fafc;
}

.image-item:hover {
  transform: translateY(-5px);
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.15);
}

.result-image {
  width: 100%;
  height: auto;
  display: block;
  object-fit: contain;
  max-height: 300px;
  margin: 0 auto;
}

/* 详情图列表布局 */
.images-list {
  display: flex;
  flex-direction: column;
  gap: 2rem;
  margin-top: 2rem;
}

.detail-image-item {
  position: relative;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
  background-color: #f8fafc;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.detail-image-item:hover {
  transform: translateY(-5px);
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.15);
}

.detail-image {
  width: 100%;
  height: auto;
  display: block;
  object-fit: contain;
  max-height: 600px;
}

/* 图片操作按钮 */
.image-actions {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  background: linear-gradient(transparent, rgba(0, 0, 0, 0.7));
  padding: 1rem;
  display: flex;
  justify-content: center;
  opacity: 0;
  transition: opacity 0.3s ease;
}

.image-item:hover .image-actions,
.detail-image-item:hover .image-actions {
  opacity: 1;
}

.download-btn {
  background-color: white;
  color: #667eea;
  border: none;
  padding: 0.5rem 1.5rem;
  border-radius: 4px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
}

.download-btn:hover {
  background-color: #667eea;
  color: white;
}

/* 批量操作按钮 */
.download-all-container,
.regenerate-container {
  text-align: center;
  margin-top: 2rem;
}

.download-all-btn,
.regenerate-btn {
  padding: 0.75rem 2rem;
  border: none;
  border-radius: 6px;
  font-size: 1rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
}

.download-all-btn {
  background-color: #38a169;
  color: white;
  margin-right: 1rem;
}

.download-all-btn:hover {
  background-color: #2f855a;
  transform: translateY(-1px);
}

.regenerate-btn {
  background-color: #ed8936;
  color: white;
}

.regenerate-btn:hover {
  background-color: #dd6b20;
  transform: translateY(-1px);
}

/* 页脚样式 */
.app-footer {
  background-color: #2d3748;
  color: white;
  padding: 1.5rem 0;
  text-align: center;
  margin-top: 2rem;
}

.app-footer p {
  font-size: 0.9rem;
  opacity: 0.8;
}

/* 响应式设计 */
@media (max-width: 1024px) {
  .main-images-grid {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (max-width: 768px) {
  .content-wrapper {
    grid-template-columns: 1fr;
    gap: 1.5rem;
  }

  .main-images-grid {
    grid-template-columns: 1fr;
  }

  .app-header h1 {
    font-size: 2rem;
  }

  .subtitle {
    font-size: 1rem;
  }

  .app-main {
    padding: 1rem;
  }

  .input-section,
  .generation-section,
  .results-section {
    padding: 1.5rem;
  }

  .generate-button {
    padding: 0.75rem 2rem;
    font-size: 1.1rem;
  }
}

@media (max-width: 480px) {
  .app-header {
    padding: 1.5rem 0;
  }

  .app-header h1 {
    font-size: 1.75rem;
  }

  .specification-row {
    flex-wrap: wrap;
  }

  .specification-row .spec-input {
    flex: 100%;
    margin-bottom: 0.5rem;
    margin-right: 0;
  }

  .specification-row .remove-btn {
    margin-left: auto;
  }

  .download-all-btn,
  .regenerate-btn {
    width: 100%;
    margin-bottom: 1rem;
    margin-right: 0;
  }
}
</style>
