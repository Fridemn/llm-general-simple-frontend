<template>
  <div class="card">
    <h2 class="text-xl font-semibold text-center text-gray-700 mb-4">语音交互</h2>
    <div class="btn-group">
      <button id="startRecordBtn" class="primary" @click="$emit('start-record')" :disabled="startRecordBtnDisabled">开始录音</button>
      <button id="stopRecordBtn" class="secondary" @click="$emit('stop-record')" :disabled="stopRecordBtnDisabled">停止录音</button>
    </div>
    <div class="w-full h-5 bg-gray-200 rounded-md overflow-hidden mt-4">
      <div class="h-full bg-green-600 text-center text-white text-xs leading-5 transition-all duration-300" 
           :style="{ width: progressWidth }">{{ progressText }}</div>
    </div>
    <div id="statusDisplay" class="status mt-2 text-gray-700">{{ statusText }}</div>
  </div>
</template>

<script>
export default {
  name: 'VoiceInteraction',
  props: {
    startRecordBtnDisabled: {
      type: Boolean,
      default: true
    },
    stopRecordBtnDisabled: {
      type: Boolean,
      default: true
    },
    progressWidth: {
      type: String,
      default: '0%'
    },
    progressText: {
      type: String,
      default: '0%'
    },
    statusText: {
      type: String,
      default: '等待连接...'
    }
  },
  emits: ['start-record', 'stop-record']
};
</script>

<style scoped>
/* 使用Tailwind替代了这些自定义样式 */
</style>
