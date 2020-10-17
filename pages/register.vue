<template>
  <div class="login-container">
    <el-form :rules="rules" label-width="100px" ref="registerForm"  class="login-form" :model="form">
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
        <el-input width="150px" placeholder="请输入验证码" v-model="form.captcha"></el-input>
      </el-form-item>
      <el-form-item prop="nickname" label="昵称">
        <el-input placeholder="请输入昵称" v-model="form.nickname"></el-input>
      </el-form-item>
      <el-form-item prop="passwd" label="密码">
        <el-input placeholder="请输入密码" v-model="form.passwd"></el-input>
      </el-form-item>
      <el-form-item prop="repasswd" label="确认密码">
        <el-input placeholder="请再输入密码" v-model="form.repasswd"></el-input>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" @click.native.prevent="handleRegister">注册</el-button>
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
    data() {
      return {
        code: {
          captcha: '/api/captcha'
        },
        rules: {
          email: [
            { required: true, message: '请输入邮箱' },
            { type: 'email', message: '请输入正确的邮箱格式' },
          ],
          nickname: [
            { required: true, message: '请输入昵称' },
          ],
          captche: [
            { required: true, message: '请输入验证码' },
          ],
          passwd: [
            { required: true, pattern: /^[\w_-]{6,12}$/g, message: '请输入6~12位密码' },
          ],
          repasswd: [
            { required: true, message: '请输入邮箱' },
            { 
              validator: (rules, value, callback) => {
                if (value !== this.form.passwd) {
                  callback(new Error('两次密码不一致'))
                }
                callback()
              }
            }
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
      handleRegister() {
        this.$refs.registerForm.validate(async (valid) => {
          if (valid) {
            console.log('校验成功');
            const payload = {
              email: this.form.email,
              nickname: this.form.nickname,
              captcha: this.form.captcha,
              passwd: md5(this.form.passwd),
              repasswd: md5(this.form.repasswd),
            }
            const ret = await this.$http.post('/user/register', payload)
            if (ret.data.code === 0) {
              this.$alert('注册成功', '成功', {
                confirmButtonText: '确定',
                callback: action => {
                  this.$router.push('/login')
                }
              })
            } else {
              // this.$message.error(ret.message)
            }
            
          } else {
            console.log('校验失败');
            
          }
        })
      },
      resetCaptcha() {
        this.code.captche = '/api/captcha?_t' + new Date().getTime()
      },
    },
  }
</script>

<style lang="stylus" scoped>

</style>