<script lang="ts" setup>
import { useRouter } from "vue-router";
import { ChatDotRound } from "@element-plus/icons-vue";
const router = useRouter();

// 点击 logo 的处理
const logoImgClickHandle = () => {
  window.location.href = "https://ccb.com";
};

// 点击导航菜单的处理
const navClickHandle = () => {
  router.push("/aiChat");
};

// 点击登录按钮的处理，可根据实际需求调整跳转逻辑
const loginClickHandle = () => {
  // 示例：跳转到登录页，实际路径按需改
  router.push("/login"); 
};
</script>

<template>
  <div class="page-header">
    <div class="page-header-wrapper">
      <!-- logo -->
      <el-image
        src="https://ccb.com/chn/fileDir/bg_image/2020031015231379393.png"
        class="logo-image"
        fit="cover"
        @click="logoImgClickHandle"
      />
      <!-- nav 及登录按钮容器 -->
      <div class="nav-and-login">
        <!-- nav -->
        <nav>
          <span class="nav-item" @click="navClickHandle">
            <el-icon>
              <ChatDotRound />
            </el-icon>
            <span>AI聊天助手</span>
          </span>
        </nav>
        <!-- 登录按钮 -->
        <el-button 
          class="login-btn" 
          @click="loginClickHandle"
        >
          登录
        </el-button>
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
.page-header {
  width: 100%;
  // 建议给个合适高度，比如 60px，之前 height: 100% 可能显示有问题，根据实际布局调整
  height: 60px; 
  display: flex;
  align-items: center;
  padding: 0 20px; 
  box-sizing: border-box;
  background-color: #fff; /* 可按需调整头部背景色 */
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1); /* 可选：加个阴影 */
}

.page-header-wrapper {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: space-between;
  max-width: 1200px; /* 限制内容宽度，居中 */
  margin: 0 auto;
}

// logo 样式
.logo-image {
  width: auto;
  height: 32px;
  cursor: pointer;
  transition: transform 0.2s ease;

  &:hover {
    transform: scale(1.05);
  }
}

// 导航和登录按钮容器
.nav-and-login {
  display: flex;
  align-items: center;
  gap: 16px; 
}

// 导航样式
nav {
  display: flex;
  align-items: center;

  .nav-item {
    display: flex;
    align-items: center;
    cursor: pointer;
    color: #333;
    font-size: 14px;
    padding: 8px 12px;
    border-radius: 4px;
    transition: all 0.2s ease;
    gap: 6px;

    &:hover {
      color: #409eff;
      background-color: #f5f7fa;
    }
  }

  .el-icon {
    width: 18px;
    height: 18px;
  }
}

// 登录按钮样式
.login-btn {
  background-color: #007bff; /* 蓝色底色，可按需调整色值 */
  color: #fff; /* 白色文字 */
  border: none; 
  border-radius: 20px; /* 大圆角，让按钮更圆润 */
  padding: 8px 16px; 
  font-size: 14px;
  cursor: pointer;
  transition: background-color 0.2s ease;

  &:hover {
    background-color: #0056b3; /*  hover 时加深蓝色 */
  }

  &:focus {
    outline: none;
    box-shadow: 0 0 0 2px rgba(0, 123, 255, 0.2); /* 可选：聚焦样式 */
  }
}
</style>