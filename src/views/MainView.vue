<script lang="ts" setup>
import { UserFilled, Loading, DocumentCopy, Delete } from "@element-plus/icons-vue";
import { onBeforeUnmount, onMounted, ref, watch } from "vue";
import {
  getWebsocketUrl,
  ChatItem,
  WSReqParams,
  WSResParams,
  wsSendMsgFormat,
} from "@/ts/AiChatWebsocket.ts";
import { sparkConfig } from "@/config/config.ts";
import { ElMessage, ElAvatar, ElButton, ElIcon } from "element-plus";

// 1. 状态定义（保持原逻辑，补充响应式适配相关）
let sparkWS: WebSocket;
let chatList = ref<ChatItem[]>([]);
let loadingIndex = ref<number | null | undefined>();
let problemText = ref<string>("");
let wsMsgReceiveStatus = ref<"receiveIng" | "receiveFinished">();
const maxCharCount = ref<number>(1000); // 匹配豆包：最大输入1000字
const sendBtnDisabled = ref(false);
const isMobile = ref(false); // 新增：判断移动端

// 2. 检测设备类型（响应式适配）
const checkDevice = () => {
  isMobile.value = window.innerWidth < 768;
};

// 3. 发送问题函数（保持原逻辑）
const sendQuestion = () => {
  if (sendBtnDisabled.value) return;
  const trimText = problemText.value?.trim();
  if (!trimText) {
    ElMessage.warning("请输入您想要了解的内容...");
    return;
  }
  if (wsMsgReceiveStatus.value !== "receiveIng") {
    chatList.value.push({ role: "user", content: trimText });
    sendBtnDisabled.value = true;
    askSpark();
  }
};

// 4. WebSocket 逻辑（保持原逻辑）
const askSpark = () => {
  const wsUrl = getWebsocketUrl(sparkConfig);
  if (!("WebSocket" in window)) {
    ElMessage.error("当前浏览器不支持WebSocket，无法使用聊天功能");
    return;
  }
  sparkWS = new WebSocket(wsUrl as string);

  sparkWS.onopen = () => {
    const sendData: WSReqParams = wsSendMsgFormat(sparkConfig, chatList.value);
    sparkWS.send(JSON.stringify(sendData));
    chatList.value.push({ role: "assistant", content: "" });
    loadingIndex.value = chatList.value.length - 1;
  };

  sparkWS.onmessage = (res: MessageEvent) => {
    const resObj: WSResParams = JSON.parse(res.data);
    if (resObj.header.code !== 0) {
      ElMessage.error(`提问失败：${resObj.header.message}`);
      console.error("WebSocket 错误:", resObj.header);
      sparkWS.close();
      sendBtnDisabled.value = false;
      return;
    }
    wsMsgReceiveHandle(resObj);
  };

  sparkWS.onerror = (e) => {
    ElMessage.error("WebSocket 连接异常，请刷新重试");
    console.error("WebSocket 错误:", e, "URL:", wsUrl);
    sendBtnDisabled.value = false;
  };

  sparkWS.onclose = () => {
    if (wsMsgReceiveStatus.value === "receiveIng") {
      ElMessage.warning("聊天连接已断开");
      sendBtnDisabled.value = false;
    }
  };
};

// 5. 响应处理（保持原逻辑）
const wsMsgReceiveHandle = (res: WSResParams) => {
  const dataArray = res?.payload?.choices?.text || [];
  dataArray.forEach(item => {
    chatList.value[chatList.value.length - 1].content += item.content;
  });

  switch (res.payload.choices.status) {
    case 0:
      problemText.value = "";
      wsMsgReceiveStatus.value = "receiveIng";
      break;
    case 1:
      wsMsgReceiveStatus.value = "receiveIng";
      break;
    case 2:
      wsMsgReceiveStatus.value = "receiveFinished";
      loadingIndex.value = null;
      sendBtnDisabled.value = false;
      sparkWS.close();
      break;
  }
};

// 6. 聊天列表滚动（优化平滑度）
let chatListObserver: MutationObserver;
const aiChatListRef = ref<HTMLUListElement>(null);

const createMutationServer = (targetElement: Element) => {
  chatListObserver = new MutationObserver(() => {
    if (aiChatListRef.value) {
      const scrollHeight = aiChatListRef.value.scrollHeight;
      scrollHandle(scrollHeight);
    }
  });
  chatListObserver.observe(targetElement, { childList: true, subtree: true });
};

const scrollHandle = (val: number) => {
  aiChatListRef.value?.scrollTo({
    top: val,
    behavior: "smooth"
  });
};

// 7. 工具函数（保持原逻辑，优化提示文案）
import { copyToClipboard } from "@/utils/commonUtil.ts";
const copyRecord = (item: { content: string }) => {
  copyToClipboard({
    content: item.content,
    success() {
      ElMessage.success("复制成功，已存入剪贴板");
    },
    error() {
      ElMessage.error("复制失败，请手动复制");
    }
  });
};

const handleCopyCodeSuccess = () => {
  ElMessage.success("代码复制成功");
};

const deleteRecord = (index: number) => {
  if (sendBtnDisabled.value) {
    ElMessage.warning("当前正在生成回复，暂无法删除");
    return;
  }
  chatList.value.splice(index, 1);
};

const reReply = (index: number) => {
  if (wsMsgReceiveStatus.value === "receiveIng") {
    ElMessage.warning("当前正在生成回复，暂无法重新提问");
    return;
  }
  if (chatList.value.length - 1 === index) {
    deleteRecord(index);
    sendBtnDisabled.value = true;
    askSpark();
  } else {
    let i = index - 1;
    while (i >= 0) {
      if (chatList.value[i].role === "user" && chatList.value[i].content) {
        chatList.value.push({
          role: "user",
          content: chatList.value[i].content
        });
        sendBtnDisabled.value = true;
        askSpark();
        break;
      }
      i--;
    }
  }
};

// 8. 生命周期与监听（补充窗口 resize 监听）
const problemTextWatcher = watch(
  () => problemText.value,
  () => {
    if (problemText.value.length > maxCharCount.value) {
      problemText.value = problemText.value.slice(0, maxCharCount.value);
      ElMessage.warning(`最多可输入${maxCharCount.value}字`);
    }
  }
);

onMounted(() => {
  checkDevice();
  window.addEventListener("resize", checkDevice);
  if (aiChatListRef.value) {
    createMutationServer(aiChatListRef.value);
  }
});

onBeforeUnmount(() => {
  problemTextWatcher();
  chatListObserver?.disconnect();
  window.removeEventListener("resize", checkDevice);
  sparkWS?.close();
});
</script>

<template>
  <!-- 主容器：匹配豆包全屏布局 -->
  <div class="doubao-chat-container">
    <!-- 聊天列表区域：占满剩余高度，滚动流畅 -->
    <ul class="chat-list" ref="aiChatListRef">
      <!-- 初始欢迎消息：匹配豆包引导样式 -->
      <li class="chat-item init-item" v-if="chatList.length === 0">
        <ElAvatar 
          class="chat-avatar" 
          src="https://img.anfensi.com/upload/2019-1/2019141039392150.png"
          size="40"
        />
        <div class="chat-content init-content">
          <div class="init-title">欢迎使用智审通</div>
          <div class="init-desc">我是智审通，能帮你审核资料、生成内容、陪你聊天～</div>
          <div class="init-tips">试试提问："帮我审核这份征信授权书"、"解释数据管理规范"...</div>
        </div>
      </li>

      <!-- 聊天记录：循环渲染 -->
      <li 
        class="chat-item" 
        :class="`chat-item--${item.role}`"
        v-for="(item, index) of chatList"
        :key="index"
      >
        <!-- 头像：用户用图标，豆包用logo -->
        <ElAvatar class="chat-avatar">
          <template #default>
            <UserFilled v-if="item.role === 'user'" />
            <img 
              v-else 
              src="https://img1.baidu.com/it/u=2640995470,2945739766&fm=253&fmt=auto&app=120&f=JPEG?w=500&h=500"
              alt="豆包头像"
              class="avatar-img"
            />
          </template>
        </ElAvatar>

        <!-- 聊天内容：区分用户/豆包样式 -->
        <div class="chat-content" :class="`chat-content--${item.role}`">
          <!-- 用户内容：简洁白色背景 -->
          <div v-if="item.role === 'user'" class="content-text">
            {{ item.content }}
          </div>

          <!-- 豆包内容：带功能按钮 + 加载动画 -->
          <div v-else class="assistant-content">
            <div class="content-text">
              <v-md-preview 
                :text="item.content" 
                @copy-code-success="handleCopyCodeSuccess"
                class="md-preview"
              />
              <!-- 加载动画：豆包品牌色旋转 -->
              <div class="loading-box" v-if="loadingIndex === index">
                <ElIcon class="loading-icon">
                  <Loading />
                </ElIcon>
                <span class="loading-text">豆包正在思考...</span>
              </div>
            </div>

            <!-- 操作按钮：重新回答/复制/删除 -->
            <div class="content-actions" v-if="loadingIndex !== index">
              <span 
                class="action-btn re-reply-btn"
                @click="reReply(index)"
                :class="{ disabled: sendBtnDisabled }"
              >
                重新回答
              </span>
              <div class="action-icons">
                <ElIcon 
                  class="action-icon copy-icon"
                  @click="copyRecord(item)"
                  :class="{ disabled: sendBtnDisabled }"
                >
                  <DocumentCopy />
                </ElIcon>
                <ElIcon 
                  class="action-icon delete-icon"
                  @click="deleteRecord(index)"
                  :class="{ disabled: sendBtnDisabled }"
                >
                  <Delete />
                </ElIcon>
              </div>
            </div>
          </div>
        </div>
      </li>
    </ul>

    <!-- 输入区域：固定底部，匹配豆包样式 -->
    <div class="input-container">
      <div class="input-wrapper">
        <textarea
          v-model="problemText"
          class="input-textarea"
          :rows="isMobile ? 3 : 4"
          placeholder="请输入问题..."
          @keydown.enter.exact.prevent="sendQuestion"
          @keydown.enter.shift.exact.prevent="problemText += '\n'"
        />
        <!-- 字数提示 -->
        <div class="char-count">
          {{ problemText.length }} / {{ maxCharCount }}
        </div>
        <!-- 发送按钮 -->
        <ElButton
          class="send-btn"
          type="primary"
          :disabled="sendBtnDisabled || !problemText.trim()"
          @click="sendQuestion"
        >
          发送
        </ElButton>
      </div>
      <!-- 移动端提示：适配小屏幕 -->
      <div class="mobile-tip" v-if="isMobile">
        提示：长按输入框可粘贴内容
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
// 1. 全局样式：匹配豆包品牌色与基础规范
$primary-color: #0066ff; // 豆包主色
$light-primary: #e8f3ff; // 豆包浅色背景
$text-primary: #333333; // 主文本色
$text-secondary: #666666; // 次要文本色
$text-placeholder: #999999; // 占位文本色
$border-color: #e5e7eb; // 边框色
$disabled-color: #cccccc; // 禁用色

// 2. 主容器：全屏布局
.doubao-chat-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  width: 100%;
  max-width: 1200px; // 豆包最大宽度限制
  margin: 0 auto;
  padding: 0 16px;
  box-sizing: border-box;
  background-color: #ffffff;
}

// 3. 聊天列表：滚动优化 + 间距规范
.chat-list {
  flex: 1;
  overflow-y: auto;
  padding: 24px 0;
  margin: 0;
  list-style: none;

  // 滚动条样式：豆包细滚动条
  &::-webkit-scrollbar {
    width: 6px;
    height: 6px;
  }
  &::-webkit-scrollbar-thumb {
    background-color: rgba(0, 0, 0, 0.1);
    border-radius: 3px;
  }
  &::-webkit-scrollbar-track {
    background-color: transparent;
  }

  // 单个聊天项：统一间距
  .chat-item {
    display: flex;
    align-items: flex-start;
    margin-bottom: 24px;
    position: relative;

    // 头像：固定大小 + 间距
    .chat-avatar {
      width: 40px;
      height: 40px;
      margin-right: 12px;
      flex-shrink: 0;

      .avatar-img {
        width: 100%;
        height: 100%;
        object-fit: cover;
      }
    }

    // 聊天内容容器：基础样式
    .chat-content {
      max-width: calc(100% - 52px); // 头像宽度 + 间距
      box-sizing: border-box;
      padding: 12px 16px;
      border-radius: 8px;
      line-height: 1.6;
      font-size: 14px;

      // 文本样式
      .content-text {
        color: $text-primary;
        white-space: pre-wrap; // 保留换行
        word-break: break-word; // 长词换行
      }
    }

    // 初始欢迎项：豆包引导样式
    &.init-item {
      .init-content {
        background-color: $light-primary;
        border-radius: 12px;
        padding: 24px;
        max-width: 100%;

        .init-title {
          font-size: 18px;
          font-weight: 600;
          color: $primary-color;
          margin-bottom: 8px;
        }
        .init-desc {
          color: $text-primary;
          margin-bottom: 16px;
        }
        .init-tips {
          color: $text-secondary;
          font-size: 13px;
          background-color: rgba(0, 102, 255, 0.1);
          padding: 8px 12px;
          border-radius: 4px;
        }
      }
    }

    // 用户聊天项：右侧对齐 + 蓝色背景
    &.chat-item--user {
      flex-direction: row-reverse;

      .chat-avatar {
        margin-right: 0;
        margin-left: 12px;
      }

      .chat-content--user {
        background-color: $primary-color;
        color: #ffffff;

        .content-text {
          color: #ffffff;
        }
      }
    }

    // 豆包聊天项：左侧对齐 + 浅色背景
    &.chat-item--assistant {
      .chat-content--assistant {
        background-color: $light-primary;
        border-radius: 12px;

        // 加载动画：居中显示
        .loading-box {
          display: flex;
          align-items: center;
          padding: 16px 0;
          color: $primary-color;

          .loading-icon {
            margin-right: 8px;
            animation: rotate 1.5s linear infinite;
          }
          .loading-text {
            font-size: 13px;
          }
        }

        // 操作按钮：hover 效果
        .content-actions {
          display: flex;
          justify-content: space-between;
          align-items: center;
          margin-top: 12px;
          padding-top: 8px;
          border-top: 1px solid rgba(0, 102, 255, 0.1);

          .re-reply-btn {
            color: $primary-color;
            font-size: 13px;
            cursor: pointer;
            &:hover {
              text-decoration: underline;
            }
            &.disabled {
              color: $disabled-color;
              cursor: not-allowed;
              text-decoration: none;
            }
          }

          .action-icons {
            display: flex;

            .action-icon {
              color: $primary-color;
              font-size: 14px;
              margin-left: 16px;
              cursor: pointer;
              &:hover {
                color: #004cc8; // 加深色
              }
              &.disabled {
                color: $disabled-color;
                cursor: not-allowed;
              }
            }
          }
        }

        // Markdown 预览：适配样式
        .md-preview {
          color: $text-primary;
          font-size: 14px;

          // 代码块样式：匹配豆包
          pre {
            background-color: #f5f7fa;
            border-radius: 4px;
            padding: 12px;
            margin: 8px 0;
            overflow-x: auto;
          }
          code {
            color: #e53935;
            background-color: #fef2f2;
            padding: 2px 4px;
            border-radius: 4px;
            font-size: 13px;
          }
        }
      }
    }
  }
}

// 4. 输入区域：固定底部 + 响应式
.input-container {
  padding: 16px 0;
  border-top: 1px solid $border-color;
  background-color: #ffffff;
  position: sticky;
  bottom: 0;
  z-index: 10;

  .input-wrapper {
    position: relative;
    display: flex;
    align-items: flex-end;
    background-color: #f9fafb;
    border-radius: 12px;
    padding: 8px;
    border: 1px solid $border-color;
    transition: border-color 0.2s;

    // 聚焦时高亮边框
    &:focus-within {
      border-color: $primary-color;
      box-shadow: 0 0 0 2px rgba(0, 102, 255, 0.1);
    }

    // 文本域：自适应高度 + 无边框
    .input-textarea {
      flex: 1;
      width: 100%;
      min-height: 60px; // 最小高度
      max-height: 200px; // 最大高度（防止过长）
      padding: 8px 12px;
      background-color: transparent;
      border: none;
      outline: none;
      resize: none;
      font-size: 14px;
      color: $text-primary;
      box-sizing: border-box;

      &::placeholder {
        color: $text-placeholder;
      }

      // 滚动条样式
      &::-webkit-scrollbar {
        width: 4px;
      }
      &::-webkit-scrollbar-thumb {
        background-color: rgba(0, 0, 0, 0.1);
        border-radius: 2px;
      }
    }

    // 字数提示：右下角
    .char-count {
      position: absolute;
      bottom: 12px;
      left: 16px;
      font-size: 12px;
      color: $text-secondary;
    }

    // 发送按钮：固定位置 + 品牌色
    .send-btn {
      min-width: 80px;
      height: 36px;
      margin-left: 8px;
      margin-bottom: 8px;
      border-radius: 6px;
      background-color: $primary-color;
      border-color: $primary-color;
      font-size: 14px;

      &:disabled {
        background-color: #cce0ff;
        border-color: #cce0ff;
        color: #ffffff;
        cursor: not-allowed;
      }
    }
  }

  // 移动端提示：小屏幕显示
  .mobile-tip {
    font-size: 12px;
    color: $text-secondary;
    margin-top: 8px;
    text-align: center;
  }
}

// 5. 动画：加载旋转
@keyframes rotate {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}

// 6. 响应式适配：小屏幕调整
@media (max-width: 768px) {
  .doubao-chat-container {
    padding: 0 8px;
  }

  .chat-list {
    padding: 16px 0;
  }

  .chat-item {
    margin-bottom: 16px;

    .chat-avatar {
      width: 36px;
      height: 36px;
      margin-right: 8px;
    }

    .chat-content {
      padding: 8px 12px;
      font-size: 13px;
    }

    &.init-item .init-content {
      padding: 16px;
    }
  }

  .input-container {
    padding: 8px 0;
  }

  .input-wrapper .send-btn {
    min-width: 70px;
    height: 32px;
    font-size: 13px;
  }
}
</style>