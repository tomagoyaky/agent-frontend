<template>
  <el-container style="height: 100vh;">
    <el-header style="background-color: #409EFF; color: white; display: flex; align-items: center; justify-content: space-between;">
      <h2>AI Agent 聊天机器人</h2>
      <div style="display: flex; align-items: center; gap: 10px;">
        <el-badge :value="pendingApprovals.length" :hidden="pendingApprovals.length === 0" class="approval-badge">
          <el-button @click="showApprovals = true" type="warning" :icon="Bell" circle></el-button>
        </el-badge>
        <el-switch 
          v-model="useStream" 
          active-text="流式" 
          inactive-text="普通"
        />
        <el-button @click="showSettings = true" type="primary" :icon="Setting" circle></el-button>
      </div>
    </el-header>
    
    <el-container>
      <el-aside width="300px" style="background-color: #f5f5f5; padding: 20px;">
        <h3>对话历史</h3>
        <el-button @click="newChat" type="primary" style="width: 100%; margin-bottom: 10px;">
          新对话
        </el-button>
        <div v-for="thread in threads" :key="thread.id" 
             @click="switchThread(thread.id)"
             :class="['thread-item', { active: currentThreadId === thread.id }]">
          <div class="thread-title">{{ thread.title || '新对话' }}</div>
          <div class="thread-time">{{ formatTime(thread.lastMessage) }}</div>
        </div>
      </el-aside>
      
      <el-main>
        <el-card style="height: calc(100vh - 200px); overflow-y: auto;" ref="messageContainer">
          <div v-for="(msg, idx) in currentMessages" :key="idx" class="message-item">
            <div :class="['message-bubble', msg.role === 'user' ? 'user' : 'assistant', { 'error': msg.status === 'error', 'processing': msg.status === 'processing' }]">
              <div class="message-header">
                <strong>{{ msg.role === 'user' ? '你' : 'AI助手' }}</strong>
                <div class="message-status">
                  <el-icon v-if="msg.status === 'processing'" class="loading-icon"><Loading /></el-icon>
                  <el-icon v-if="msg.status === 'error'" class="error-icon"><WarningFilled /></el-icon>
                  <span class="message-time">{{ formatTime(msg.timestamp) }}</span>
                </div>
              </div>
              <div class="message-content">
                <div v-if="msg.role === 'assistant'" v-html="formatMessageContent(msg.content)" class="assistant-content"></div>
                <div v-else class="user-content">{{ msg.content }}</div>
                
                <!-- 错误消息显示 -->
                <div v-if="msg.status === 'error'" class="error-info">
                  <el-alert
                    title="消息发送失败"
                    :description="msg.errorMessage"
                    type="error"
                    show-icon
                    :closable="false"
                  />
                  <el-button @click="retryMessage(msg)" type="primary" size="small" class="retry-btn">
                    <el-icon><RefreshRight /></el-icon>
                    重试
                  </el-button>
                </div>
                
                <!-- 处理中状态 -->
                <div v-if="msg.status === 'processing'" class="processing-info">
                  <el-tag type="info" size="small">处理中...</el-tag>
                </div>
                <div v-if="msg.searchResults && msg.searchResults.results && msg.searchResults.results.length > 0" class="search-results">
                  <el-divider content-position="left">
                    <el-icon><Search /></el-icon>
                    搜索引用
                  </el-divider>
                  
                  <!-- 搜索答案摘要 -->
                  <div v-if="msg.searchResults.answer" class="search-answer">
                    <el-icon><InfoFilled /></el-icon>
                    <div class="answer-content">
                      <strong>搜索摘要：</strong>
                      <div v-html="formatSearchContent(msg.searchResults.answer)" class="answer-text"></div>
                    </div>
                  </div>
                  
                  <!-- 引用来源 -->
                  <div class="search-sources">
                    <h4 class="sources-title">
                      <el-icon><Link /></el-icon>
                      参考来源：
                    </h4>
                    <div v-for="(result, index) in msg.searchResults.results.slice(0, 5)" :key="result.url" class="search-result-item">
                      <div class="source-header">
                        <span class="source-number">[{{ index + 1 }}]</span>
                        <a :href="result.url" target="_blank" class="search-title">
                          {{ result.title }}
                        </a>
                      </div>
                      <p class="search-snippet" v-html="formatSearchContent(result.content.substring(0, 300))"></p>
                      <div class="source-footer">
                        <el-icon><Document /></el-icon>
                        <span class="source-url">{{ new URL(result.url).hostname }}</span>
                        <el-button size="small" type="text" @click="copyUrl(result.url)">
                          <el-icon><CopyDocument /></el-icon>
                          复制链接
                        </el-button>
                      </div>
                    </div>
                  </div>
                </div>
                <div v-if="msg.approved" class="approval-badge">
                  <el-tag type="success" size="small">已批准</el-tag>
                </div>
              </div>
            </div>
          </div>
          
          <!-- 加载状态 -->
          <div v-if="loading" class="message-item">
            <div class="message-bubble assistant">
              <el-loading-text>{{ loadingText }}</el-loading-text>
            </div>
          </div>
          
          <!-- 流式输出 -->
          <div v-if="streamingResponse" class="message-item">
            <div class="message-bubble assistant">
              <div class="message-header">
                <strong>AI助手</strong>
                <span class="message-time">{{ formatTime(new Date()) }}</span>
              </div>
              <div class="message-content">
                {{ streamingResponse }}
                <span class="streaming-cursor">|</span>
              </div>
            </div>
          </div>
        </el-card>
        
        <div style="margin-top: 20px;">
          <el-input
            v-model="input"
            placeholder="请输入内容..."
            @keyup.enter="sendMessage"
            :disabled="loading || !!streamingResponse"
            class="message-input"
          >
            <template #append>
              <el-button 
                @click="sendMessage" 
                type="primary" 
                :disabled="loading || !!streamingResponse || !input.trim()"
                :loading="loading"
              >
                发送
              </el-button>
            </template>
          </el-input>
          
          <div style="margin-top: 10px; display: flex; gap: 10px; align-items: center;">
            <el-checkbox v-model="enableSearch" size="small">
              <el-icon><Search /></el-icon>
              启用搜索
            </el-checkbox>
            <el-checkbox v-model="needsApproval" size="small">需要人工批准</el-checkbox>
          </div>
        </div>
      </el-main>
    </el-container>
    
    <!-- 设置对话框 -->
    <el-dialog v-model="showSettings" title="系统设置" width="600px">
      <el-form :model="settings" label-width="120px">
        <el-form-item label="模型选择">
          <el-select v-model="settings.model" placeholder="请选择模型" style="width: 100%;">
            <el-option label="GLM-4-Flash" value="glm-4-flash"></el-option>
            <el-option label="GLM-4" value="glm-4"></el-option>
            <el-option label="GLM-4-Plus" value="glm-4-plus"></el-option>
            <el-option label="GLM-4-Air" value="glm-4-air"></el-option>
          </el-select>
        </el-form-item>
        
        <el-form-item label="API Key">
          <el-input 
            v-model="settings.apiKey" 
            type="password" 
            placeholder="请输入GLM API Key"
            show-password
            style="width: 100%;"
          ></el-input>
        </el-form-item>
        
        <el-form-item label="功能设置">
          <div style="display: flex; flex-direction: column; gap: 10px;">
            <el-checkbox v-model="settings.enableThinking">启用思考模式 (Thinking)</el-checkbox>
            <el-checkbox v-model="settings.enableStream">默认使用流式响应</el-checkbox>
            <el-checkbox v-model="settings.enableSearch">默认启用搜索</el-checkbox>
          </div>
        </el-form-item>
        
        <el-form-item label="危险操作">
          <el-button 
            @click="confirmClearHistory = true" 
            type="danger" 
            :icon="Delete"
            style="width: 100%;"
          >
            清理所有历史对话
          </el-button>
        </el-form-item>
      </el-form>
      
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="showSettings = false">取消</el-button>
          <el-button type="primary" @click="saveSettings">保存设置</el-button>
        </span>
      </template>
    </el-dialog>
    
    <!-- 确认清理历史对话框 -->
    <el-dialog v-model="confirmClearHistory" title="确认清理" width="400px">
      <div style="text-align: center; padding: 20px;">
        <el-icon size="48" color="#E6A23C"><WarningFilled /></el-icon>
        <p style="margin: 20px 0; font-size: 16px;">
          确定要清理所有历史对话吗？<br/>
          <strong>此操作不可恢复！</strong>
        </p>
      </div>
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="confirmClearHistory = false">取消</el-button>
          <el-button type="danger" @click="clearAllHistory">确认清理</el-button>
        </span>
      </template>
    </el-dialog>

    <!-- 批准对话框 -->
    <el-dialog v-model="showApprovals" title="待批准操作" width="600px">
      <div v-if="pendingApprovals.length === 0" style="text-align: center; color: #999;">
        暂无待批准操作
      </div>
      <div v-for="approval in pendingApprovals" :key="approval.id" class="approval-item">
        <el-card style="margin-bottom: 10px;">
          <div><strong>用户:</strong> {{ approval.user }}</div>
          <div><strong>请求:</strong> {{ approval.message }}</div>
          <div><strong>建议回复:</strong> {{ approval.response }}</div>
          <div><strong>时间:</strong> {{ formatTime(approval.timestamp) }}</div>
          <div style="margin-top: 10px;">
            <el-button @click="approveRequest(approval.id, true)" type="success" size="small">
              批准
            </el-button>
            <el-button @click="approveRequest(approval.id, false)" type="danger" size="small">
              拒绝
            </el-button>
          </div>
        </el-card>
      </div>
    </el-dialog>
  </el-container>
</template>

<script setup>
import { ref, reactive, onMounted, nextTick, computed } from 'vue';
import { Bell, Search, InfoFilled, Link, Document, CopyDocument, Setting, Delete, WarningFilled, Loading, RefreshRight } from '@element-plus/icons-vue';
import { ElMessage } from 'element-plus';
import axios from 'axios';
import { v4 as uuidv4 } from 'uuid';

const input = ref('');
const loading = ref(false);
const loadingText = ref('正在思考...');
const useStream = ref(true);
const needsApproval = ref(false);
const enableSearch = ref(true);  // 默认启用搜索
const showApprovals = ref(false);
const showSettings = ref(false);
const confirmClearHistory = ref(false);
const streamingResponse = ref('');

// 设置数据
const settings = reactive({
  model: 'glm-4-flash',
  apiKey: '',
  enableThinking: false,
  enableStream: true,
  enableSearch: true
});

const currentThreadId = ref(null);
const currentMessages = ref([]);
const threadList = ref([]);
const pendingApprovals = ref([]);

const user = 'user1'; // 可根据实际登录用户调整
const messageContainer = ref(null);

// 计算属性
const threads = computed(() => {
  return threadList.value.map(thread => ({
    id: thread.id,
    title: thread.title || thread.last_message?.substring(0, 30) || '新对话',
    lastMessage: thread.last_message_time
  }));
});

// 格式化时间
const formatTime = (timestamp) => {
  if (!timestamp) return '';
  const date = new Date(timestamp);
  return date.toLocaleString('zh-CN');
};

// 滚动到底部
const scrollToBottom = () => {
  nextTick(() => {
    if (messageContainer.value) {
      const container = messageContainer.value.$el.querySelector('.el-card__body');
      container.scrollTop = container.scrollHeight;
    }
  });
};

// 新建对话
const newChat = () => {
  currentThreadId.value = uuidv4(); // 为新会话生成默认的threadId
  currentMessages.value = [];
};

// 切换对话线程
const switchThread = async (threadId) => {
  try {
    const response = await axios.get(`http://localhost:3000/thread/${threadId}`);
    currentThreadId.value = threadId;
    currentMessages.value = response.data.messages || [];
    scrollToBottom();
  } catch (error) {
    console.error('切换对话失败:', error);
  }
};

// 加载待批准列表
const loadPendingApprovals = async () => {
  try {
    const response = await axios.get('http://localhost:3000/approvals');
    pendingApprovals.value = response.data;
  } catch (error) {
    console.error('加载待批准列表失败:', error);
  }
};

// 批准或拒绝请求
const approveRequest = async (approvalId, approved) => {
  try {
    const response = await axios.post('http://localhost:3000/approve', {
      approvalId,
      approved
    });
    
    // 刷新待批准列表
    await loadPendingApprovals();
    
    // 如果是当前对话的批准，刷新消息
    if (currentThreadId.value) {
      await switchThread(currentThreadId.value);
    }
    
    ElMessage.success(approved ? '已批准操作' : '已拒绝操作');
  } catch (error) {
    console.error('批准操作失败:', error);
    ElMessage.error('操作失败');
  }
};

// 发送消息
const sendMessage = async () => {
  if (!input.value.trim() || loading.value || streamingResponse.value) return;
  
  const message = input.value;
  input.value = '';
  
  // 添加用户消息到当前对话
  currentMessages.value.push({
    role: 'user',
    content: message,
    timestamp: new Date().toISOString()
  });
  
  scrollToBottom();
  
  try {
    if (useStream.value) {
      // 流式响应
      await sendStreamMessage(message);
    } else {
      // 普通响应
      await sendNormalMessage(message);
    }
  } catch (error) {
    console.error('发送消息失败:', error);
    ElMessage.error('发送失败: ' + error.message);
  } finally {
    loading.value = false;
    streamingResponse.value = '';
  }
};

// 发送普通消息
const sendNormalMessage = async (message) => {
  loading.value = true;
  loadingText.value = enableSearch.value ? '正在搜索...' : '正在思考...';
  
  const response = await axios.post('http://localhost:3000/chat', {
    user,
    message,
    threadId: currentThreadId.value,
    needsApproval: needsApproval.value,
    enableSearch: enableSearch.value
  });
  
  if (response.data.needsApproval) {
    // 需要批准的情况
    currentMessages.value.push({
      role: 'assistant',
      content: response.data.reply,
      timestamp: new Date().toISOString(),
      needsApproval: true
    });
    
    ElMessage.warning('您的请求需要人工批准');
    await loadPendingApprovals();
  } else {
    // 正常回复
    currentMessages.value.push({
      role: 'assistant',
      content: response.data.reply,
      timestamp: new Date().toISOString(),
      searchResults: response.data.searchResults
    });
    
    // 更新当前线程ID
    if (!currentThreadId.value) {
      currentThreadId.value = response.data.threadId;
    }
    
    // 刷新线程列表
    loadThreads();
  }
  
  scrollToBottom();
};

// 发送流式消息
const sendStreamMessage = async (message) => {
  streamingResponse.value = '';
  loading.value = true;
  loadingText.value = '正在连接...';
  
  const response = await fetch('http://localhost:3000/chat/stream', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      user,
      message,
      threadId: currentThreadId.value,
      enableSearch: enableSearch.value
    })
  });
  
  loading.value = false;
  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  
  try {
    while (true) {
      const { value, done } = await reader.read();
      if (done) break;
      
      const chunk = decoder.decode(value);
      const lines = chunk.split('\n');
      
      for (const line of lines) {
        if (line.startsWith('data: ')) {
          try {
            const data = JSON.parse(line.slice(6));
            
            switch (data.type) {
              case 'status':
                loadingText.value = data.content;
                break;
              case 'search':
                // 搜索结果暂存
                break;
              case 'token':
                streamingResponse.value += data.content;
                scrollToBottom();
                break;
              case 'done':
                // 完成，添加到消息列表
                currentMessages.value.push({
                  id: data.messageId,
                  role: 'assistant',
                  content: streamingResponse.value,
                  timestamp: new Date().toISOString(),
                  searchResults: data.searchResults,
                  status: 'success'
                });
                
                if (!currentThreadId.value) {
                  currentThreadId.value = data.threadId;
                }
                
                // 刷新线程列表
                loadThreads();
                
                streamingResponse.value = '';
                scrollToBottom();
                break;
              case 'error':
                // 处理错误状态
                currentMessages.value.push({
                  id: data.messageId,
                  role: 'assistant',
                  content: `请求失败: ${data.content}`,
                  timestamp: new Date().toISOString(),
                  status: 'error',
                  errorMessage: data.content
                });
                
                ElMessage.error(`请求失败: ${data.content}`);
                streamingResponse.value = '';
                scrollToBottom();
                break;
            }
          } catch (e) {
            // 忽略解析错误
          }
        }
      }
    }
  } catch (error) {
    console.error('流式响应错误:', error);
    streamingResponse.value = '';
  }
};

// 复制链接功能
const copyUrl = async (url) => {
  try {
    await navigator.clipboard.writeText(url);
    ElMessage.success('链接已复制到剪贴板');
  } catch (error) {
    console.error('复制失败:', error);
    // 降级方案
    const textArea = document.createElement('textarea');
    textArea.value = url;
    document.body.appendChild(textArea);
    textArea.select();
    try {
      document.execCommand('copy');
      ElMessage.success('链接已复制到剪贴板');
    } catch (err) {
      ElMessage.error('复制失败');
    }
    document.body.removeChild(textArea);
  }
};

// 格式化搜索内容，添加换行和段落
const formatSearchContent = (content) => {
  if (!content) return '';
  
  // 处理内容格式化
  return content
    // 将句号后的空格替换为换行
    .replace(/\. /g, '.<br/>')
    // 将问号后的空格替换为换行
    .replace(/\? /g, '?<br/>')
    // 将感叹号后的空格替换为换行
    .replace(/! /g, '!<br/>')
    // 将中文句号后替换为换行
    .replace(/。 /g, '。<br/>')
    .replace(/？ /g, '？<br/>')
    .replace(/！ /g, '！<br/>')
    // 移除多余的换行
    .replace(/(<br\/>){3,}/g, '<br/><br/>')
    // 确保以省略号结尾
    + '...';
};

// 格式化消息内容，为AI回复添加段落格式
const formatMessageContent = (content) => {
  if (!content) return '';
  
  return content
    // 将双换行替换为段落分隔
    .replace(/\n\n/g, '<br/><br/>')
    // 将单换行替换为换行
    .replace(/\n/g, '<br/>')
    // 处理引用标记，将 [1] [2] 等转换为带样式的引用
    .replace(/\[(\d+)\]/g, '<sup class="citation-mark">[$1]</sup>')
    // 处理中英文标点后的换行
    .replace(/([。！？])\s*/g, '$1<br/>')
    // 处理数字列表
    .replace(/^(\d+\.\s)/gm, '<br/>$1')
    // 处理项目符号
    .replace(/^[-•]\s/gm, '<br/>• ')
    // 移除开头的换行
    .replace(/^<br\/>/g, '')
    // 移除多余的连续换行
    .replace(/(<br\/>){3,}/g, '<br/><br/>');
};

// 加载线程列表
const loadThreads = async () => {
  try {
    const response = await axios.get('http://localhost:3000/threads');
    threadList.value = response.data;
    
    // 如果没有当前选中的线程，自动选中最新的线程
    if (!currentThreadId.value && threadList.value.length > 0) {
      const latestThread = threadList.value[0]; // 线程列表已按时间倒序排列
      if (latestThread && latestThread.last_message) {
        // 只有当线程有消息时才自动选中
        await switchThread(latestThread.id);
      }
    }
  } catch (error) {
    console.error('加载线程列表失败:', error);
  }
};

// 页面加载时初始化
onMounted(() => {
  loadSettings();
  loadThreads();
  loadPendingApprovals();
  // 定期刷新待批准列表
  setInterval(loadPendingApprovals, 30000); // 每30秒刷新一次
});

// 加载设置
const loadSettings = () => {
  try {
    const savedSettings = localStorage.getItem('aiAgentSettings');
    if (savedSettings) {
      const parsed = JSON.parse(savedSettings);
      Object.assign(settings, parsed);
      
      // 同步到界面状态
      useStream.value = settings.enableStream;
      enableSearch.value = settings.enableSearch;
    }
  } catch (error) {
    console.error('加载设置失败:', error);
  }
};

// 保存设置
const saveSettings = async () => {
  try {
    // 验证API Key格式
    if (settings.apiKey && !settings.apiKey.trim()) {
      ElMessage.warning('请输入有效的API Key');
      return;
    }

    // 保存到本地存储
    localStorage.setItem('aiAgentSettings', JSON.stringify(settings));
    
    // 同步到界面状态
    useStream.value = settings.enableStream;
    enableSearch.value = settings.enableSearch;
    
    // 发送设置到后端
    await axios.post('http://localhost:3000/settings', {
      model: settings.model,
      apiKey: settings.apiKey,
      enableThinking: settings.enableThinking
    });
    
    showSettings.value = false;
    ElMessage.success('设置保存成功');
  } catch (error) {
    console.error('保存设置失败:', error);
    ElMessage.error('保存设置失败: ' + error.message);
  }
};

// 重试失败的消息
const retryMessage = async (failedMessage) => {
  if (!currentThreadId.value) return;
  
  // 找到失败消息前面的用户消息
  const messageIndex = currentMessages.value.findIndex(msg => msg.id === failedMessage.id);
  if (messageIndex > 0) {
    const userMessage = currentMessages.value[messageIndex - 1];
    if (userMessage && userMessage.role === 'user') {
      // 移除失败的消息
      currentMessages.value.splice(messageIndex, 1);
      
      // 重新发送
      if (useStream.value) {
        await sendStreamMessage(userMessage.content);
      } else {
        await sendNormalMessage(userMessage.content);
      }
    }
  }
};

// 清理所有历史对话
const clearAllHistory = async () => {
  try {
    await axios.delete('http://localhost:3000/threads/all');
    
    // 清空本地状态
    threadList.value = [];
    currentMessages.value = [];
    currentThreadId.value = null;
    
    confirmClearHistory.value = false;
    ElMessage.success('所有历史对话已清理');
  } catch (error) {
    console.error('清理历史对话失败:', error);
    ElMessage.error('清理失败: ' + error.message);
  }
};
</script>

<style scoped>
.message-item {
  margin-bottom: 15px;
  display: flex;
}

.message-bubble {
  max-width: 70%;
  padding: 12px 16px;
  border-radius: 12px;
  position: relative;
}

.message-bubble.user {
  background-color: #409EFF;
  color: white;
  margin-left: auto;
  border-bottom-right-radius: 4px;
}

.message-bubble.assistant {
  background-color: #f5f5f5;
  color: #333;
  margin-right: auto;
  border-bottom-left-radius: 4px;
}

.message-bubble.error {
  background-color: #fef0f0;
  border: 1px solid #fbc4c4;
}

.message-bubble.processing {
  background-color: #f0f9ff;
  border: 1px solid #bfdbfe;
}

.message-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
  font-size: 12px;
}

.message-status {
  display: flex;
  align-items: center;
  gap: 5px;
}

.loading-icon {
  animation: spin 1s linear infinite;
  color: #409EFF;
}

.error-icon {
  color: #F56C6C;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

.error-info {
  margin-top: 10px;
  padding: 10px;
  border-radius: 6px;
  background-color: #fef2f2;
}

.retry-btn {
  margin-top: 8px;
}

.processing-info {
  margin-top: 8px;
}

.message-time {
  opacity: 0.7;
  font-size: 11px;
}

.message-content {
  line-height: 1.5;
  word-wrap: break-word;
}

.assistant-content {
  line-height: 1.6;
  white-space: normal;
}

.user-content {
  line-height: 1.5;
}

.search-results {
  margin-top: 12px;
  padding: 12px;
  background-color: rgba(64, 158, 255, 0.05);
  border: 1px solid rgba(64, 158, 255, 0.2);
  border-radius: 8px;
}

.search-answer {
  margin-bottom: 16px;
  padding: 12px;
  background-color: rgba(103, 194, 58, 0.1);
  border-radius: 6px;
  border-left: 4px solid #67c23a;
  display: flex;
  align-items: flex-start;
  gap: 8px;
}

.answer-content {
  flex: 1;
}

.answer-text {
  margin-top: 8px;
  line-height: 1.6;
}

.search-sources {
  margin-top: 12px;
}

.sources-title {
  margin: 0 0 12px 0;
  font-size: 14px;
  color: #409EFF;
  display: flex;
  align-items: center;
  gap: 6px;
}

.search-result-item {
  margin-bottom: 12px;
  padding: 12px;
  background-color: white;
  border-radius: 6px;
  border: 1px solid #e4e7ed;
  transition: all 0.2s;
}

.search-result-item:hover {
  border-color: #409EFF;
  box-shadow: 0 2px 4px rgba(64, 158, 255, 0.1);
}

.source-header {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  margin-bottom: 8px;
}

.source-number {
  background-color: #409EFF;
  color: white;
  font-size: 12px;
  font-weight: bold;
  padding: 2px 6px;
  border-radius: 4px;
  min-width: 20px;
  text-align: center;
  flex-shrink: 0;
}

.search-title {
  color: #409EFF;
  text-decoration: none;
  font-weight: 600;
  font-size: 14px;
  line-height: 1.4;
  flex: 1;
}

.search-title:hover {
  text-decoration: underline;
  color: #337ecc;
}

.search-snippet {
  margin: 8px 0;
  font-size: 13px;
  color: #666;
  line-height: 1.6;
  text-align: justify;
  white-space: normal;
  word-break: break-word;
}

.source-footer {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-top: 8px;
  padding-top: 8px;
  border-top: 1px solid #f0f0f0;
}

.source-url {
  font-size: 12px;
  color: #909399;
  flex: 1;
}

.thread-item {
  padding: 12px;
  margin-bottom: 8px;
  background-color: white;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s;
  border: 1px solid #e4e7ed;
}

.thread-item:hover {
  background-color: #f0f9ff;
  border-color: #409EFF;
}

.thread-item.active {
  background-color: #409EFF;
  color: white;
  border-color: #409EFF;
}

.thread-title {
  font-weight: bold;
  margin-bottom: 4px;
  font-size: 14px;
}

.thread-time {
  font-size: 12px;
  opacity: 0.7;
}

.approval-item {
  margin-bottom: 12px;
}

.approval-badge {
  margin-left: 8px;
}

.streaming-cursor {
  animation: blink 1s infinite;
  font-weight: bold;
}

@keyframes blink {
  0%, 50% { opacity: 1; }
  51%, 100% { opacity: 0; }
}

.message-input {
  width: 100%;
}

/* 加载状态样式 */
.el-loading-text {
  color: #666;
  font-style: italic;
}

/* 引用标记样式 */
.citation-mark {
  background-color: #409EFF;
  color: white;
  font-size: 10px;
  font-weight: bold;
  padding: 1px 4px;
  border-radius: 3px;
  margin-left: 2px;
  cursor: pointer;
  transition: all 0.2s;
}

.citation-mark:hover {
  background-color: #337ecc;
  transform: scale(1.1);
}

/* 响应式设计 */
@media (max-width: 768px) {
  .el-aside {
    width: 250px !important;
  }
  
  .message-bubble {
    max-width: 85%;
  }
}
</style>
