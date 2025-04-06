<template>
  <div class="max-w-4xl mx-auto px-4 py-8">
    <h1 class="text-3xl font-bold text-center text-gray-800 mb-8">语音助手演示</h1>
    
    <ConnectionSettings 
      :api-url="apiUrl"
      :is-connected="isConnected"
      :connection-status-text="connectionStatusText"
      :connection-status-class="connectionStatusClass"
      :connect-btn-disabled="connectBtnDisabled"
      :disconnect-btn-disabled="disconnectBtnDisabled"
      @update:api-url="apiUrl = $event"
      @connect="connectWebSocket"
      @disconnect="disconnectWebSocket"
    />
    
    <AssistantConfig
      :history-id="historyId"
      :selected-model="selectedModel"
      :available-models="availableModels"
      :create-history-btn-disabled="createHistoryBtnDisabled"
      @update:history-id="historyId = $event; onHistoryIdChange()"
      @update:selected-model="selectedModel = $event; setSessionParams()"
      @create-new-history="createNewHistory"
    />
    
    <TTSSettings
      :provider="provider"
      :api-base="apiBase"
      :character="character"
      :emotion="emotion"
      @update:provider="provider = $event; setSessionParams()"
      @update:api-base="apiBase = $event; setSessionParams()"
      @update:character="character = $event; setSessionParams()"
      @update:emotion="emotion = $event; setSessionParams()"
    />
    
    <VoiceInteraction
      :start-record-btn-disabled="startRecordBtnDisabled"
      :stop-record-btn-disabled="stopRecordBtnDisabled"
      :progress-width="progressWidth"
      :progress-text="progressText"
      :status-text="statusText"
      :is-voice-chatting="isRecording"
      :is-speaking="isSpeaking"
      @start-record="startVoiceChat"
      @stop-record="stopVoiceChat"
    />
    
    <ConversationHistory :conversation-history="conversationHistory" />
    
    <AudioOutput 
      ref="audioPlayerComponent"
      :current-audio="currentAudio"
      :audio-queue="audioQueue"
      :muted="shouldMuteLocalAudio"
      @audio-ended="onAudioEnded"
      @audio-error="onAudioError"
      @audio-load-start="onAudioLoadStart"
      @audio-can-play="onAudioCanPlay"
    />
    
    <LogPanel :logs="logs" />
  </div>
</template>

<script>
import { ref, onMounted, computed, inject, onUnmounted } from 'vue'; // 添加onUnmounted引入
import ConnectionSettings from './components/ConnectionSettings.vue';
import AssistantConfig from './components/AssistantConfig.vue';
import TTSSettings from './components/TTSSettings.vue';
import VoiceInteraction from './components/VoiceInteraction.vue';
import ConversationHistory from './components/ConversationHistory.vue';
import AudioOutput from './components/AudioOutput.vue';
import LogPanel from './components/LogPanel.vue';

export default {
  name: 'HomeView',
  components: {
    ConnectionSettings,
    AssistantConfig,
    TTSSettings,
    VoiceInteraction,
    ConversationHistory,
    AudioOutput,
    LogPanel
  },
  setup() {
    // 获取Live2D控制接口
    const live2dControls = inject('live2dControls');

    // Reactive data
    const socket = ref(null);
    const clientId = ref(null);
    const isRecording = ref(false);
    const isConnected = ref(false);
    const isProcessingLlm = ref(false);
    const historyId = ref('');
    const availableModels = ref([]);
    const recordingInterval = ref(null);
    const currentStreamMessage = ref(null);
    const currentStreamBuffer = ref('');
    const heartbeatInterval = ref(null);
    const lastServerActivity = ref(Date.now());
    const isSpeaking = ref(false); // 添加是否正在说话的状态

    // 添加重连控制
    const maxReconnectAttempts = ref(3);
    const reconnectCount = ref(0);
    const reconnectDelay = ref(1000); // 初始延迟1秒

    const apiUrl = ref('ws://localhost:8000/ws/realtime-voice-chat'); // 修改默认URL
    const selectedModel = ref('');
    const provider = ref('gsvi');
    const apiBase = ref('http://127.0.0.1:5000');
    const character = ref('');
    const emotion = ref('');
    const statusText = ref('等待连接...');
    const connectionStatusText = ref('未连接');
    const conversationHistory = ref([{ sender: 'assistant', text: '您好，我是语音助手，请点击"连接服务器"开始对话。' }]);
    const logs = ref([]);
    const audioQueue = ref([]);
    const isPlaying = ref(false);
    const connectionStatusClass = ref('disconnected');
    const progressWidth = ref('0%');
    const progressText = ref('0%');
    const audioPlayer = ref(null);
    const currentAudio = ref(null);
    
    // Computed properties for button states
    const connectBtnDisabled = computed(() => isConnected.value);
    const disconnectBtnDisabled = computed(() => !isConnected.value);
    const createHistoryBtnDisabled = computed(() => !isConnected.value);
    const startRecordBtnDisabled = computed(() => {
      return !(isConnected.value && historyId.value && selectedModel.value) || isRecording.value;
    });
    const stopRecordBtnDisabled = computed(() => !isRecording.value);

    // 计算属性：当Live2D可见时，本地音频应该静音
    const shouldMuteLocalAudio = computed(() => {
      return !!(live2dControls && live2dControls.isVisible.value);
    });

    // WebSocket connection function with reconnect logic
    const connectWebSocket = () => {
      try {
        const wsUrl = apiUrl.value.trim();

        if (!wsUrl.startsWith('ws://') && !wsUrl.startsWith('wss://')) {
          addToLog("错误：WebSocket URL必须以ws://或wss://开头");
          updateConnectionStatus("URL错误", 'disconnected');
          return;
        }

        updateStatus('正在连接服务器...');
        updateConnectionStatus('正在连接...', 'processing');
        addToLog(`尝试连接到: ${wsUrl}`);

        // 确保先关闭之前的连接
        if (socket.value) {
          socket.value.close();
          socket.value = null;
        }

        socket.value = new WebSocket(wsUrl);
        
        // 设置连接超时
        const connectionTimeout = setTimeout(() => {
          if (socket.value && socket.value.readyState !== WebSocket.OPEN) {
            socket.value.close();
            handleConnectionError('连接超时');
          }
        }, 5000);
        
        // 添加更详细的错误处理
        socket.value.addEventListener('error', (event) => {
            console.error('WebSocket Error:', event);
            addToLog(`WebSocket详细错误信息: ${JSON.stringify(event)}`);
        });

        socket.value.onopen = () => {
            clearTimeout(connectionTimeout);
            console.log('WebSocket连接已打开');
            addToLog('WebSocket连接已打开');
            updateStatus('已连接到服务器');
            updateConnectionStatus('已连接', 'connected');
            isConnected.value = true;
            reconnectCount.value = 0;  // 重置重连计数
            fetchAvailableModels();
            startHeartbeat();
        };

        socket.value.onclose = handleDisconnect;
        socket.value.onerror = handleConnectionError;
        socket.value.onmessage = handleServerMessage;

      } catch (error) {
        console.error('连接错误:', error);
        addToLog(`详细错误信息: ${error.toString()}`);
        updateStatus(`连接失败: ${error.message}`);
        updateConnectionStatus('连接失败', 'disconnected');
      }
    };

    // 处理连接关闭
    const handleDisconnect = (event) => {
      stopHeartbeat();
      isConnected.value = false;
      isRecording.value = false;
      isSpeaking.value = false;
      
      const reason = event.reason || '未知原因';
      updateConnectionStatus('未连接', 'disconnected');
      updateStatus('连接已关闭');
      addToLog(`WebSocket连接已关闭 - 代码: ${event.code}, 原因: ${reason}`);
      
      // 如果不是用户主动关闭，尝试重连
      if (event.code !== 1000 && reconnectCount.value < maxReconnectAttempts.value) {
        reconnectCount.value++;
        const delay = reconnectDelay.value * reconnectCount.value;
        addToLog(`将在 ${delay/1000} 秒后尝试重连(${reconnectCount.value}/${maxReconnectAttempts.value})...`);
        
        setTimeout(() => {
          connectWebSocket();
        }, delay);
      }
    };

    // 处理连接错误
    const handleConnectionError = (error) => {
      console.error('WebSocket Error:', error);
      addToLog(`WebSocket错误: ${error}`);
      
      if (!isConnected.value) {
        return; // 让 onclose 处理重试
      }
      
      updateStatus('连接错误');
      updateConnectionStatus('连接错误', 'disconnected');
      
      // 尝试重连
      if (reconnectCount.value < maxReconnectAttempts.value) {
        reconnectCount.value++;
        const delay = reconnectDelay.value * reconnectCount.value;
        addToLog(`将在 ${delay/1000} 秒后尝试重连(${reconnectCount.value}/${maxReconnectAttempts.value})...`);
        
        setTimeout(() => {
          connectWebSocket();
        }, delay);
      }
    };

    // WebSocket disconnection function
    const disconnectWebSocket = () => {
      stopHeartbeat();

      if (socket.value && socket.value.readyState === WebSocket.OPEN) {
        addToLog('断开连接');
        socket.value.close(1000, '用户主动断开');
        updateStatus('已断开连接');
        updateConnectionStatus('已断开', 'disconnected');
      }
      resetConnectionState();
    };

    // Update connection status
    const updateConnectionStatus = (message, status) => {
      connectionStatusText.value = message;
      connectionStatusClass.value = status;
    };

    // Reset connection state
    const resetConnectionState = () => {
      isConnected.value = false;
      if (isRecording.value) stopVoiceChat();

      progressWidth.value = '0%';
      progressText.value = '0%';
    };

    // Set session parameters
    const setSessionParams = () => {
      if (!socket.value || socket.value.readyState !== WebSocket.OPEN) {
        addToLog('WebSocket未连接');
        return;
      }

      if (!historyId.value) {
        addToLog('警告: 未设置对话历史ID');
        return;
      }

      const model = selectedModel.value;
      if (!model) {
        addToLog('警告: 未选择模型');
        return;
      }

      const params = {
        command: 'set_params',
        model: model,
        history_id: historyId.value,
        user_id: 'demo_user',
        provider: provider.value,
        api_base: apiBase.value
      };

      if (character.value.trim()) {
        params.character = character.value.trim();
      }

      if (emotion.value.trim()) {
        params.emotion = emotion.value.trim();
      }

      socket.value.send(JSON.stringify(params));
      addToLog('已更新会话参数');
    };

    // Fetch available models
    const fetchAvailableModels = async () => {
      try {
        const baseUrl = window.location.protocol + '//' + window.location.hostname + ':8000';
        addToLog(`正在从 ${baseUrl}/llm/available-models 获取模型列表...`);

        const response = await fetch(`${baseUrl}/llm/available-models`);

        if (response.ok) {
          const data = await response.json();
          const models = data['available-models'];

          availableModels.value = models;

          selectedModel.value = models.length > 0 ? models[0] : '';

          if (models.length > 0) {
            addToLog(`已加载 ${models.length} 个可用模型`);
          } else {
            addToLog('未找到可用模型');
          }
        } else {
          addToLog(`获取模型失败: ${response.status} ${response.statusText}`);
        }
      } catch (error) {
        addToLog(`获取模型出错: ${error}`);
      }
    };

    // Create new history
    const createNewHistory = async () => {
      try {
        const baseUrl = window.location.protocol + '//' + window.location.hostname + ':8000';
        addToLog(`正在请求 ${baseUrl}/llm/history/new 创建新对话...`);

        const response = await fetch(`${baseUrl}/llm/history/new`, {
          method: 'POST'
        });

        if (response.ok) {
          const data = await response.json();
          historyId.value = data.history_id;

          addToLog(`创建了新对话，ID: ${historyId.value}`);

          setSessionParams();
        } else {
          addToLog(`创建对话失败: ${response.status} ${response.statusText}`);
        }
      } catch (error) {
        addToLog(`创建对话出错: ${error}`);
      }
    };

    // Start voice chat - 修改为实时语音对话
    const startVoiceChat = () => {
      if (!socket.value || socket.value.readyState !== WebSocket.OPEN) {
        addToLog('WebSocket未连接');
        return;
      }

      if (!historyId.value) {
        addToLog('请先创建一个对话或输入对话历史ID');
        statusText.value = '请先创建一个对话或输入对话历史ID';
        return;
      }

      if (!selectedModel.value) {
        addToLog('请选择一个模型');
        statusText.value = '请选择一个模型';
        return;
      }

      setSessionParams();

      updateStatus('开始实时语音对话...');

      socket.value.send(JSON.stringify({
        command: 'start',
        server_url: "ws://127.0.0.1:8765"
      }));

      isRecording.value = true;
      addToLog('开始实时语音对话');
    };

    // Stop recording - 修改为停止实时对话
    const stopVoiceChat = () => {
      if (!socket.value || socket.value.readyState !== WebSocket.OPEN || !isRecording.value) {
        return;
      }

      addToLog('停止实时语音对话');
      socket.value.send(JSON.stringify({ command: 'stop' }));

      isRecording.value = false;
      isSpeaking.value = false;
      updateStatus('已停止语音对话');
    };

    // Handle server message - 更新为适应实时语音对话
    const handleServerMessage = async (event) => {
      try {
        lastServerActivity.value = Date.now();

        const message = JSON.parse(event.data);

        if (message.pong === true || message.type === 'ping' ||
          message.type === 'keep_alive' || message.type === 'audio_ready') {
          console.log('收到心跳响应');
          return;
        }

        const logContent = message.content ?
          `${JSON.stringify(message).substring(0, 100)}...` :
          JSON.stringify(message).substring(0, 150);
        addToLog(`收到: ${logContent}`);

        if (message.type === 'connection' && message.client_id) {
          clientId.value = message.client_id;
          addToLog(`已分配客户端ID: ${clientId.value}`);
        }

        switch (message.type) {
          case 'connection':
            updateStatus(`已连接: ${message.message}`);
            break;

          case 'params':
          case 'config_success':
            if (message.status === "updated" || message.message) {
              addToLog('配置已更新');
            }
            break;

          case 'start':
            if (message.status === 'success') {
              updateStatus('实时语音对话已启动');
            }
            break;

          case 'stop':
            if (message.status === 'success') {
              isRecording.value = false;
              updateStatus('实时语音对话已停止');
            }
            break;

          case 'voice_activity':
            handleVoiceActivity(message);
            break;

          case 'recognition_result':
            handleRecognitionResult(message);
            break;

          case 'llm_stream_chunk':
            handleLLMStreamChunk(message);
            break;
          
          case 'llm_stream_start':
            updateStatus('LLM处理中...');
            isProcessingLlm.value = true;

            if (!document.querySelector('.assistant-response-building')) {
              addMessageToConversation('', 'assistant', 'assistant-response-building');
            }

            currentStreamBuffer.value = '';
            break;

          case 'llm_stream_end':
            updateStatus('LLM处理完成，准备生成语音...');
            isProcessingLlm.value = false;

            const buildingResponse = document.querySelector('.assistant-response-building');
            if (buildingResponse) {
              buildingResponse.classList.remove('assistant-response-building');
            }

            if (message.history_id && (!historyId.value || historyId.value !== message.history_id)) {
              historyId.value = message.history_id;
              addToLog(`更新对话历史ID: ${historyId.value}`);
            }
            break;

          case 'tts_start':
            updateStatus('正在生成语音...');
            addToLog('TTS处理开始');
            break;

          case 'tts_sentence_complete':
            handleTTSSentenceComplete(message);
            break;

          case 'error':
            updateStatus(`错误: ${message.message}`);
            addToLog(`错误: ${message.message}`);
            console.error('Server Error:', message.message);
            isProcessingLlm.value = false;
            isRecording.value = false;
            break;

          default:
            console.log('接收到未处理消息类型:', message);
        }

      } catch (error) {
        console.error('Error handling message:', error, event.data);
        addToLog(`处理消息出错: ${error}`);
      }
    };

    // 新增：处理语音活动检测
    const handleVoiceActivity = (message) => {
      if (message.is_speaking !== undefined) {
        isSpeaking.value = message.is_speaking;
      }
    };

    // 新增：处理语音识别结果
    const handleRecognitionResult = (message) => {
      const text = message.text || '';
      updateStatus(`已识别: ${text.substring(0, 30)}${text.length > 30 ? '...' : ''}`);
      addMessageToConversation(text, 'user');
      isSpeaking.value = false;
    };

    // 新增：处理LLM流式输出
    const handleLLMStreamChunk = (message) => {
      const content = message.content || message.chunk || '';
      if (content) {
        if (!document.querySelector('.assistant-response-building')) {
          addMessageToConversation('', 'assistant', 'assistant-response-building');
        }
        appendToLastMessage(content);
        currentStreamBuffer.value += content;
      }
    };

    // 新增：处理TTS句子完成
    const handleTTSSentenceComplete = (message) => {
      // 将新的音频添加到队列
      const fullUrl = window.location.protocol + '//' + window.location.hostname + ':8000' + message.audio_url;
      
      // 检查队列中是否已存在相同URL的音频
      if (audioQueue.value.some(audio => audio.url === fullUrl)) {
        addToLog(`跳过重复音频: ${message.audio_url}`);
        return;
      }
      
      addToLog(`收到TTS音频: ${message.text.substring(0, 20)}...`);
      audioQueue.value.push({
        url: fullUrl,
        text: message.text
      });
      
      // 如果没有正在播放，开始播放
      if (!isPlaying.value) {
        playNextAudio();
      }
    };

    // Update status
    const updateStatus = (message) => {
      statusText.value = message;
      console.log('Status:', message);
    };

    // Add message to conversation
    const addMessageToConversation = (text, sender, className = '') => {
      conversationHistory.value.push({ sender: sender, text: text, className: className });
    };

    // Append to last message
    const appendToLastMessage = (text) => {
      const lastMessage = conversationHistory.value.find(message => message.className === 'assistant-response-building');
      if (lastMessage) {
        lastMessage.text += text;
      }
    };

    // Play audio response
    const playAudioResponse = (url) => {
      const baseUrl = window.location.protocol + '//' + window.location.hostname + ':8000';
      const fullUrl = baseUrl + url;
      
      // 检查队列中是否已存在相同URL的音频
      if (audioQueue.value.some(audio => audio.url === fullUrl)) {
          addToLog(`跳过重复音频: ${url}`);
          return;
      }
      
      // 添加调试日志
      console.log('准备播放音频:', fullUrl);
      addToLog(`添加音频到播放队列: ${url}`);
      
      // 添加到播放队列
      audioQueue.value.push({
          url: fullUrl,
          text: '语音回复'
      });
      
      // 如果没有正在播放，开始播放
      if (!isPlaying.value) {
          playNextAudio();
      }
    };

    // Add log
    const addToLog = (message) => {
      const now = new Date();
      const timeStr = now.toLocaleTimeString();
      logs.value.push(`[${timeStr}] ${message}`);
    };

    // Start heartbeat
    const startHeartbeat = () => {
      stopHeartbeat();

      heartbeatInterval.value = setInterval(() => {
        if (socket.value && socket.value.readyState === WebSocket.OPEN) {
          if (Date.now() - lastServerActivity.value > 30000) {
            sendHeartbeat();
          }
        } else {
          stopHeartbeat();
        }
      }, 15000);

      addToLog('心跳检测已启动');
    };

    // Send heartbeat
    const sendHeartbeat = () => {
      try {
        socket.value.send(JSON.stringify({
          type: 'ping',
          timestamp: Date.now()
        }));
        console.log('发送心跳...');
      } catch (e) {
        addToLog(`发送心跳失败: ${e.message}`);
        console.error('心跳发送失败', e);
      }
    };

    // Stop heartbeat
    const stopHeartbeat = () => {
      if (heartbeatInterval.value) {
        clearInterval(heartbeatInterval.value);
        heartbeatInterval.value = null;
      }
    };

    // Handle audio events - 修改为支持连续音频的口型同步
    const onAudioEnded = () => {
      addToLog('音频播放完成');
      
      // 检查是否有下一段音频
      const hasNextAudio = audioQueue.value.length > 1;
      
      // 移除已播放的音频
      audioQueue.value.shift();
      isPlaying.value = false;
      currentAudio.value = null;
      
      // 发送播放完成消息
      if (socket.value && socket.value.readyState === WebSocket.OPEN) {
        socket.value.send(JSON.stringify({
          command: "playback_complete",
          timestamp: Date.now()
        }));
      }
      
      // 如果没有下一段音频，才停止Live2D说话
      if (!hasNextAudio && live2dControls) {
        console.log('AssistantView: 通知Live2D停止说话 (所有音频播放完毕)');
        live2dControls.stopSpeaking();
        
        // 派发全局音频结束事件
        dispatchAudioEvent('ai-audio-ended', {
          finalSegment: true
        });
      }
      
      // 继续播放队列中的下一个
      if (audioQueue.value.length > 0) {
        setTimeout(() => playNextAudio(), 50); // 短暂延迟确保事件处理完成
      }
    };
    
    // 自定义音频事件派发
    const dispatchAudioEvent = (eventName, detail = {}) => {
      try {
        const event = new CustomEvent(eventName, {
          bubbles: true,
          detail: detail
        });
        window.dispatchEvent(event);
        console.log(`AssistantView: 派发${eventName}事件:`, detail);
      } catch (error) {
        console.error(`AssistantView: 派发${eventName}事件失败:`, error);
      }
    };

    const onAudioError = (error) => {
      console.error('音频播放错误:', error);
      addToLog(`音频播放错误: ${error.message || '未知错误'}`);
      
      if (socket.value && socket.value.readyState === WebSocket.OPEN) {
        socket.value.send(JSON.stringify({
          command: "playback_error",
          error: error.message || '未知错误',
          timestamp: Date.now()
        }));
      }
      
      // 跳过错误的音频
      isPlaying.value = false;
      audioQueue.value.shift();
      currentAudio.value = null;
      
      // 尝试播放下一个
      playNextAudio();
    };

    const onAudioLoadStart = () => {
      if (socket.value && socket.value.readyState === WebSocket.OPEN) {
        socket.value.send(JSON.stringify({
          keep_alive: true,
          playback_starting: true,
          timestamp: Date.now()
        }));
      }
    };

    const onAudioCanPlay = () => {
      if (socket.value && socket.value.readyState === WebSocket.OPEN) {
        socket.value.send(JSON.stringify({
          keep_alive: true,
          playback_loaded: true,
          timestamp: Date.now()
        }));
      }
    };

    // Play next audio in queue - 修改为支持连续口型同步
    const playNextAudio = () => {
      if (audioQueue.value.length === 0 || isPlaying.value) return;
      
      // 移除重复音频
      if (audioQueue.value.length > 1) {
        const uniqueUrls = new Set();
        audioQueue.value = audioQueue.value.filter(audio => {
          if (!uniqueUrls.has(audio.url)) {
            uniqueUrls.add(audio.url);
            return true;
          }
          return false;
        });
      }
      
      currentAudio.value = audioQueue.value[0];
      isPlaying.value = true;
      
      // 生成唯一ID
      const audioId = 'assistant-audio-' + Date.now();
      currentAudio.value.id = audioId;
      
      // 通知Live2D和发送全局事件
      if (live2dControls && live2dControls.isVisible.value) {
        live2dControls.speak(currentAudio.value.url);
        dispatchAudioEvent('ai-audio-play', {
          audioUrl: currentAudio.value.url,
          sourceElementId: audioId,
          isNewSegment: true,
          text: currentAudio.value.text
        });
      }
    };

    // Handle history id change
    const onHistoryIdChange = () => {
      if (historyId.value.trim()) {
        historyId.value = historyId.value.trim();
        addToLog(`手动设置对话历史ID: ${historyId.value}`);
        setSessionParams();
      }
    };

    // Lifecycle hook
    onMounted(() => {
      if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
        updateStatus('您的浏览器不支持录音功能');
        addToLog('错误: 浏览器不支持录音功能');
        return;
      }

      addToLog('页面已加载，请点击"连接"按钮连接到WebSocket服务器');
    });
    
    // 在组件卸载时清理资源
    onUnmounted(() => {
      if (socket.value) {
        socket.value.close();
        socket.value = null;
      }
      stopHeartbeat();
    });

    return {
      socket,
      clientId,
      isRecording,
      isConnected,
      isProcessingLlm,
      historyId,
      availableModels,
      apiUrl,
      selectedModel,
      provider,
      apiBase,
      character,
      emotion,
      statusText,
      connectionStatusText,
      conversationHistory,
      logs,
      audioQueue,
      isPlaying,
      connectionStatusClass,
      progressWidth,
      progressText,
      connectBtnDisabled,
      disconnectBtnDisabled,
      createHistoryBtnDisabled,
      startRecordBtnDisabled,
      stopRecordBtnDisabled,
      connectWebSocket,
      disconnectWebSocket,
      setSessionParams,
      fetchAvailableModels,
      createNewHistory,
      startVoiceChat,
      stopVoiceChat,
      handleServerMessage,
      updateStatus,
      addMessageToConversation,
      appendToLastMessage,
      playAudioResponse,
      addToLog,
      startHeartbeat,
      sendHeartbeat,
      stopHeartbeat,
      onAudioEnded,
      onAudioError,
      onAudioLoadStart,
      onAudioCanPlay,
      playNextAudio,
      onHistoryIdChange,
      currentAudio,
      dispatchAudioEvent,
      shouldMuteLocalAudio,
      isSpeaking
    };
  }
};
</script>

<style>
/* 全局共享样式可以放在这里 */
.card {
  @apply bg-white rounded-lg shadow-md p-6 mb-6 border border-gray-200;
}

.btn-group {
  @apply flex flex-wrap justify-center gap-3 my-4;
}

.config-row {
  @apply flex flex-wrap items-center mb-4;
}

.config-panel {
  @apply mt-4;
}

.status {
  @apply text-center font-medium my-3;
}

.connected {
  @apply text-green-600;
}

.disconnected {
  @apply text-red-600;
}

.processing {
  @apply text-yellow-600;
}

label {
  @apply mr-2 w-32 text-gray-700;
}

input, select {
  @apply px-3 py-2 border border-gray-300 rounded-md flex-1 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent;
}

button {
  @apply px-4 py-2 rounded-md font-medium transition-colors duration-200 focus:outline-none focus:ring-2 focus:ring-opacity-50;
}

button.primary {
  @apply bg-green-600 text-white hover:bg-green-700 focus:ring-green-500;
}

button.secondary {
  @apply bg-red-600 text-white hover:bg-red-700 focus:ring-red-500;
}

button.neutral {
  @apply bg-blue-600 text-white hover:bg-blue-700 focus:ring-blue-500;
}

button:disabled {
  @apply bg-gray-400 cursor-not-allowed hover:bg-gray-400;
}

.voice-indicator {
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background-color: #ccc;
}

.voice-indicator.active {
  background-color: #4CAF50;
  animation: pulse 1s infinite;
}

@keyframes pulse {
  0% { transform: scale(1); }
  50% { transform: scale(1.2); }
  100% { transform: scale(1); }
}
</style>
