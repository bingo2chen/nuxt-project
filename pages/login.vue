<template>
  <div class="login-container">
    <el-form :rules="rules" label-width="100px" ref="loginForm"  class="login-form" :model="form">
      <div class="title-container">
        <logo />
      </div>
      <el-form-item prop="email" label="邮箱">
        <el-input placeholder="请输入邮箱" v-model="form.email"></el-input>
      </el-form-item>
      <el-form-item prop="captcha" label="验证码" class="captcha-container">
        <div class="captcha">
          <img :src="code.captcha" alt="" @click="resetCaptcha">
        </div>
        <el-input placeholder="请输入验证码" v-model="form.captcha"></el-input>
      </el-form-item>
      <el-form-item prop="emailcode" label="邮箱验证码" class="captcha-container">
        <div class="captcha">
          <el-button type="primary" @click="sendEmailCode" :disabled="send.timer>0">{{sendText}}</el-button>
        </div>
        <el-input placeholder="请输入邮箱验证码" v-model="form.captcha"></el-input>
      </el-form-item>
      <el-form-item prop="passwd" label="密码">
        <el-input placeholder="请输入密码" v-model="form.passwd"></el-input>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" @click.native.prevent="handleLogin">登录</el-button>
      </el-form-item>
    </el-form>
  </div>
</template>

<script>
  import Logo from '~/components/Logo.vue'
  import md5 from 'md5'
  export default {
    layout: 'login',
    components: {
      Logo,
    },
    computed: {
      sendText() {
        if (this.send.timer <= 0) {
          return '发送'
        }
        return `${this.send.timer}s后发送`
      }
    },
    data() {
      return {
        send: {
          timer: 0,
        },
        code: {
          captcha: '/api/captcha'
        },
        rules: {
          email: [
            { required: true, message: '请输入邮箱' },
            { type: 'email', message: '请输入正确的邮箱格式' },
          ],
          captcha: [
            { required: true, message: '请输入验证码' },
          ],
          emailcode: [
            { required: true, message: '请输入邮件验证码' },
          ],
          passwd: [
            { required: true, pattern: /^[\w_-]{6,12}$/g, message: '请输入6~12位密码' },
          ],
        },
        form: {
          email: '1018513521@qq.com',
          captcha: '',
          passwd: '123456',
          repasswd: '123456',
          nickname: 'edcwwb',
        }
      }
    },
    methods: {
      async sendEmailCode() {
        await this.$http.get('/sendcode?email=' + this.form.email)
        
        //todo
        this.send.timer = 10
        this.timer = setInterval(() => {
          this.send.timer -= 1
          if (this.send.timer === 0) {
            clearInterval(this.timer)
          }
        }, 1000);
      },
      handleLogin() {
        this.$refs.loginForm.validate(async (valid) => {
          if (valid) {
            const payload = {
              email: this.form.email,
              captcha: this.form.captcha,
              passwd: md5(this.form.passwd),
            }
            const ret = await this.$http.post('/user/login', payload)
            if (ret.code === 0) {
              // token存储，登录成功，返回token
              this.$message.success('登录成功')
              setTimeout(() => {
                this.$router.push('/login') 
              }, 500);
            } else {
              // this.$message.error(ret.message)
            }
            
          } else {
            console.log('校验失败');
            
          }
        })
      },
      resetCaptcha() {
        this.code.captcha = '/api/captcha?_t' + new Date().getTime()
      },
    },
  }
</script>

<style lang="stylus" scoped>

</style>