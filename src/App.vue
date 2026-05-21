<template>

  <div class="flex h-screen">
    <!-- 左侧区块 -->
    <div :class="[
      'bg-white overflow-auto h-screen transition-all duration-300 ease-in-out',
      isSidebarOpen ? 'w-[200px]' : 'w-0',
      'absolute md:relative md:w-[300px]'
    ]" id="leftSidebar">
      <!-- 头部 -->
      <div class="flex items-center justify-between px-4 py-2">
        <h1 class="text-3xl font-bold text-pink-500">ChatFUN</h1>
        <!-- 开始新对话 -->
        <div class="hidden md:block">
          <el-button circle @click="startNewChat">
            <img src="/new.svg" width="20" height="20" alt="Start new chat" />
          </el-button>
        </div>
        <div class="md:hidden">
          <el-button circle @click="toggleSidebar">
            <el-icon>
              <Close />
            </el-icon>
          </el-button>
        </div>
      </div>

      <!-- 搜索框 -->
      <div class="mt-4 px-4 mb-4">
        <!-- <SearchInput @search="handleSearch" /> -->
        <el-input placeholder="搜素..." v-model="searchValue">
          <template #append>
            <el-button :icon="Search" />
          </template>
        </el-input>
      </div>

      <!-- 聊天列表 -->
      <div class="overflow-y-auto">
        <div v-for="chat in chatList" :key="chat.id" class="px-4 py-2 cursor-pointer hover:bg-gray-100">
          <div class="font-bold text-lg">{{ chat.title }}</div>
          <div class="text-sm text-gray-500">{{ chat.lastMessage }}</div>
        </div>
      </div>
    </div>

    <!-- 右侧区块 -->
    <div class="flex flex-1 flex-col bg-gray-100 h-screen">
      <!-- 聊天标题 -->
      <div class="p-4 border-b flex items-center justify-between">
        <div :class="[isSidebarOpen ? 'hidden' : 'block', 'md:hidden']">
          <el-button circle @click="toggleSidebar">
            <el-icon>
              <More />
            </el-icon>
          </el-button>
        </div>
        <h1 class="text-xl font-bold text-gray-500 flex-grow text-center">{{ currentChatTitle }}</h1>
        <div class="md:hidden">
          <el-button circle @click="startNewChat">
            <el-icon>
              <Plus />
            </el-icon>
          </el-button>
        </div>
      </div>

      <!-- 消息列表 -->
      <div class="flex-1 overflow-auto p-4">
        <div v-for="(message, index) in messages" :key="index"
          :class="['flex', { 'flex-row-reverse': message.user === 'user' }]">
          <div class='min-w-[40px] min-h-[40px]'>
            <img :src="message.user === 'ai' ? '/robot_ai.png' : '/me.png'" class="rounded-full" width=40 height=40
              alt="avatar" />
          </div>
          <div
            :class="{ 'flex flex-col mb-5': true, 'mr-14 ml-3': message.user === 'ai', 'mr-3 ml-14': message.user === 'user' }">
            <div
              :class="{ 'px-4 py-2 rounded-lg shadow-lg md:max-w-fit': true, 'bg-white text-black': message.user === 'ai', 'bg-green-500 text-white': message.user === 'user' }">
              <div v-if="message.user === 'ai'">
                <div v-for="(part, partIndex) in parseMessage(message.content)" :key="partIndex">
                  <highlightjs v-if="part.isCode" :code="part.content" :language="part.language" />
                  <span v-else v-html="part.content"></span>
                </div>
              </div>
              <span v-else>{{ message.content }}</span>
            </div>
          </div>
        </div>
      </div>

      <!-- 底部输入框 -->
      <div class="border-t p-4">
        <el-input size=large placeholder="请输入..." v-model="userInputValue" @keyup.enter="sendMessage"
          :disabled="isProcessing">
          <template #append>
            <el-button :icon="Promotion" @click="sendMessage" :disabled="isProcessing" />
          </template>
        </el-input>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { Search, Promotion } from '@element-plus/icons-vue'
import axios from 'axios'
import { nanoid } from 'nanoid';

// 数据
const userInputValue = ref('')
const searchValue = ref('')
const messages = ref([])
const isProcessing = ref(false)
// const isInitialized = ref(false)
const seq = ref(0)
const currentChatId = ref(nanoid())
const chatList = ref([])
const currentChatTitle = ref('对话标题')
const isSidebarOpen = ref(false)
const chatContainer = ref(null);
const chatApiUrl = 'https://www.chatfun.site/api/python'


// 计算属性
const bjtime = computed(() => new Date().toLocaleString('zh-CN', { timeZone: 'Asia/Shanghai' }))

// 方法
/* 切换侧边栏 */
function toggleSidebar() {
  isSidebarOpen.value = !isSidebarOpen.value
}

/* 开始新对话 */
function startNewChat() {
  currentChatId.value = nanoid()
  seq.value = 0
  messages.value = []
  currentChatTitle.value = '新对话'
  initializeChat()
}

/* 用户提交聊天并返回结果 */
async function sendMessage() {
  if (userInputValue.value.trim() === '') return

  // 添加用户消息
  const userMessage = {
    ssid: currentChatId.value,
    seq: seq.value,
    user: 'user',
    content: userInputValue.value,
    timestamp: bjtime.value
  }
  messages.value.push(userMessage)
  const userMessageContent = userInputValue.value
  userInputValue.value = ''
  seq.value++

  // 准备 AI 回复
  const aiMessage = {
    ssid: currentChatId.value,
    seq: seq.value,
    user: 'ai',
    content: '',
    timestamp: bjtime.value
  }
  messages.value.push(aiMessage)
  seq.value++

  // AI 回复
  try {
    const response = await axios.post(chatApiUrl, {
      content: userMessageContent,
      chatHistory: JSON.stringify(messages.value)
    }, {
      responseType: 'text',
      onDownloadProgress: progressEvent => {
        const responseText = progressEvent.event.target.responseText;
        if (responseText) {
          // 只获取新的内容
          const newContent = responseText.slice(aiMessage.content.length);
          aiMessage.content += newContent;

          // 触发视图更新
          messages.value = [...messages.value];
        }
      }
    });

    // 更新聊天列表
    updateChatList()
  } catch (error) {
    console.error('Error:', error)
    messages.value[messages.value.length - 1].content = '抱歉，发生了错误。请稍后再试。'
  } finally {
    isProcessing.value = false
  }
}

// 初始化聊天
function initializeChat() {
  const initialMessage = {
    ssid: nanoid(),
    seq: seq.value,
    user: 'ai',
    content: '你好，有什么可以帮助你的？',
    timestamp: bjtime.value
  }
  messages.value.push(initialMessage)
  console.log('初始化聊天', messages.value)
  currentChatId.value = initialMessage.ssid
  seq.value++ // 增加seq
  updateChatList()
}

// 代码显示
function parseMessage(message) {
  const parts = []
  const codeBlockRegex = /```(\w+)?\n([\s\S]*?)```/g
  let lastIndex = 0
  let match

  while ((match = codeBlockRegex.exec(message)) !== null) {
    // Add text before code block
    if (match.index > lastIndex) {
      parts.push({
        isCode: false,
        content: message.slice(lastIndex, match.index).replace(/\n/g, '<br>')
      })
    }

    // Add code block
    parts.push({
      isCode: true,
      language: match[1] || 'plaintext',
      content: match[2].trim()
    })

    lastIndex = match.index + match[0].length
  }

  // Add remaining text after last code block
  if (lastIndex < message.length) {
    parts.push({
      isCode: false,
      content: message.slice(lastIndex).replace(/\n/g, '<br>')
    })
  }

  return parts
}

// 更新聊天列表
function updateChatList() {
  const existingChatIndex = chatList.value.findIndex(chat => chat.id === currentChatId.value)
  const lastMessage = messages.value[messages.value.length - 1].content
  // 显示20个字符
  const displayedMessage = lastMessage.length > 20 ? lastMessage.substring(0, 20) + "..." : lastMessage;

  if (existingChatIndex !== -1) {
    chatList.value[existingChatIndex].lastMessage = displayedMessage
  } else {
    chatList.value.unshift({
      id: currentChatId.value,
      title: currentChatTitle.value,
      lastMessage: displayedMessage
    })
  }
}

// 处理移动端窗口大小变化
const handleResize = () => {
  if (chatContainer.value) {
    chatContainer.value.style.height = `${window.innerHeight}px`;
  }
};

onMounted(() => {
  initializeChat();
  handleResize();
})



</script>


<style scoped>
/* 你可能需要添加一些样式来优化代码块的显示 */
:deep(.hljs) {
  padding: 1em;
  border-radius: 5px;
  font-size: 0.9em;
  overflow-x: auto;
}
</style>
