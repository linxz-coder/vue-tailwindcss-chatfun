<template>
  <div class="relative flex h-[100dvh] overflow-hidden bg-[#f6f8fc] text-slate-900">
    <button
      v-if="isSidebarOpen"
      type="button"
      class="fixed inset-0 z-20 bg-slate-950/35 backdrop-blur-[2px] md:hidden"
      aria-label="关闭会话列表"
      @click="toggleSidebar"
    />

    <!-- 左侧区块 -->
    <div
      :class="[
        'fixed inset-y-0 left-0 z-30 flex w-[86vw] max-w-[340px] flex-col overflow-hidden border-r border-slate-200 bg-white shadow-2xl transition-transform duration-300 ease-out md:relative md:z-auto md:w-[300px] md:translate-x-0 md:shadow-none',
        isSidebarOpen ? 'translate-x-0' : '-translate-x-full'
      ]"
      id="leftSidebar"
    >
      <!-- 头部 -->
      <div class="flex items-center justify-between border-b border-slate-100 px-4 py-4">
        <div>
          <h1 class="text-2xl font-bold tracking-tight text-slate-950">ChatFUN</h1>
          <p class="text-xs text-slate-500">本地会话</p>
        </div>
        <!-- 开始新对话 -->
        <div class="hidden md:block">
          <button type="button" class="icon-button" aria-label="新增会话" @click="startNewChat">
            <img src="/new.svg" class="h-5 w-5" width="20" height="20" alt="Start new chat" />
          </button>
        </div>
        <div class="md:hidden">
          <button type="button" class="icon-button" aria-label="关闭会话管理" @click="toggleSidebar">
            <Close class="h-5 w-5" />
          </button>
        </div>
      </div>

      <!-- 搜索框 -->
      <div class="px-4 py-3">
        <el-input class="search-input" placeholder="搜索..." v-model="searchValue">
          <template #append>
            <el-button :icon="Search" />
          </template>
        </el-input>
      </div>

      <!-- 聊天列表 -->
      <div class="flex-1 space-y-1 overflow-y-auto px-3 pb-6">
        <div
          v-for="chat in filteredChatList"
          :key="chat.id"
          class="relative overflow-hidden rounded-2xl"
        >
          <button
            type="button"
            class="absolute inset-y-0 left-0 w-20 rounded-2xl bg-rose-500 text-sm font-semibold text-white"
            @click.stop="deleteChat(chat.id)"
          >
            删除
          </button>
          <button
            type="button"
            :class="[
              'relative block w-full rounded-2xl px-4 py-3 text-left transition active:scale-[0.99]',
              chat.id === currentChatId
                ? 'bg-teal-50 text-teal-950 ring-1 ring-teal-100'
                : 'bg-white text-slate-700 hover:bg-slate-50'
            ]"
            :style="{ transform: swipedChatId === chat.id ? 'translateX(80px)' : 'translateX(0)' }"
            @touchstart="handleChatTouchStart($event, chat.id)"
            @touchend="handleChatTouchEnd($event, chat.id)"
            @click="handleChatClick(chat.id)"
          >
            <div class="truncate text-[15px] font-semibold">{{ chat.title }}</div>
            <div class="mt-1 truncate text-sm text-slate-500">{{ chat.lastMessage }}</div>
          </button>
        </div>
        <div v-if="filteredChatList.length === 0" class="rounded-2xl px-4 py-5 text-sm text-slate-500">
          无相关搜索结果
        </div>
      </div>
    </div>

    <!-- 右侧区块 -->
    <div class="flex h-[100dvh] min-w-0 flex-1 flex-col overflow-hidden bg-[radial-gradient(circle_at_top_left,#ecfeff_0,#f6f8fc_36%,#f8fafc_100%)]">
      <!-- 聊天标题 -->
      <div class="flex min-h-[64px] items-center justify-between gap-2 border-b border-white/80 bg-white/85 px-4 shadow-sm backdrop-blur">
        <div :class="[isSidebarOpen ? 'hidden' : 'block', 'md:hidden']">
          <button type="button" class="icon-button" aria-label="会话管理" @click="toggleSidebar">
            <More class="h-[22px] w-[22px]" />
          </button>
        </div>
        <h1 class="min-w-0 flex-grow truncate px-2 text-center text-lg font-bold text-slate-700 md:text-left md:text-xl">
          {{ currentChatTitle }}
        </h1>
        <div class="md:hidden">
          <button type="button" class="icon-button icon-button-primary" aria-label="新增会话" @click="startNewChat">
            <Plus class="h-[22px] w-[22px]" />
          </button>
        </div>
      </div>

      <!-- 消息列表 -->
      <div ref="messageListRef" class="flex-1 overflow-auto px-3 py-4 md:px-6">
        <div
          v-for="(message, index) in messages"
          :key="message.id || index"
          :class="['flex w-full', { 'flex-row-reverse': message.user === 'user' }]"
        >
          <div class="min-h-[34px] min-w-[34px] md:min-h-[40px] md:min-w-[40px]">
            <img
              :src="message.user === 'ai' ? '/robot_ai.png' : '/me.png'"
              class="h-[34px] w-[34px] rounded-full shadow-sm ring-2 ring-white md:h-10 md:w-10"
              width="40"
              height="40"
              alt="avatar"
            />
          </div>
          <div
            :class="{
              'mb-5 flex min-w-0 max-w-[82%] flex-col md:max-w-[72%]': true,
              'mr-7 ml-3 items-start md:mr-14': message.user === 'ai',
              'mr-3 ml-7 items-end md:ml-14': message.user === 'user'
            }"
          >
            <div
              :class="{
                'max-w-full overflow-hidden rounded-2xl px-4 py-2 text-[15px] leading-7 shadow-sm ring-1': true,
                'rounded-tl-md bg-white text-slate-900 ring-slate-200': message.user === 'ai',
                'rounded-tr-md bg-teal-600 text-white ring-teal-500': message.user === 'user'
              }"
            >
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
      <div class="border-t border-white/80 bg-white/85 p-3 shadow-[0_-10px_30px_rgba(15,23,42,0.08)] backdrop-blur md:p-4">
        <el-input
          class="composer-input mx-auto block max-w-3xl"
          size="large"
          placeholder="请输入..."
          v-model="userInputValue"
          @keyup.enter.exact="sendMessage"
          :disabled="isProcessing"
        >
          <template #append>
            <button type="button" class="send-button" @click="sendMessage" :disabled="isProcessing" aria-label="发送">
              <Promotion class="h-6 w-6" />
            </button>
          </template>
        </el-input>
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed, nextTick, onMounted, ref } from 'vue'
import { Close, More, Plus, Promotion, Search } from '@element-plus/icons-vue'
import axios from 'axios'
import { nanoid } from 'nanoid'

const STORAGE_KEY = 'chatfun-vue-sessions-v1'
const chatApiUrl = 'https://www.chatfun.site/api/python'
const titleApiUrl = 'https://www.chatfun.site/api/title'

const userInputValue = ref('')
const searchValue = ref('')
const messages = ref([])
const isProcessing = ref(false)
const seq = ref(0)
const currentChatId = ref('')
const sessions = ref({})
const currentChatTitle = ref('对话标题')
const isSidebarOpen = ref(false)
const messageListRef = ref(null)
const swipedChatId = ref('')
const touchStart = ref({ chatId: '', x: 0, y: 0 })

let activeController = null

const chatList = computed(() =>
  Object.values(sessions.value).sort((a, b) => (b.updatedAtMs || 0) - (a.updatedAtMs || 0))
)

const filteredChatList = computed(() => {
  const keyword = searchValue.value.trim().toLowerCase()
  if (!keyword) return chatList.value

  return chatList.value.filter(chat => {
    return (
      chat.title.toLowerCase().includes(keyword) ||
      chat.lastMessage.toLowerCase().includes(keyword)
    )
  })
})

function nowText() {
  return new Date().toLocaleString('zh-CN', {
    timeZone: 'Asia/Shanghai',
    hour12: false
  })
}

function cloneMessages(source) {
  return source.map(message => ({ ...message }))
}

function maxSeq(source) {
  return source.reduce((max, message) => Math.max(max, Number(message.seq) || 0), 0)
}

function createInitialMessage(chatId) {
  return {
    id: nanoid(),
    ssid: chatId,
    seq: 0,
    user: 'ai',
    content: '你好，有什么可以帮助你的？',
    timestamp: nowText()
  }
}

function buildSession(id, title, sourceMessages, existing = {}, options = {}) {
  const savedMessages = cloneMessages(sourceMessages)
  const lastMessage = savedMessages[savedMessages.length - 1]
  const nowMs = Date.now()
  const shouldTouch = options.touchUpdatedAt !== false || !existing.updatedAtMs

  return {
    id,
    title: title || '新对话',
    messages: savedMessages,
    lastMessage: lastMessage?.content || '新对话',
    createdAt: existing.createdAt || nowText(),
    createdAtMs: existing.createdAtMs || nowMs,
    updatedAt: shouldTouch ? nowText() : existing.updatedAt,
    updatedAtMs: shouldTouch ? nowMs : existing.updatedAtMs
  }
}

function persistSessions() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(sessions.value))
}

function saveCurrentSession(options = {}) {
  if (!currentChatId.value) return

  const existing = sessions.value[currentChatId.value] || {}
  sessions.value = {
    ...sessions.value,
    [currentChatId.value]: buildSession(
      currentChatId.value,
      currentChatTitle.value,
      messages.value,
      existing,
      options
    )
  }
  persistSessions()
}

function activateSession(session) {
  currentChatId.value = session.id
  currentChatTitle.value = session.title || '新对话'
  messages.value = cloneMessages(session.messages || [])
  seq.value = maxSeq(messages.value) + 1
  userInputValue.value = ''
  isSidebarOpen.value = false
  scrollToBottom()
}

function loadSessions() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    const parsed = raw ? JSON.parse(raw) : {}
    const validSessions = Object.fromEntries(
      Object.entries(parsed).filter(([, session]) => session && Array.isArray(session.messages))
    )

    if (Object.keys(validSessions).length > 0) {
      sessions.value = validSessions
      activateSession(chatList.value[0])
      return
    }
  } catch (error) {
    console.error('Error loading local sessions:', error)
  }

  startNewChat()
}

function cancelActiveRequest() {
  if (activeController) {
    activeController.abort()
    activeController = null
  }
  isProcessing.value = false
}

function toggleSidebar() {
  isSidebarOpen.value = !isSidebarOpen.value
  swipedChatId.value = ''
}

function startNewChat() {
  cancelActiveRequest()

  const chatId = nanoid()
  const initialMessage = createInitialMessage(chatId)

  currentChatId.value = chatId
  currentChatTitle.value = '新对话'
  messages.value = [initialMessage]
  seq.value = 1
  userInputValue.value = ''
  saveCurrentSession()
  scrollToBottom()
}

function selectChat(chatId) {
  if (chatId === currentChatId.value) {
    isSidebarOpen.value = false
    swipedChatId.value = ''
    return
  }

  cancelActiveRequest()
  saveCurrentSession({ touchUpdatedAt: false })

  const session = sessions.value[chatId]
  if (session) activateSession(session)
}

function handleChatTouchStart(event, chatId) {
  const touch = event.touches[0]
  touchStart.value = {
    chatId,
    x: touch.clientX,
    y: touch.clientY
  }
}

function handleChatTouchEnd(event, chatId) {
  if (touchStart.value.chatId !== chatId) return

  const touch = event.changedTouches[0]
  const deltaX = touch.clientX - touchStart.value.x
  const deltaY = Math.abs(touch.clientY - touchStart.value.y)

  if (deltaX > 56 && deltaY < 42) {
    swipedChatId.value = chatId
  } else if (deltaX < -24 || deltaY >= 42) {
    swipedChatId.value = ''
  }

  touchStart.value = { chatId: '', x: 0, y: 0 }
}

function handleChatClick(chatId) {
  if (swipedChatId.value === chatId) {
    swipedChatId.value = ''
    return
  }

  selectChat(chatId)
}

function deleteChat(chatId) {
  const remainingSessions = { ...sessions.value }
  delete remainingSessions[chatId]
  sessions.value = remainingSessions
  swipedChatId.value = ''

  if (chatId === currentChatId.value) {
    const nextSession = chatList.value[0]
    if (nextSession) {
      activateSession(nextSession)
    } else {
      currentChatId.value = ''
      startNewChat()
    }
  }

  persistSessions()
}

async function sendMessage() {
  const content = userInputValue.value.trim()
  if (!content || isProcessing.value) return

  isProcessing.value = true

  const userMessage = {
    id: nanoid(),
    ssid: currentChatId.value,
    seq: seq.value,
    user: 'user',
    content,
    timestamp: nowText()
  }
  seq.value += 1

  const aiMessage = {
    id: nanoid(),
    ssid: currentChatId.value,
    seq: seq.value,
    user: 'ai',
    content: '',
    timestamp: nowText()
  }
  seq.value += 1

  messages.value = [...messages.value, userMessage, aiMessage]
  userInputValue.value = ''
  saveCurrentSession()
  scrollToBottom()

  activeController = new AbortController()

  try {
    const response = await axios.post(
      chatApiUrl,
      {
        content,
        chatHistory: JSON.stringify(messages.value)
      },
      {
        responseType: 'text',
        signal: activeController.signal,
        onDownloadProgress: progressEvent => {
          const responseText =
            progressEvent.event?.target?.responseText ||
            progressEvent.currentTarget?.responseText ||
            ''

          if (responseText) {
            aiMessage.content = responseText
            replaceMessage(aiMessage)
            saveCurrentSession()
            scrollToBottom()
          }
        }
      }
    )

    if (!aiMessage.content && typeof response.data === 'string') {
      aiMessage.content = response.data
      replaceMessage(aiMessage)
    }

    await updateTitle()
  } catch (error) {
    if (error.code === 'ERR_CANCELED') return

    console.error('Error:', error)
    aiMessage.content = '抱歉，发生了错误。请稍后再试。'
    replaceMessage(aiMessage)
  } finally {
    activeController = null
    isProcessing.value = false
    saveCurrentSession()
    scrollToBottom()
  }
}

function replaceMessage(updatedMessage) {
  messages.value = messages.value.map(message =>
    message.id === updatedMessage.id ? { ...updatedMessage } : message
  )
}

async function updateTitle() {
  const nonEmptyMessages = messages.value.filter(message => message.content.trim())
  const firstUserMessage = messages.value.find(message => message.user === 'user')

  if (!firstUserMessage) {
    currentChatTitle.value = '新对话'
    saveCurrentSession()
    return
  }

  const fallbackTitle = firstUserMessage.content.slice(0, 18)

  try {
    const response = await axios.post(
      titleApiUrl,
      {
        content: nonEmptyMessages.slice(-4).map(message => message.content).join('\n')
      },
      { responseType: 'text' }
    )

    currentChatTitle.value = response.data?.trim() || fallbackTitle
  } catch (error) {
    console.error('Error updating title:', error)
    currentChatTitle.value = fallbackTitle
  }

  saveCurrentSession()
}

function parseMessage(message) {
  const parts = []
  const codeBlockRegex = /```(\w+)?\n([\s\S]*?)```/g
  let lastIndex = 0
  let match

  while ((match = codeBlockRegex.exec(message)) !== null) {
    if (match.index > lastIndex) {
      parts.push({
        isCode: false,
        content: formatText(message.slice(lastIndex, match.index))
      })
    }

    parts.push({
      isCode: true,
      language: match[1] || 'plaintext',
      content: match[2].trim()
    })

    lastIndex = match.index + match[0].length
  }

  if (lastIndex < message.length) {
    parts.push({
      isCode: false,
      content: formatText(message.slice(lastIndex))
    })
  }

  return parts
}

function formatText(text) {
  return escapeHtml(text).replace(/\n/g, '<br>')
}

function escapeHtml(text) {
  return text
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#039;')
}

async function scrollToBottom() {
  await nextTick()
  if (messageListRef.value) {
    messageListRef.value.scrollTop = messageListRef.value.scrollHeight
  }
}

onMounted(() => {
  loadSessions()
})
</script>

<style scoped>
:deep(.hljs) {
  padding: 1em;
  border-radius: 12px;
  font-size: 0.9em;
  overflow-x: auto;
}

.icon-button {
  display: grid;
  width: 44px;
  min-width: 44px;
  height: 44px;
  min-height: 44px;
  place-items: center;
  border: 1px solid rgb(226 232 240);
  border-radius: 999px;
  background: #ffffff;
  color: #334155;
  box-shadow: 0 8px 24px rgba(15, 23, 42, 0.08);
  transition: transform 0.15s ease, background-color 0.15s ease;
}

.icon-button:active {
  transform: scale(0.96);
}

.icon-button-primary {
  border-color: #0f172a;
  background: #0f172a;
  color: #ffffff;
}

.search-input {
  border-radius: 18px;
  border: 1px solid rgb(203 213 225);
  background: #eef5f6;
  padding: 3px;
  box-shadow: 0 10px 24px rgba(15, 23, 42, 0.08);
}

.search-input :deep(.el-input__wrapper),
.composer-input :deep(.el-input__wrapper) {
  border-radius: 16px 0 0 16px;
  box-shadow: none;
}

.search-input :deep(.el-input__wrapper) {
  background: #ffffff;
}

.search-input :deep(.el-input-group__append),
.composer-input :deep(.el-input-group__append) {
  border-radius: 0 16px 16px 0;
  background: #f8fafc;
  box-shadow: none;
}

.search-input :deep(.el-input-group__append) {
  border-left: 1px solid rgb(226 232 240);
  background: #ffffff;
}

.search-input :deep(.el-input__inner),
.composer-input :deep(.el-input__inner) {
  font-size: 16px;
}

.composer-input :deep(.el-input__wrapper) {
  min-height: 54px;
}

.composer-input :deep(.el-input-group__append) {
  min-width: 60px;
  padding: 0 6px;
}

.send-button {
  display: grid;
  width: 48px;
  height: 48px;
  place-items: center;
  border: 0;
  border-radius: 999px;
  background: #0f172a;
  color: #ffffff;
  transition: transform 0.15s ease, background-color 0.15s ease;
}

.send-button:active {
  transform: scale(0.96);
}

.send-button:disabled {
  background: #cbd5e1;
  color: #ffffff;
  cursor: not-allowed;
}

@media (max-width: 767px) {
  .composer-input :deep(.el-input__wrapper) {
    min-height: 52px;
  }

  .composer-input :deep(.el-input-group__append) {
    min-width: 62px;
  }

  .send-button {
    width: 48px;
    height: 48px;
  }
}
</style>
