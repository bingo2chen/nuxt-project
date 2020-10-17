<template>
  <div>
    <el-form class="login-form" :rules="rules">
      <el-form-item prop="email">
        <span class="el-icon-mobile"></span>
        <el-input placeholder="邮箱"></el-input>
      </el-form-item>
      <el-form-item prop="passwd">
        <span class="el-icon-lock"></span>
        <el-input placeholder="密码"></el-input>
      </el-form-item>
      <el-form-item prop="captcha">
        <el-input placeholder="验证码"></el-input>
        <img @click="updateCaptcha" :src="captchaUrl" alt="">
      </el-form-item>
    </el-form>
  </div>
</template>

<script>
  export default {
    layout: 'login',
    data() {
      return {
        rules: {
          email: [
            { required: true, message: '请输入邮箱' },
            { type: 'email', message: '请输入正确的邮箱格式' },
          ],
          captche: [
            { required: true, message: '请输入验证码' },
          ],
          passwd: [
            { required: true, pattern: /^[w_-]{6,12}&/g, message: '请输入6~12位密码' },
          ],
        },
        captchaUrl: '/api/captcha?_t=' + new Date().getTime()
      }
    },
    methods: {
      updateCaptcha() {
        this.captchaUrl = '/api/captcha?_t=' + new Date().getTime()
      }
    },
  }
</script>

<style lang="stylus" scoped>
.login-form 
  width: 800px
  margin: 0 auto
</style>