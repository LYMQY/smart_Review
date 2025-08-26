<template>
  <div class="login-container">
    <!-- 左侧品牌区域 -->
    <div class="login-brand">
      <div class="brand-content">
        <el-image
          src="https://ccb.com/chn/fileDir/bg_image/2020031015231379393.png"
          class="brand-logo"
          fit="contain"
        />
        <h2 class="brand-title">智能审核平台</h2>
        <p class="brand-desc">
          建行数据智能审核与更新平台
        </p>
        <div class="brand-bg-pattern"></div>
      </div>
    </div>

    <!-- 右侧登录表单区域 -->
    <div class="login-form-wrapper">
      <div class="login-form-container">
        <div class="login-header">
          <h1 class="login-title">欢迎登录</h1>
          <p class="login-subtitle">请输入账号信息登录系统</p>
        </div>

        <el-form
          ref="loginFormRef"
          :model="loginForm"
          :rules="loginRules"
          class="login-form"
          label-position="left"
        >
          <!-- 用户名输入 -->
          <el-form-item prop="username">
            <el-input
              v-model="loginForm.username"
              placeholder="请输入用户名"
              :prefix-icon="User"
              class="login-input"
              autocomplete="username"
            />
          </el-form-item>

          <!-- 密码输入 -->
          <el-form-item prop="password">
            <el-input
              v-model="loginForm.password"
              type="password"
              placeholder="请输入密码"
              :prefix-icon="Lock"
              :show-password="showPassword"
              @click:password="showPassword = !showPassword"
              class="login-input"
              autocomplete="current-password"
            />
          </el-form-item>

          <!-- 记住密码和忘记密码 -->
          <div class="form-actions">
            <el-checkbox v-model="rememberMe" class="remember-me">
              记住密码
            </el-checkbox>
            <el-link type="primary" class="forgot-password" @click="handleForgotPassword">
              忘记密码?
            </el-link>
          </div>

          <!-- 登录按钮 -->
          <el-form-item>
            <el-button
              type="primary"
              class="login-button"
              @click="handleLogin"
              :loading="loginLoading"
            >
              登录
            </el-button>
          </el-form-item>

          <!-- 其他登录方式 -->
          <div class="other-login-methods">
            <div class="divider">
              <span>更多信息</span>
            </div>
            <div class="social-login">
              <el-icon class="social-icon" @click="handleSocialLogin('user')">
                <User />
              </el-icon>
              <el-icon class="social-icon" @click="handleSocialLogin('promotion')">
                <Promotion />
              </el-icon>
              <el-icon class="social-icon" @click="handleSocialLogin('lock')">
                <Lock />
              </el-icon>
            </div>
          </div>
        </el-form>
      </div>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { ref, reactive } from 'vue';
import { useRouter } from 'vue-router';
import { ElForm, ElFormItem, ElInput, ElButton, ElCheckbox, ElLink, ElIcon, ElMessage } from 'element-plus';
import { User, Lock, Promotion } from '@element-plus/icons-vue'; // 删除 Wechat, QQ, Alipay

const router = useRouter();

// 表单相关
const loginFormRef = ref<InstanceType<typeof ElForm>>();
const loginLoading = ref(false);
const showPassword = ref(false);
const rememberMe = ref(true);

// 登录表单数据
const loginForm = reactive({
  username: '',
  password: ''
});

// 表单验证规则
const loginRules = reactive({
  username: [
    { required: true, message: '请输入用户名', trigger: 'blur' },
    { min: 3, max: 20, message: '用户名长度在 3 到 20 个字符', trigger: 'blur' }
  ],
  password: [
    { required: true, message: '请输入密码', trigger: 'blur' },
    { min: 6, max: 30, message: '密码长度在 6 到 30 个字符', trigger: 'blur' }
  ]
});

// 处理登录
const handleLogin = async () => {
  if (!loginFormRef.value) return;
  
  try {
    // 表单验证
    await loginFormRef.value.validate();
    loginLoading.value = true;
    
    // 模拟登录请求
    setTimeout(() => {
      // 实际项目中这里会调用登录API
      console.log('登录信息:', loginForm);
      
      // 登录成功处理
      ElMessage.success('登录成功');
      loginLoading.value = false;
      
      // 跳转到首页或之前的页面
      router.push('/aiChat');
    }, 1500);
  } catch (error) {
    console.error('表单验证失败:', error);
    loginLoading.value = false;
  }
};

// 处理忘记密码
const handleForgotPassword = () => {
  ElMessage.info('即将跳转到密码重置页面');
  // 实际项目中这里会跳转到忘记密码页面
  // router.push('/forgot-password');
};

// 处理社交登录
const handleSocialLogin = (type: string) => {
  ElMessage.info(`即将使用${getSocialName(type)}登录`);
  // 实际项目中这里会处理对应社交平台的登录逻辑
};

// 获取社交平台名称
const getSocialName = (type: string) => {
  const names = {
    'user': '用户',
    'promotion': '推广',
    'lock': '安全'
  };
  return names[type] || type;
};
</script>

<style lang="scss" scoped>
// 主容器样式
.login-container {
  display: flex;
  min-height: 100vh;
  width: 100%;
  overflow: hidden;
  background: linear-gradient(120deg, #e0e7ff 0%, #f5f7fa 100%);
  position: relative;
}

// 左侧品牌区域样式
.login-brand {
  flex: 1;
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0 50px;
  overflow: hidden;
  min-width: 350px;
  // 设置背景为总部照片
  background: url('/back.png') center center / cover no-repeat;
  color: #fff;

  // 渐变遮罩层
  &::before {
    content: '';
    position: absolute;
    inset: 0;
    background: linear-gradient(135deg, rgba(0,82,217,0.35) 0%, rgba(0,102,255,0.15) 100%);
    z-index: 1;
    pointer-events: none;
  }

  .brand-content {
    position: relative;
    z-index: 2;
    max-width: 500px;
    background: rgba(255,255,255,0.10);
    border-radius: 24px;
    padding: 40px 32px;
    box-shadow: 0 8px 32px rgba(0,82,217,0.12);
    backdrop-filter: blur(4px);
    animation: fadeInLeft 1s ease-out;
  }

  .brand-logo {
    height: 60px;
    margin-bottom: 40px;
    filter: drop-shadow(0 2px 8px rgba(0,0,0,0.08));
  }

  .brand-title {
    font-size: 36px;
    font-weight: 700;
    margin-bottom: 20px;
    line-height: 1.3;
    letter-spacing: 2px;
    text-shadow: 0 2px 8px rgba(0,0,0,0.18);
  }

  .brand-desc {
    font-size: 18px;
    opacity: 0.96;
    line-height: 1.6;
    margin-bottom: 16px;
    text-shadow: 0 1px 4px rgba(0,0,0,0.12);
  }

  .brand-bg-pattern {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-image: url("data:image/svg+xml,%3Csvg width='60' height='60' viewBox='0 0 60 60' xmlns='http://www.w3.org/2000/svg'%3E%3Cg fill='none' fill-rule='evenodd'%3E%3Cg fill='%23ffffff' fill-opacity='0.05'%3E%3Cpath d='M36 34v-4h-2v4h-4v2h4v4h2v-4h4v-2h-4zm0-30V0h-2v4h-4v2h4v4h2V6h4V4h-4zM6 34v-4H4v4H0v2h4v4h2v-4h4v-2H6zM6 4V0H4v4H0v2h4v4h2V6h4V4H6z'/%3E%3C/g%3E%3C/g%3E%3C/svg%3E");
    z-index: 0;
    opacity: 0.18;
    pointer-events: none;
  }
}

// 右侧登录表单区域样式
.login-form-wrapper {
  flex: 0 0 450px;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 40px;
  background: rgba(255,255,255,0.7);
  backdrop-filter: blur(4px);

  .login-form-container {
    width: 100%;
    max-width: 380px;
    animation: fadeInRight 1s ease-out;
    background: #fff;
    border-radius: 24px;
    box-shadow: 0 8px 32px rgba(0,82,217,0.10);
    padding: 40px 32px 32px 32px;
  }

  .login-header {
    margin-bottom: 40px;
    text-align: center;
  }

  .login-title {
    font-size: 28px;
    font-weight: 700;
    color: #1d2129;
    margin-bottom: 8px;
    letter-spacing: 1px;
  }

  .login-subtitle {
    font-size: 15px;
    color: #86909c;
    margin-bottom: 8px;
  }

  .login-form {
    width: 100%;
  }

  .login-input {
    height: 48px;
    border-radius: 12px;
    font-size: 16px;
    transition: all 0.3s;
    background: #f5f7fa;
    border: 1px solid #e5e6eb;

    &:focus {
      border-color: #0066ff;
      box-shadow: 0 0 0 2px rgba(0, 102, 255, 0.15);
      background: #fff;
    }
  }

  .form-actions {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 24px;
    font-size: 14px;
  }

  .remember-me {
    color: #4e5969;
  }

  .forgot-password {
    font-size: 14px;
    color: #0066ff;
    cursor: pointer;
    transition: color 0.2s;
    &:hover {
      color: #0052d9;
      text-decoration: underline;
    }
  }

  .login-button {
    width: 100%;
    height: 48px;
    font-size: 17px;
    font-weight: 600;
    background: linear-gradient(90deg, #0066ff 0%, #409eff 100%);
    border: none;
    border-radius: 12px;
    box-shadow: 0 2px 8px rgba(0,102,255,0.08);
    transition: background 0.3s, box-shadow 0.3s;

    &:hover, &:focus {
      background: linear-gradient(90deg, #0052d9 0%, #409eff 100%);
      box-shadow: 0 4px 16px rgba(0,102,255,0.12);
    }
  }

  .other-login-methods {
    margin-top: 30px;
  }

  .divider {
    display: flex;
    align-items: center;
    margin-bottom: 20px;
    color: #86909c;
    font-size: 14px;

    &::before, &::after {
      content: '';
      flex: 1;
      height: 1px;
      background-color: #e5e6eb;
    }

    span {
      padding: 0 16px;
    }
  }

  .social-login {
    display: flex;
    justify-content: center;
    gap: 24px;
  }

  .social-icon {
    width: 44px;
    height: 44px;
    padding: 8px;
    border-radius: 50%;
    background: linear-gradient(135deg, #e0e7ff 0%, #f5f7fa 100%);
    color: #409eff;
    cursor: pointer;
    font-size: 24px;
    box-shadow: 0 2px 8px rgba(64,158,255,0.08);
    transition: all 0.3s;

    &:hover {
      background: linear-gradient(135deg, #409eff 0%, #e0e7ff 100%);
      color: #fff;
      transform: translateY(-3px) scale(1.08);
      box-shadow: 0 4px 16px rgba(64,158,255,0.16);
    }
  }
}

// 动画效果
@keyframes fadeInLeft {
  from {
    opacity: 0;
    transform: translateX(-50px);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

@keyframes fadeInRight {
  from {
    opacity: 0;
    transform: translateX(50px);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

// 响应式调整
@media (max-width: 900px) {
  .login-brand {
    display: none;
  }

  .login-form-wrapper {
    flex: 1;
    padding: 24px;
    background: #fff;
  }
}

@media (max-width: 480px) {
  .login-form-wrapper {
    padding: 10px;
  }

  .login-form-container {
    max-width: 100%;
    padding: 24px 8px 16px 8px;
    border-radius: 12px;
  }
}
</style>
