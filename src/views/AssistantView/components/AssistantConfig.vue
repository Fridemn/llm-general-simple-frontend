<template>
  <div class="card">
    <h2 class="text-xl font-semibold text-center text-gray-700 mb-4">语音助手配置</h2>
    <div class="config-panel">
      <div class="config-row">
        <label for="model" class="text-gray-700">LLM模型:</label>
        <select id="model" :value="selectedModel" @change="$emit('update:selected-model', $event.target.value)"
                class="flex-1">
          <option v-for="model in availableModels" :key="model" :value="model">
            {{ model }}
          </option>
        </select>
      </div>
      <div class="config-row">
        <label for="history-id" class="text-gray-700">对话历史ID:</label>
        <input type="text" id="history-id" :value="historyId" @change="$emit('update:history-id', $event.target.value)" 
               class="flex-1" />
        <button id="create-history-btn" class="neutral ml-3" @click="$emit('create-new-history')" 
                :disabled="createHistoryBtnDisabled">创建新对话</button>
      </div>
      <div class="config-row">
        <label for="duration" class="text-gray-700">录音时长 (秒):</label>
        <input type="number" id="duration" min="1" max="60" :value="duration" 
               @input="$emit('update:duration', Number($event.target.value))" 
               class="w-24" />
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'AssistantConfig',
  props: {
    historyId: {
      type: String,
      default: '01d49649-003f-4587-a610-229ef6f9b9ad'
    },
    selectedModel: {
      type: String,
      default: ''
    },
    availableModels: {
      type: Array,
      default: () => []
    },
    createHistoryBtnDisabled: {
      type: Boolean,
      default: true
    },
    duration: {
      type: Number,
      default: 2
    }
  },
  emits: ['update:history-id', 'update:selected-model', 'update:duration', 'create-new-history'],
  mounted() {
    // 确保historyId有值，如果为空则使用默认值
    if (!this.historyId) {
      this.$emit('update:history-id', '01d49649-003f-4587-a610-229ef6f9b9ad');
    }
    
    if (!this.duration) {
      this.$emit('update:duration', 2);
    }
  }
};
</script>
