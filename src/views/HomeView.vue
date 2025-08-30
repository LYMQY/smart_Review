<template>
  <div class="doubao-chat-container">
    <!-- 标题区域 -->
    <div class="header">
      <h1 class="title">你好，欢迎使用智审通</h1>
      <p class="subtitle">有什么我可以帮助你的吗？</p>
    </div>

    <!-- 输入区域 -->
    <div class="input-container">
      <div class="input-wrapper">
        <textarea
          v-model="problemText"
          class="input-textarea"
          :rows="isMobile ? 1 : 2"
          placeholder="发消息或输入 / 选择技能"
          @keydown.enter.exact.prevent="sendQuestion"
          @keydown.enter.shift.exact.prevent="problemText += '\n'"
        />
        <div class="input-btns">
          <div class="left-btns">
            <el-button type="text" class="btn-icon" @click="handleFileUpload">
              <el-icon><Document /></el-icon>
            </el-button>
            <el-button type="text" class="skill-btn" @click="handleDeepThinking">
              <span class="skill-icon">深度思考</span>
            </el-button>
          </div>
          <div class="right-btns">
            <el-button type="text" class="btn-icon" @click="handleVoiceInput">
              <el-icon><Message /></el-icon>
            </el-button>
            <el-button type="text" class="btn-icon" @click="handleEdit">
              <el-icon><Edit /></el-icon>
            </el-button>
            <el-button
              type="primary"
              class="send-btn"
              :disabled="sendBtnDisabled || !problemText.trim()"
              @click="sendQuestion"
            >
              <el-icon><ArrowUp /></el-icon>
            </el-button>
          </div>
        </div>
      </div>
      
      <!-- 功能按钮区域 -->
      <div class="function-btns">
        <el-button type="text" class="func-btn" @click="handleFunction('research')">深入研究</el-button>
        <el-button type="text" class="func-btn" @click="handleFunction('write')">帮我写作</el-button>
        <el-button type="text" class="func-btn" @click="handleFunction('image')">图像生成</el-button>
        <el-button type="text" class="func-btn" @click="handleFunction('code')">AI 编程</el-button>
        <el-button type="text" class="func-btn" @click="handleFunction('translate')">翻译</el-button>
        <el-button type="text" class="func-btn" @click="handleFunction('meeting')">记录会议</el-button>
        <el-button type="text" class="func-btn more-btn" @click="handleMoreFunctions">更多</el-button>
      </div>
      
      <div class="mobile-tip" v-if="isMobile">
        提示：长按输入框可粘贴内容
      </div>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { Document, Edit, Message, ArrowUp } from "@element-plus/icons-vue";
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

// 状态定义
let sparkWS: WebSocket;
let chatList = ref<ChatItem[]>([]);
let loadingIndex = ref<number | null | undefined>();
let problemText = ref<string>("");
let wsMsgReceiveStatus = ref<"receiveIng" | "receiveFinished">();
const maxCharCount = ref<number>(1000);
const sendBtnDisabled = ref(false);
const isMobile = ref(false);
const showChatList = ref(true); // 控制聊天列表显示

// 检测设备类型
const checkDevice = () => {
  isMobile.value = window.innerWidth < 768;
};

// 发送问题函数
const sendQuestion = () => {
  if (sendBtnDisabled.value) return;
  const trimText = problemText.value?.trim();
  if (!trimText) {
    ElMessage.warning("请输入您想要了解的内容...");
    return;
  }
  
  showChatList.value = true; // 显示聊天列表
  
  if (wsMsgReceiveStatus.value !== "receiveIng") {
    chatList.value.push({ role: "user", content: trimText });
    sendBtnDisabled.value = true;
    askSpark();
  }
};

// WebSocket 逻辑
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

// 响应处理
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

// 聊天列表滚动
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

// 工具函数
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

// 功能按钮处理函数
const handleFileUpload = () => {
  ElMessage.info("文件上传功能即将开放");
};

const handleDeepThinking = () => {
  problemText.value += "[深度思考]";
  ElMessage.info("已开启深度思考模式");
};

const handleVoiceInput = () => {
  ElMessage.info("语音输入功能即将开放");
};

const handleEdit = () => {
  ElMessage.info("编辑功能即将开放");
};

const handleFunction = (type: string) => {
  const functionMap = {
    'research': '深入研究',
    'write': '帮我写作',
    'image': '图像生成',
    'code': 'AI 编程',
    'translate': '翻译',
    'meeting': '记录会议'
  };
  
  problemText.value += `[${functionMap[type]}] `;
  ElMessage.info(`已选择${functionMap[type]}功能`);
};

const handleMoreFunctions = () => {
  ElMessage.info("更多功能即将开放");
};

// 生命周期与监听
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

<style lang="scss" scoped>
// 全局样式变量
$primary-color: #0066ff; // 主色
$light-primary: #e8f3ff; // 浅色背景
$text-primary: #333333; // 主文本色
$text-secondary: #666666; // 次要文本色
$text-placeholder: #999999; // 占位文本色
$border-color: #e5e7eb; // 边框色
$disabled-color: #cccccc; // 禁用色

// 主容器
.doubao-chat-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  width: 100%;
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 16px;
  box-sizing: border-box;
  background-color: #ffffff;
}

// 标题区域
.header {
  text-align: center;
  margin-top: 60px;
  margin-bottom: 20px;
  transition: all 0.3s ease;

  .title {
    font-size: 36px;
    font-weight: 700;
    color: $text-primary;
    margin-bottom: 8px;
  }

  .subtitle {
    font-size: 16px;
    color: $text-secondary;
    opacity: 0.8;
  }
}
// 输入区域
.input-container {
  position: sticky;
  bottom: 0;
  z-index: 10;
  width: 100%;
  padding: 16px 0;
  background-color: #ffffff;
  //box-shadow: 0 -2px 16px rgba(0,102,255,0.04);

  .input-wrapper {
    display: flex;
    flex-direction: column; // 垂直排列
    align-items: stretch;
    background: rgba(255,255,255,0.95);
    border: 1.5px solid $border-color;
    border-radius: 28px;
    box-shadow: 0 2px 12px rgba(0,102,255,0.06);
    overflow: hidden;
    transition: border-color 0.2s, box-shadow 0.2s;
    width: 100%;
    min-height: 140px;
    max-width: 820px;
    margin: 0 auto; 

    &:focus-within {
      border-color: $primary-color;
      box-shadow: 0 0 0 3px rgba(0,102,255,0.12);
    }
  }

  .input-textarea {
    flex: 1;
    width: 100%;
    min-height: 44px;
    max-height: 180px;
    padding: 12px 16px;
    background: transparent;
    border: none;
    outline: none;
    resize: none;
    font-size: 15px;
    color: $text-primary;
    box-sizing: border-box;
    margin-bottom: 8px; // 增加与按钮区域的间距

    &::placeholder {
      color: $text-placeholder;
      font-size: 15px;
      opacity: 0.7;
    }

    &::-webkit-scrollbar {
      display: none;
    }
  }

  .input-btns {
    display: flex;
    justify-content: space-between; // 两侧分布
    align-items: center;
    gap: 12px;
    padding: 0 12px 4px 12px;

    .left-btns,
    .right-btns {
      display: flex;
      align-items: center;
      gap: 8px;
    }
  }

  .left-btns {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  .right-btns {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  .btn-icon {
    color: $text-primary;
    font-size: 20px;
    padding: 10px;
    border-radius: 50%;
    background: transparent;
    transition: all 0.2s;

    &:hover {
      background: linear-gradient(135deg, #e0e7ff 0%, #cce0ff 100%);
      color: $primary-color;
      box-shadow: 0 2px 8px rgba(0,102,255,0.10);
    }
  }

  .skill-btn {
    color: $primary-color;
    font-size: 15px;
    font-weight: 500;
    padding: 8px 18px;
    border-radius: 18px;
    background: linear-gradient(90deg, #e8f3ff 0%, #f5f7fa 100%);
    box-shadow: 0 2px 8px rgba(0,102,255,0.06);
    transition: all 0.2s;

    &:hover {
      background: linear-gradient(90deg, #0066ff 0%, #409eff 100%);
      color: #fff;
      box-shadow: 0 4px 16px rgba(0,102,255,0.14);
    }

    .skill-icon {
      display: inline-flex;
      align-items: center;
      gap: 4px;
    }
  }

  .send-btn {
    min-width: 44px;
    height: 40px;
    margin-left: 10px;
    border-radius: 16px;
    background: linear-gradient(90deg, #0066ff 0%, #409eff 100%);
    border: none;
    font-size: 16px;
    color: #fff;
    font-weight: 600;
    box-shadow: 0 2px 8px rgba(0,102,255,0.10);
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.2s;

    &:hover, &:focus {
      background: linear-gradient(90deg, #0052d9 0%, #409eff 100%);
      box-shadow: 0 4px 16px rgba(0,102,255,0.16);
    }

    &:disabled {
      background: #cce0ff;
      color: #fff;
      cursor: not-allowed;
      box-shadow: none;
    }

    .el-icon {
      font-size: 18px;
    }
}

  // 功能按钮区域
  .function-btns {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    gap: 12px;
    margin-top: 16px;
    padding: 8px 0;

    .func-btn {
      color: $text-primary;
      font-size: 14px;
      padding: 6px 16px;
      border: 1px solid $border-color;
      border-radius: 20px;
      background-color: #ffffff;
      transition: all 0.2s;

      &:hover {
        color: $primary-color;
        border-color: $primary-color;
        background-color: rgba(0, 102, 255, 0.05);
      }
    }

    .more-btn {
      position: relative;

      &::after {
        content: '...';
        position: absolute;
        right: 8px;
        top: 50%;
        transform: translateY(-50%);
        font-size: 12px;
      }
    }
  }

  .mobile-tip {
    font-size: 13px;
    color: $text-secondary;
    margin-top: 6px;
    text-align: center;
    opacity: 0.85;
  }
}

// 动画
@keyframes rotate {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

// 响应式适配
@media (max-width: 768px) {
  .header {
    margin-top: 40px;
    margin-bottom: 10px;

    .title {
      font-size: 28px;
    }
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
  }

  .input-container {
    padding: 10px 0;

    .input-wrapper {
      border-radius: 24px;
    }

    .left-btns, .right-btns {
      padding: 0 8px;
      gap: 4px;

      .btn-icon {
        font-size: 18px;
        padding: 8px;
      }

      .skill-btn {
        padding: 6px 12px;
        font-size: 14px;
      }

      .send-btn {
        min-width: 38px;
        height: 36px;
        margin-left: 4px;
      }
    }

    .function-btns {
      gap: 8px;
      margin-top: 12px;

      .func-btn {
        font-size: 13px;
        padding: 5px 12px;
      }
    }
  }
}
</style>
