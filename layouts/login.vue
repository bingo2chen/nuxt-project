<template>
  <div>
    <!-- <div>导航栏</div> -->
    <nuxt></nuxt>
    <!-- <div>底部区域</div> -->
  </div>
</template>

<script>
  export default {
    
  }
</script>

<style lang="stylus">
.login-container
  width 100%
  min-height 100%
  .login-form
    width 520px
    margin 0 auto
    padding 160px 0
    overflow hidden
  .title-container
    text-align center
    margin-bottom 20px
    img
      width 200px
  .captcha-container
    position relative
    width 340px
    .captcha
      position absolute
      right -110px
      img
        width 90px
        height 50px
        cursor pointer
      button
        width 90px
        height 40px
        padding 0
</style>