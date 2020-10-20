<template>
  <div>
    用户中心
  </div>
</template>

<script>
  export default {
    async mounted() {
      const ret = await this.$http.get('/user/info')
      console.log('ret', ret)
    },
  }
</script>

<style lang="stylus" scoped>

</style>