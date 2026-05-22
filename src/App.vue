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
            class="absolute inset-y-0 right-0 w-20 rounded-2xl bg-rose-500 text-sm font-semibold text-white md:hidden"
            @click.stop="deleteChat(chat.id)"
          >
            删除
          </button>
          <div
            class="relative rounded-2xl bg-white transition"
            :style="{ transform: swipedChatId === chat.id ? 'translateX(-80px)' : 'translateX(0)' }"
          >
            <button
              type="button"
              :class="[
                'block w-full rounded-2xl px-4 py-3 text-left transition active:scale-[0.99] md:pb-8 md:pr-12',
                chat.id === currentChatId
                  ? 'bg-teal-50 text-teal-950 ring-1 ring-teal-100'
                  : 'bg-white text-slate-700 hover:bg-slate-50'
              ]"
              @touchstart="handleChatTouchStart($event, chat.id)"
              @touchend="handleChatTouchEnd($event, chat.id)"
              @click="handleChatClick(chat.id)"
            >
              <div class="flex items-start justify-between gap-3">
                <div class="min-w-0 truncate text-[15px] font-semibold">{{ chat.title }}</div>
                <time class="shrink-0 pt-0.5 text-[11px] font-medium text-slate-400">
                  {{ formatSessionTimestamp(chat) }}
                </time>
              </div>
              <div class="mt-1 truncate text-sm text-slate-500">{{ chat.lastMessage }}</div>
            </button>
            <button
              type="button"
              :aria-label="`删除${chat.title || '会话'}`"
              class="absolute bottom-2 right-2 hidden h-8 w-8 place-items-center rounded-full text-slate-400 transition hover:bg-rose-50 hover:text-rose-600 md:grid"
              @click.stop="deleteChat(chat.id)"
            >
              <svg stroke="currentColor" fill="none" stroke-width="1.7" viewBox="0 0 24 24" stroke-linecap="round" stroke-linejoin="round" class="h-4 w-4" xmlns="http://www.w3.org/2000/svg">
                <path d="M3 6h18"></path>
                <path d="M8 6V4h8v2"></path>
                <path d="M19 6l-1 14H6L5 6"></path>
                <path d="M10 11v5"></path>
                <path d="M14 11v5"></path>
              </svg>
            </button>
          </div>
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
                'rounded-tr-md bg-teal-600 text-white ring-teal-500 markdown-user': message.user === 'user'
              }"
            >
              <div class="markdown-body" v-html="renderMarkdown(message.content)"></div>
            </div>
          </div>
        </div>
      </div>

      <!-- 底部输入框 -->
      <div class="border-t border-white/80 bg-white/85 p-3 shadow-[0_-10px_30px_rgba(15,23,42,0.08)] backdrop-blur md:p-4">
        <el-input
          class="composer-input block w-full"
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
import hljs from 'highlight.js'
import MarkdownIt from 'markdown-it'
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

const markdown = new MarkdownIt({
  html: false,
  linkify: true,
  breaks: true,
  highlight(code, rawLanguage) {
    const language = String(rawLanguage || '').trim().split(/\s+/)[0].toLowerCase()
    const safeLanguage = language && hljs.getLanguage(language) ? language : ''

    try {
      const highlighted = safeLanguage
        ? hljs.highlight(code, { language: safeLanguage, ignoreIllegals: true }).value
        : hljs.highlightAuto(code).value

      const languageClass = safeLanguage ? ` language-${safeLanguage}` : ''
      return `<pre class="markdown-code"><code class="hljs${languageClass}">${highlighted}</code></pre>`
    } catch {
      return `<pre class="markdown-code"><code class="hljs">${escapeHtml(code)}</code></pre>`
    }
  }
})

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

function nowText(date = new Date()) {
  return date.toLocaleString('zh-CN', {
    timeZone: 'Asia/Shanghai',
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit',
    hour12: false
  }).replace(/\//g, '-')
}

function formatSessionTimestamp(chat) {
  const timestampMs = Number(chat.updatedAtMs || chat.createdAtMs)
  if (Number.isFinite(timestampMs) && timestampMs > 0) {
    return new Date(timestampMs).toLocaleString('zh-CN', {
      timeZone: 'Asia/Shanghai',
      month: '2-digit',
      day: '2-digit',
      hour: '2-digit',
      minute: '2-digit',
      hour12: false
    }).replace(/\//g, '-')
  }

  const rawTime = chat.updatedAt || chat.createdAt || ''
  const parsedMs = Date.parse(String(rawTime).replace(/-/g, '/'))
  if (Number.isFinite(parsedMs)) {
    return new Date(parsedMs).toLocaleString('zh-CN', {
      timeZone: 'Asia/Shanghai',
      month: '2-digit',
      day: '2-digit',
      hour: '2-digit',
      minute: '2-digit',
      hour12: false
    }).replace(/\//g, '-')
  }

  return rawTime
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

function normalizeSession(session, index = 0) {
  const fallbackMs = Date.now() - index
  const createdAtMs = Number(session.createdAtMs || session.updatedAtMs) || fallbackMs
  const updatedAtMs = Number(session.updatedAtMs || session.createdAtMs) || createdAtMs

  return {
    ...session,
    createdAt: session.createdAt || session.updatedAt || nowText(new Date(createdAtMs)),
    createdAtMs,
    updatedAt: session.updatedAt || session.createdAt || nowText(new Date(updatedAtMs)),
    updatedAtMs
  }
}

function buildSession(id, title, sourceMessages, existing = {}, options = {}) {
  const savedMessages = cloneMessages(sourceMessages)
  const lastMessage = [...savedMessages].reverse().find(message => message.content?.trim())
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
      Object.entries(parsed)
        .filter(([, session]) => session && Array.isArray(session.messages))
        .map(([sessionId, session], index) => [sessionId, normalizeSession(session, index)])
    )

    if (Object.keys(validSessions).length > 0) {
      sessions.value = validSessions
      persistSessions()
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

  if (deltaX < -56 && deltaY < 42) {
    swipedChatId.value = chatId
  } else if (deltaX > 24 || deltaY >= 42) {
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
            saveCurrentSession({ touchUpdatedAt: false })
            scrollToBottom()
          }
        }
      }
    )

    if (!aiMessage.content && typeof response.data === 'string') {
      aiMessage.content = response.data
      replaceMessage(aiMessage)
      saveCurrentSession({ touchUpdatedAt: false })
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
    saveCurrentSession({ touchUpdatedAt: false })
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
    saveCurrentSession({ touchUpdatedAt: false })
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

  saveCurrentSession({ touchUpdatedAt: false })
}

function renderMarkdown(content = '') {
  return markdown.render(String(content || ''))
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
.markdown-body {
  line-height: 1.75;
  overflow-wrap: anywhere;
}

.markdown-body :deep(p) {
  margin: 0 0 0.6rem;
}

.markdown-body :deep(p:last-child),
.markdown-body :deep(ul:last-child),
.markdown-body :deep(ol:last-child),
.markdown-body :deep(blockquote:last-child),
.markdown-body :deep(pre:last-child),
.markdown-body :deep(table:last-child) {
  margin-bottom: 0;
}

.markdown-body :deep(h1),
.markdown-body :deep(h2),
.markdown-body :deep(h3) {
  margin: 0.45rem 0 0.55rem;
  font-weight: 800;
  line-height: 1.35;
}

.markdown-body :deep(h1) {
  font-size: 1.25rem;
}

.markdown-body :deep(h2) {
  font-size: 1.12rem;
}

.markdown-body :deep(h3) {
  font-size: 1rem;
}

.markdown-body :deep(ul),
.markdown-body :deep(ol) {
  margin: 0.45rem 0 0.75rem 1.25rem;
  padding: 0;
}

.markdown-body :deep(ul) {
  list-style: disc;
}

.markdown-body :deep(ol) {
  list-style: decimal;
}

.markdown-body :deep(li) {
  margin: 0.2rem 0;
  padding-left: 0.05rem;
}

.markdown-body :deep(a) {
  color: #0f766e;
  font-weight: 600;
  text-decoration: underline;
  text-underline-offset: 3px;
}

.markdown-body :deep(code:not(pre code)) {
  border-radius: 7px;
  background: rgba(15, 23, 42, 0.08);
  padding: 0.12rem 0.38rem;
  color: #0f172a;
  font-size: 0.9em;
}

.markdown-body :deep(pre) {
  max-width: 100%;
  margin: 0.75rem 0;
  overflow-x: auto;
  border: 1px solid rgba(148, 163, 184, 0.35);
  border-radius: 14px;
  background: #f8fafc;
}

.markdown-body :deep(pre code) {
  display: block;
  min-width: max-content;
  padding: 1rem;
  border-radius: 14px;
  font-size: 0.9em;
  line-height: 1.65;
}

.markdown-body :deep(blockquote) {
  margin: 0.7rem 0;
  border-left: 3px solid #14b8a6;
  padding: 0.05rem 0 0.05rem 0.8rem;
  color: #475569;
}

.markdown-body :deep(table) {
  display: block;
  max-width: 100%;
  margin: 0.75rem 0;
  overflow-x: auto;
  border-collapse: collapse;
  font-size: 0.92em;
}

.markdown-body :deep(th),
.markdown-body :deep(td) {
  border: 1px solid #cbd5e1;
  padding: 0.45rem 0.6rem;
  text-align: left;
}

.markdown-body :deep(th) {
  background: #f1f5f9;
  font-weight: 700;
}

.markdown-user .markdown-body :deep(a),
.markdown-user .markdown-body :deep(code:not(pre code)) {
  color: #ffffff;
}

.markdown-user .markdown-body :deep(code:not(pre code)) {
  background: rgba(255, 255, 255, 0.18);
}

.markdown-user .markdown-body :deep(blockquote) {
  border-left-color: rgba(255, 255, 255, 0.72);
  color: rgba(255, 255, 255, 0.88);
}

.markdown-user .markdown-body :deep(pre),
.markdown-user .markdown-body :deep(table) {
  color: #0f172a;
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
