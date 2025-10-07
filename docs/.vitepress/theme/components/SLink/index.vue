<template>
  <div class="my-links-container">
    <!-- 页面主标题区域 -->
    <div class="my-links-title">
      <h1>{{ title }}</h1>
    </div>

    <!-- 友链分组列表，每个分组包含标题、描述和友链列表 -->
    <div
      v-for="(group, index) in linksData"
      :key="index"
      class="my-links-group"
    >
      <!-- 分组标题容器 -->
      <div class="title-wrapper">
        <h3>{{ group.title }}</h3>
      </div>

      <!-- 分组描述文本 -->
      <p class="group-desc">{{ group.descr }}</p>

      <!-- 友链列表容器 -->
      <div class="links-grid">
        <!-- 每个友链项使用LinkItem子组件展示，通过:data传递友链信息 -->
        <div
          v-for="link in group.list"
          :key="link.link"
          class="links-grid__item"
        >
          <LinkItem :data="link" />
        </div>
      </div>
    </div>

    <!-- 留言/评论区域，默认显示，可通过frontmatter隐藏 -->
    <div v-if="shouldShow" class="my-message-section">
      <div class="title-wrapper">
        <h3>留链吗</h3>
      </div>
      <p>留恋的小伙伴，想要和我做友链 💞</p>

      <!-- 留言卡片容器 -->
      <div class="message-card">
        <p>欢迎在评论区留言，格式如下：</p>
        <!-- 示例格式 -->
        <div class="example-container">
          <pre ref="exampleRef">
网站名称: Hyde Blog
网站链接: https://teek.seasir.top/
网站头像: https://teek.seasir.top/avatar/avatar.webp
网站描述: 人心中的成见是一座大山~</pre
          >
          <button class="copy-button" @click="copyExample">
            <span class="copy-icon"></span>
            复制示例
          </button>
        </div>
        <!-- 评论区插槽 -->
        <!-- 默认为Twikoo评论组件，可通过插槽自定义其他评论系统 -->
        <slot name="comments">
          <Twikoo />
        </slot>
      </div>
    </div>

    <!-- 滚动到评论区按钮 -->
    <ScrollToComment
      v-if="shouldShow"
      :show="showScrollButton"
      :scroll-to-comment="scrollToComment"
    />
  </div>
</template>

<script setup>
import { useData } from "vitepress";
import LinkItem from "./LinkItem.vue";
import Twikoo from "../Twikoo.vue";
import ScrollToComment from "../ScrollToComment.vue";
import { computed, ref, onMounted, onUnmounted } from "vue";
import { TkMessage } from "vitepress-theme-teek";

/**
 * 单个友链的数据结构定义
 * @typedef {Object} Link
 * @property {string} name - 友链网站名称
 * @property {string} link - 友链网站URL地址
 * @property {string} avatar - 友链网站头像/Logo的图片URL
 * @property {string} descr - 友链网站的简短描述
 * @property {boolean} [irregular] - 可选参数，默认值为false，为false时将把头像处理为圆形
 */

/**
 * 友链分组的数据结构定义
 * @typedef {Object} LinkGroup
 * @property {string} title - 分组标题
 * @property {string} desc - 分组描述文字
 * @property {Link[]} list - 该分组下的友链列表数组
 */

// 从页面frontmatter中获取配置数据
const { frontmatter } = useData();

// 从frontmatter中读取links字段，如果未定义则使用空数组
const linksData = computed(() => frontmatter.value.links || []);

// 从frontmatter中读取title字段，默认值为"我的友链"
const title = computed(() => frontmatter.value.title || "我的友链");

// 当frontmatter中comments为false时隐藏，默认显示
const shouldShow = computed(() => frontmatter.value.comments !== false);

// 示例文本引用
const exampleRef = ref(null);

// 复制示例文本函数
const copyExample = async () => {
  if (exampleRef.value) {
    const exampleText = exampleRef.value.textContent;
    try {
      await navigator.clipboard.writeText(exampleText);
      TkMessage({
        message: "示例格式已复制",
        type: "success",
      });
    } catch (err) {
      // 降级方案：使用 document.execCommand
      const textArea = document.createElement("textarea");
      textArea.value = exampleText;
      document.body.appendChild(textArea);
      textArea.select();
      try {
        document.execCommand("copy");
        TkMessage({
          message: "示例格式已复制",
          type: "success",
        });
      } catch (fallbackErr) {
        TkMessage({
          message: "复制失败，请手动复制示例文本",
          type: "error",
        });
      } finally {
        document.body.removeChild(textArea);
      }
    }
  }
};

// 控制按钮显示状态
const showScrollButton = ref(true);

// 滚动到评论区的函数
const scrollToComment = () => {
  const commentElement = document.querySelector(
    "#twikoo, .my-message-section, .message-card"
  );
  if (commentElement) {
    commentElement.scrollIntoView({
      behavior: "smooth",
      block: "start",
    });

    // 显示成功消息提示
    TkMessage({
      message: "已跳转到留链区，欢迎留下您的友链信息✨",
      type: "success",
    });
  } else {
    // 如果没有找到评论区域，显示提示
    TkMessage({
      message: "未找到留链区",
      type: "warning",
    });
  }
};

// 检查是否滚动到评论区
const checkScrollPosition = () => {
  const commentElement = document.querySelector(
    ".my-message-section, .message-card"
  );
  if (commentElement) {
    const rect = commentElement.getBoundingClientRect();
    const windowHeight = window.innerHeight;

    // 如果评论区域的顶部进入视窗，则隐藏按钮
    showScrollButton.value = rect.top > windowHeight * 0.3;
  }
};

// 节流函数，避免频繁触发
let throttleTimer = null;
const throttledCheckScroll = () => {
  if (throttleTimer) return;
  throttleTimer = setTimeout(() => {
    checkScrollPosition();
    throttleTimer = null;
  }, 100);
};

// 组件挂载时添加滚动监听
onMounted(() => {
  window.addEventListener("scroll", throttledCheckScroll);
  // 初始检查
  setTimeout(checkScrollPosition, 100);
});

// 组件卸载时移除监听
onUnmounted(() => {
  window.removeEventListener("scroll", throttledCheckScroll);
  if (throttleTimer) {
    clearTimeout(throttleTimer);
  }
});
</script>

<style scoped>
/* 主容器样式 */
.my-links-container {
  max-width: 1500px;
  margin: 0 auto;
  padding: 40px 10px;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen,
    Ubuntu, Cantarell, "Open Sans", "Helvetica Neue", sans-serif;
}

/* 标题区域样式 */
.my-links-title {
  margin-bottom: 50px;
  padding: 0 10px;
  /* 居中 */
  text-align: center;
}

/* 主标题样式 */
.my-links-title h1 {
  font-size: 2rem;
  font-weight: 600;
  background: -webkit-linear-gradient(
    107deg,
    rgb(255, 182, 133) -30.6%,
    rgb(255, 111, 29) -1.11%,
    rgb(252, 181, 232) 39.14%,
    rgb(135, 148, 255) 73.35%,
    rgb(60, 112, 255) 97.07%,
    rgb(60, 112, 255) 118.97%
  );
  background-clip: text;
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  -webkit-font-smoothing: antialiased;
  text-rendering: optimizeLegibility;
  line-height: 1.2;
  display: inline-block;
  font-size: 1.5rem;
}

/* 分组标题装饰线样式 */
.title-wrapper {
  position: relative;
  margin: 40px 0;
  height: 1px;
  background: #ddd;
  transition: 0.3s;
}

/* 分组标题文字样式 */
.title-wrapper h3 {
  position: absolute;
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
  background: var(--vp-c-bg);
  padding: 0 20px;
  font-size: 1.3rem;
  font-weight: 600;
  color: var(--vp-c-text-1);
  z-index: 1;
}

/* 分组描述文字样式 */
.group-desc {
  text-align: center;
  color: var(--vp-c-text-2);
  font-size: 0.95rem;
  margin-bottom: 30px;
  padding: 0 10px;
}

/* 友链网格布局 - 核心响应式实现 */
.links-grid {
  display: flex;
  flex-wrap: wrap;
  justify-content: center; /* 让所有行的内容居中对齐 */
  gap: 24px;
  margin-bottom: 60px;
  padding: 0 8px;
}

/* 每个友链项的样式，设置基础宽度 */
.links-grid__item {
  flex: 0 0 calc(100% - 24px); /* 移动设备：每行1个 */
  break-inside: avoid;
}

.link-content:hover {
  margin-left: calc(-5 * 16px);
}

/* 平板设备：每行2个 */
@media (min-width: 768px) {
  .links-grid__item {
    flex: 0 0 calc(50% - 24px);
  }
}

/* 桌面设备：每行最多4个 */
@media (min-width: 1024px) {
  .links-grid__item {
    flex: 0 0 calc(25% - 24px);
  }
}

/* 留言区样式 */
.my-message-section {
  text-align: center;
  margin-top: 20px;
}

/* 留言卡片样式 */
.message-card {
  width: 100%;
  max-width: 1500px;
  margin: 30px auto;
  padding: 32px;
  border-radius: 12px;
  background: var(--vp-c-bg);
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
  border: 1px solid var(--vp-c-divider);
  text-align: left;
  transition: all 0.2s ease;
}

/* 移动端留言卡片适配 */
@media (max-width: 768px) {
  .message-card {
    padding: 24px;
    margin: 24px auto;
  }
}

/* 示例容器样式 */
.example-container {
  position: relative;
  margin: 20px 0;
}

/* 示例格式代码块样式 */
.message-card pre {
  background: var(--vp-code-block-bg);
  padding: 16px;
  border-radius: 8px;
  font-size: 0.95rem;
  overflow-x: auto;
  margin: 0;
  border: 1px solid var(--vp-c-divider);
  line-height: 1.5;
}

/* 复制按钮样式 */
.copy-button {
  position: absolute;
  top: 8px;
  right: 8px;
  background: var(--vp-c-brand);
  color: white;
  padding: 4px;
  border-radius: 6px;
  font-size: 0.85rem;
  display: flex;
  align-items: center;
  transition: all 0.2s ease;
}

.copy-button:hover {
  background: var(--vp-c-indigo-3);
  transform: translateY(-2px);
}

.copy-button:active {
  transform: translateY(0);
}

.copy-icon {
  font-size: 0.9rem;
}

/* 留言卡片悬停效果 */
.message-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 28px rgba(0, 0, 0, 0.12);
}
</style>
