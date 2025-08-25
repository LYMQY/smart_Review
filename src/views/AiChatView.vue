<script lang="ts" setup>
import { UserFilled } from "@element-plus/icons-vue";
import { onBeforeUnmount, onMounted, ref, watch } from "vue";
import {
  getWebsocketUrl,
  ChatItem,
  WSReqParams,
  WSResParams,
  wsSendMsgFormat,
} from "@/ts/AiChatWebsocket.ts";
import { sparkConfig } from "@/config/config.ts";
import { ElMessage } from "element-plus";
//定义websocket 实例
let sparkWS: WebSocket;
//会话列表
let chatList = ref<ChatItem[]>([]);
//加载当前回答的index
let loadingIndex = ref<number | null | undefined>();
//提问问题内容
let problemText = ref<string>("");
//websocket 响应数据的状态
let wsMsgReceiveStatus = ref<"receiveIng" | "receiveFinished">();
//提问的最大字数
const maxCharCount = ref<number>(300);
//发送按钮的禁用状态
const sendBtnDisabled = ref(false);

//发送问题的函数
const sendQuestion = () => {
  if (sendBtnDisabled.value) {
    //阻止内容发送
    return;
  }
  if (problemText.value?.trim()?.length <= 0) {
    //输入问题文字是空字符串的提示
    ElMessage.warning({ message: "请输入您想要了解的内容..." });
    return;
  }
  //不在接受消息的时候才可以发送问题
  if (wsMsgReceiveStatus.value !== "receiveIng") {
    chatList.value.push({
      role: "user",
      content: problemText.value,
    });
    sendBtnDisabled.value = true;
    //调用连接星火认知大模型并发送问题的函数
    askSpark();
  }
};
//连接星火认知大模型并发送问题
const askSpark = () => {
  //1. 生成鉴权URL
  let wsUrl = getWebsocketUrl(sparkConfig);
  //2.判断浏览器是否websocket
  if ("WebSocket" in window) {
    //创建websocket 实例
    sparkWS = new WebSocket(wsUrl as string);
  } else {
    ElMessage.error("当前浏览器不支持websocket");
    return;
  }
  //3. 建立websocket连接
  //3.1 打开连接
  sparkWS.onopen = () => {
    //发送数据
    const sendData: WSReqParams = wsSendMsgFormat(sparkConfig, chatList.value);
    console.log("发送数据", JSON.stringify(sendData));
    sparkWS.send(JSON.stringify(sendData));

    chatList.value.push({
      role: "assistant",
      content: "",
    });
    loadingIndex.value = chatList.value.length - 1;
  };
  //3.2 消息监听
  sparkWS.onmessage = (res: MessageEvent) => {
    //响应消息
    let resObj: WSResParams = JSON.parse(res.data);
    if (resObj.header.code !== 0) {
      ElMessage.error("提问失败");
      console.error(
        `提问失败：${resObj.header.code} - ${resObj.header.message}`
      );
      //连接关闭
      sparkWS.close();
    } else {
      //处理响应回来的数据
      wsMsgReceiveHandle(resObj);
    }
  };
  //3.3 连接异常
  sparkWS.onerror = (e) => {
    ElMessage.error("WebSocket 连接失败" + e);
    console.error(`WebSocket 连接失败,连接的URL${wsUrl}`);
  };
  //3.4 连接关闭
  sparkWS.onclose = () => {};
};
/**
 * 处理websocket 返回的数据
 * @param res WSResParams
 */
const wsMsgReceiveHandle = (res: WSResParams) => {
  let dataArray = res?.payload?.choices?.text || [];
  for (let i = 0; i < dataArray.length; i++) {
    chatList.value[chatList.value.length - 1].content += dataArray[i].content;
  }
  //开始接受消息
  if (res.payload.choices.status === 0) {
    problemText.value = "";
    wsMsgReceiveStatus.value = "receiveIng";
  }
  //继续接受消息
  if (res.payload.choices.status === 1) {
    wsMsgReceiveStatus.value = "receiveIng";
  }
  //接受消息完毕
  if (res.payload.choices.status === 2) {
    wsMsgReceiveStatus.value = "receiveFinished";
    loadingIndex.value = null;
    sendBtnDisabled.value = false;
    sparkWS.close();
  }
};

//定义会话列表的观察对象（观察子元素的变化，当会话有变化时，自动滚动到变化的位置，用于提高用户的体验度）
let chatListObserver: MutationObserver;
//聊天列表引用对象
const aiChatListRef = ref();
/**
 * 聊天列表变化的观察对象：用于监听目标元素的高度变化
 * @param targetElement
 */
const createMutationServer = (targetElement: Element) => {
  //创建MutationServer 实例，用于监测Dom变化
  chatListObserver = new MutationObserver((mutationList, observer) => {
    //当子元素变化发生变化时，获取元素滚动的高度
    const scrollHeight = targetElement.scrollHeight;
    //调用滚动处理函数
    scrollHandle(scrollHeight);
  });
  //启动检测器并配置所需的观察选项
  chatListObserver.observe(targetElement, { childList: true, subtree: true });
};
/**
 * 滚动定位处理
 * @param val
 */
const scrollHandle = (val: number) => {
  aiChatListRef.value?.scrollTo({
    //scrollTo 把内容滚动到指定的坐标
    top: val,
    behavior: "smooth", //平滑滚动
  });
};
onMounted(() => {
  //调用监听目标元素高度变化的函数
  createMutationServer(aiChatListRef.value);
});
//导入复制内容到剪贴板的ts库
import { copyToClipboard } from "@/utils/commonUtil.ts";
/**
 * 复制会话内容到剪贴板
 * @param item
 * @param index
 */
const copyRecord = (item: { content: any }, index: any) => {
  const content = item.content;
  copyToClipboard({
    content,
    success() {
      ElMessage({
        message: "复制成功",
        type: "success",
      });
    },
    error() {
      ElMessage({
        message: "复制失败",
        type: "error",
      });
    },
  });
};
/**
 * markdown内容复制
 */
const handleCopyCodeSuccess = () => {
  ElMessage({
    message: "复制成功",
    type: "success",
  });
};
/**
 * 删除聊天记录
 * @param index
 */
const deleteRecord = (index: number) => {
  if (!sendBtnDisabled.value) {
    chatList.value.splice(index, 1);
  }
};
/**
 * 重新回答
 */
const reReply = (index: number) => {
  if (wsMsgReceiveStatus.value !== "receiveIng") {
    //如果是最后一条内容的重新回答，直接删除最后一条回答记录并重新作答
    if (chatList.value.length - 1 === index) {
      deleteRecord(index);
      sendBtnDisabled.value = true;
      askSpark();
    } else {
      //如果不是最后一条重新回答，则在后面重新添加问题继续进行询问
      //有可能上一条回答内容已被删除，所以需要循环找到最近的一条问题记录进行作答
      let i = index - 1;
      while (i >= 0) {
        //角色为用户并且有问题内容
        if (chatList.value[i].role === "user" && chatList.value[i].content) {
          chatList.value.push({
            role: "user",
            content: chatList.value[index - 1].content,
          });
          sendBtnDisabled.value = true;
          askSpark();
          break;
        }
        i--;
      }
    }
  }
};
/**
 * 监听问题内容长度的函数
 */
const problemTextWatcher = watch(
  () => problemText.value,
  () => {
    //限制最大字数
    if (problemText.value.length > maxCharCount.value) {
      problemText.value = problemText.value.slice(0, maxCharCount.value);
    }
  }
);
/**
 * 组件销毁之前的监听处理
 */
onBeforeUnmount(() => {
  //停止监听问题内容长度
  problemTextWatcher();
  //停止会话内容监听
  chatListObserver.disconnect();
});
</script>
<template>
  <div class="ai-chat-view">
    <ul class="ai-chat-list" ref="aiChatListRef">
      <li class="ai-chat-item">
        <!-- 头像 -->
        <div class="ai-chat-avatar">
          <el-avatar
            src="https://img1.baidu.com/it/u=2640995470,2945739766&fm=253&fmt=auto&app=120&f=JPEG?w=500&h=500"
            :size="40"
          />
        </div>
        <!-- 聊天内容 -->
        <div class="ai-chat-content-box init-box">
          <div class="ai-chat-title">AI工具</div>
          <div class="ai-chat-text">能够学习和理解人类的语言，进行多轮对话</div>
          <div class="ai-chat-text">
            回答问题，高效便捷地帮助人们获取信息、知识和灵感
          </div>
        </div>
      </li>
      <li
        class="ai-chat-item"
        :class="item.role + '-item'"
        v-for="(item, index) of chatList"
      >
        <!-- 头像 -->
        <div class="ai-chat-avatar">
          <el-avatar
            v-if="item.role === 'user'"
            :icon="UserFilled"
            :size="40"
          />
          <el-avatar
            v-if="item.role === 'assistant'"
            src="https://img1.baidu.com/it/u=2640995470,2945739766&fm=253&fmt=auto&app=120&f=JPEG?w=500&h=500"
            :size="40"
          />
        </div>
        <!-- 聊天内容 -->
        <div
          class="ai-chat-content-box"
          :class="item.role + '-box'"
          v-if="item.role === 'user'"
        >
          {{ item.content }}
        </div>
        <div
          class="ai-chat-content-box"
          :class="item.role + '-box'"
          v-if="item.role === 'assistant'"
        >
          AI回复内容
          <!-- 支持md 预览 -->
          <v-md-preview
            :text="item.content"
            @copy-code-success="handleCopyCodeSuccess"
          ></v-md-preview>
          <!-- 加载图标 -->
          <div class="loading-icon-box" v-if="loadingIndex === index">
            <el-icon>
              <Loading />
            </el-icon>
          </div>
          <div class="ai-chat-operate">
            <!--重新回答  -->
            <span
              class="re-reply-btn"
              @click="reReply(index)"
              :class="{ disabled: sendBtnDisabled }"
            >
              重新回答
            </span>
            <div
              class="operate-icon-box"
              :class="{ disabled: sendBtnDisabled }"
            >
              <!-- 聊天内容复制 -->
              <el-icon @click="copyRecord(item, index)">
                <DocumentCopy />
              </el-icon>
              <!-- 删除聊天内容 -->
              <el-icon @click="deleteRecord(index)">
                <Delete />
              </el-icon>
            </div>
          </div>
        </div>
      </li>
    </ul>
    <!-- 文本发送区域 -->
    <div class="ai-chat-form-wrapper">
      <div class="ai-chat-form-box">
        <textarea
          v-model="problemText"
          :rows="4"
          placeholder="在此输入您想要了解的内容..."
          @keydown.enter.exact.prevent="sendQuestion"
          @keydown.enter.shift.exact.prevent="problemText += '\n'"
        >
        </textarea>
        <div class="chat-form-footer">
          <div class="btns">
            <span class="content-tips">
              {{ problemText.length }} / {{ maxCharCount }}
            </span>
            <span>
              <el-button
                type="primary"
                :disabled="sendBtnDisabled"
                @click="sendQuestion"
                >发送</el-button
              >
            </span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
<style lang="scss" scoped>
.ai-chat-view {
  display: flex;
  background-color: #fff;
  flex-direction: column;
  flex: 1;
  overflow: hidden;
  padding: 40px 15px;

  //对话列表
  .ai-chat-list {
    display: flex;
    flex: 1;
    flex-direction: column;
    overflow-y: auto;

    //消除浏览器滚动条
    &::-webkit-scrollbar {
      display: none;
    }

    //会话项
    .ai-chat-item {
      display: flex;
      align-items: flex-start;
      margin-bottom: 40px;
    }

    //会话头像
    .ai-chat-avatar {
      margin-right: 25px;
    }

    //会话盒子
    .ai-chat-content-box {
      padding: 16px 30px;

      //加载图标盒子
      .loading-icon-box {
        .el-icon {
          transform: translate(0, 0);
          animation: rotate 3s linear infinite;
        }

        @keyframes rotate {
          0% {
            transform: translate(0, 0) rotate(0deg);
          }

          100% {
            transform: translate(0, 0) rotate(360deg);
          }
        }
      }

      //会话列表初始化盒子
      &.init-box {
        width: 100%;
        background-color: #eff7ff;
        background-image: url("https://ydcqoss.ydcode.cn/static/officialhome/ai-chat-init-bg.png");
        background-repeat: no-repeat;
        background-size: cover;
        border-radius: 10px;

        .ai-chat-title {
          font-size: 1.125rem;
          color: #005fdb;
          margin-bottom: 1rem;
        }

        .ai-chat-text {
          font-size: 0.875rem;
          color: #666666;
          line-height: 1.8;
        }
      }

      //会话列表用户盒子
      &.user-box {
        background-color: #fff;
        line-height: 2;
        padding-left: 0;
        padding-top: 0;
      }

      //会话列表ai回复盒子
      &.assistant-box {
        border-radius: 10px;
        background: #eff7ff;
        width: 100%;
      }
    }

    //会话操作
    .ai-chat-operate {
      display: flex;
      justify-content: space-between;
      align-items: center;
      cursor: pointer;

      //重复回答
      .re-reply-btn {
        font-size: 14px;
        color: #005fdb;

        &.disabled {
          color: #ccc;
        }
      }

      //操作图标
      .operate-icon-box {
        display: flex;
        align-items: center;

        .el-icon {
          color: #005fdb;
          font-size: 16px;
          margin-left: 16px;
          cursor: pointer;
        }

        &.disabled .el-icon {
          color: #ccc;
        }
      }
    }
  }
}

//发送问题表达
.ai-chat-form-wrapper {
  background-color: #fff;
  padding-left: calc(40px + 25px);

  .ai-chat-form-box {
    border: 1px solid #005fdb;
    border-radius: 10px;
    position: relative;
  }

  //文本域
  textarea {
    width: calc(100vh - 4px); // 减去滚动条的宽度
    margin-top: 2px;
    padding: 0.5rem 6rem 1rem 1.25rem;
    border: none;
    outline: none;
    resize: none;
    border-radius: 10px;
    color: #666;

    &::-webkit-scrollbar {
      width: 3px;
    }
  }

  //发送问题表单footer
  .chat-form-footer {
    display: flex;
    justify-content: flex-end;
    margin-top: -5px;
    position: absolute;
    bottom: 1rem;
    right: 1rem;

    //内容数字提示
    .content-tips {
      margin-right: 1.25rem;
    }
  }
}
</style>
