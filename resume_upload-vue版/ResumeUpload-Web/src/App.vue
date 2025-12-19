<template>
  <!-- 根组件仅作为容器，引入简历上传组件 -->
  <ResumeUpload />
</template>

<script>
import ResumeUpload from './components/ResumeUpload.vue'

export default {
  components: {
    ResumeUpload
  }
}
</script>

<style>
/* 全局样式 */
body {
  position: relative;
  margin: 0;
  font-family: Arial, sans-serif;
}
</style>
