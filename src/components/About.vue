<template>
  <div>
    <a-modal v-model="about_flag" width="420px" :footer="null">
      <div slot="title"><a-icon type="check-circle" style="color: lightgreen; font-size: 22px"/> {{version}}</div>
      <div style="text-align: center">
        <p>您正在使用免费开源的 <a-tag color="green">RedisGo</a-tag></p>
        <p>如果觉得好用，不妨在  <a target="_blank" href="https://github.com/liuzhuoling2011/RedisGo"> <a-icon type="github" />Github</a> 给个 <a-icon type="star" theme="filled" style="color: sandybrown" /></p>
        <p>良好的关注度和参与度有助于开源项目的长远发展~</p>
      </div>
    </a-modal>
  </div>
</template>

<script>
export default {
  name: 'About',
  data() {
    return {
      version: 'RedisGo Web V3.0.0',
      about_flag: false,
      pic_src: 'http://gitee.com/liuzhuoling/pic-go/raw/master/RedisGo/%E6%9C%80%E6%87%82%E6%B8%AF%E9%99%A9%E7%9A%84%E7%A8%8B%E5%BA%8F%E7%8C%BF.jpg'
    }
  },
  methods: {
    showAbout(){
      this.about_flag = true
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>

</style>
