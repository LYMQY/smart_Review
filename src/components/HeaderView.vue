<script lang="ts" setup>
import { useRouter } from "vue-router";
const router = useRouter();
//点击logo 的处理
const logoImgClickHandle = () => {
  window.location.href = "https://www.xaiu.edu.cn/";
};
//点击导航菜单 的处理
const navClickHandle = () => {
  router.push("/aiChat");
};
</script>
<template>
  <div class="page-header">
    <div class="page-header-wrapper">
      <!-- logo -->
      <el-image
        src="https://5b0988e595225.cdn.sohucs.com/images/20190916/577ec8cdff7840918d9d1b03048c7dfd.jpeg"
        class="logo-image"
        fit="cover"
        @click="logoImgClickHandle"
      />
      <!-- nav -->
      <nav>
        <span class="nav-item" @click="navClickHandle">
          <el-icon>
            <ChatDotRound />
          </el-icon>
          <span>AI聊天助手</span>
        </span>
      </nav>
    </div>
  </div>
</template>
<style lang="scss" scoped>
.page-header {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;

  .page-header-wrapper {
    flex: 1;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
  }

  //logo
  .logo-image {
    width: auto;
    height: 28px;
    cursor: pointer;
  }

  //nav
  nav {
    display: flex;
    align-items: center;

    .nav-item {
      display: flex;
      align-items: center;
      cursor: pointer;
    }

    i {
      width: 24px;
      height: 24px;
    }
    &:hover {
      color: #409eff;
    }
  }
}
</style>
