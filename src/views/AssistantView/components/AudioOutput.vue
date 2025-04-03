<template>
  <div class="card">
    <h2 class="text-xl font-semibold text-center text-gray-700 mb-4">音频输出</h2>
    <div class="relative min-h-[50px]">
      <audio ref="audioPlayer"
             :id="audioElementId"
             @ended="handleAudioEnded"
             @error="handleError"
             @loadstart="$emit('audio-load-start')"
             @canplay="$emit('audio-can-play')"
             @play="handleAudioPlay"
             @pause="handleAudioPause"
             :muted="muted"
             class="hidden ai-audio">
      </audio>
      <div v-if="currentAudio" class="p-3 my-2 bg-blue-100 rounded-md text-center">
        正在播放: {{ currentAudio.text || '语音响应' }}
        <span v-if="muted" class="text-xs text-gray-500 ml-2">(数字人模式)</span>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'AudioOutput',
  props: {
    currentAudio: { type: Object, default: null },
    muted: { type: Boolean, default: false }
  },
  data() {
    return {
      audioElementId: 'assistant-audio-player',
      segmentIndex: 0
    }
  },
  emits: ['audio-ended', 'audio-error', 'audio-load-start', 'audio-can-play'],
  methods: {
    // 简化的事件派发逻辑
    dispatchAudioEvent(eventName, detail = {}) {
      try {
        window.dispatchEvent(new CustomEvent(eventName, { 
          bubbles: true, 
          detail: {...detail, sourceElementId: this.audioElementId, segmentIndex: this.segmentIndex} 
        }));
      } catch (error) {
        console.error(`AudioOutput: 事件派发失败:`, error);
      }
    },
    
    handleError(event) {
      const errorMsg = event.target.error ? event.target.error.message : '音频播放出错';
      this.$emit('audio-error', { message: errorMsg });
      this.dispatchAudioEvent('ai-audio-error', { error: errorMsg });
    },
    
    handleAudioPlay() {
      this.dispatchAudioEvent('ai-audio-play', { 
        audioUrl: this.currentAudio?.url || '',
        isSegmented: true 
      });
    },
    
    handleAudioPause() {
      this.dispatchAudioEvent('ai-audio-pause');
    },
    
    handleAudioEnded() {
      this.dispatchAudioEvent('ai-audio-ended');
      this.$emit('audio-ended');
      this.segmentIndex++;
    },
    
    play(url) {
      if (!this.$refs.audioPlayer) return;
      
      this.$refs.audioPlayer.src = url;
      this.$refs.audioPlayer.muted = this.muted;
      
      this.$refs.audioPlayer.play().catch(error => {
        console.error('播放音频失败:', error);
        this.$emit('audio-error', error);
      });
    }
  },
  watch: {
    currentAudio(newAudio, oldAudio) {
      if (!newAudio || !newAudio.url) return;
      
      // 重置段索引如果是新的音频URL
      if (!oldAudio || newAudio.url !== oldAudio.url) {
        this.segmentIndex = 0;
      }
      
      // 设置ID和播放音频
      this.audioElementId = newAudio.id || 'assistant-audio-' + Date.now();
      
      this.$nextTick(() => {
        if (this.$refs.audioPlayer) {
          this.$refs.audioPlayer.id = this.audioElementId;
          this.play(newAudio.url);
        }
      });
    }
  }
};
</script>

<style scoped>
/* 使用Tailwind替代了这些自定义样式 */
</style>
