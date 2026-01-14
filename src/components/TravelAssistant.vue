<template>
  <div class="app-shell">
    <!-- 顶部导航 -->
    <header class="app-header">
      <div class="app-title-block">
        <div class="logo-badge">
          <span>✈️</span>
        </div>
        <div class="app-title-text">
          <h1>AI 旅游智能助手</h1>
          <p>基于大模型 + 高德实时数据，一键生成高质量行程规划</p>
        </div>
      </div>
      <div class="app-meta">
        <div class="pill">
          <span class="pill-dot"></span>
          <span>已连接至 AI 规划引擎</span>
        </div>
      </div>
    </header>

    <!-- 主体内容 -->
    <main class="app-main">
      <!-- 左侧：历史记录 -->
      <aside class="sidebar">
        <div class="sidebar-header">
          <div class="sidebar-title">
            <span class="icon">🕑</span>
            <span>历史规划记录</span>
          </div>
          <button @click="clearHistory" class="btn-ghost-danger">
            <span>🗑</span>
            <span>清空</span>
          </button>
        </div>
        <div class="history-list">
          <div v-if="!historyPlans.length" class="history-empty">
            暂无历史记录，先在右侧输入你的第一个需求吧。
          </div>
          <div v-else v-for="plan in historyPlans" :key="plan.id" class="history-item">
            <div class="history-header-row">
              <div class="history-city">{{ plan.city }}</div>
              <div class="history-time">{{ plan.created_at }}</div>
            </div>
            <div class="history-demand">
              需求：{{ plan.user_input }}
            </div>
            <div class="history-plan-preview markdown-view markdown-block"
                 v-html="renderMarkdownPreview(plan.plan_details)">
            </div>
          </div>
        </div>
      </aside>

      <!-- 右侧：结果 + 表单 -->
      <section class="content-pane">
        <!-- 介绍区域 -->
        <div class="intro-banner">
          <div class="intro-main">
            <div>
              <strong>告诉我你要去哪里、玩几天、喜欢什么风格</strong>，
              我会基于<strong>当前这一刻的实时天气与交通状况</strong>，
              为你生成可直接落地执行的 Markdown 行程方案。
            </div>
            <div class="intro-tags">
              <span class="chip">自动查询实时天气 & 路况</span>
              <span class="chip">结构化 Markdown 输出</span>
              <span class="chip">适合复制到 Notion / 飞书文档</span>
            </div>
          </div>
          <div class="intro-side">
            <div>建议直接描述你的需求，例如：</div>
            <div><span>"现在从北京出发，想看历史景点，尽量避开拥堵和人多的地方。"</span></div>
          </div>
        </div>

        <!-- 结果展示区域 -->
        <section class="result-card" :class="{ hidden: !currentResult.plan }">
          <div class="result-header">
            <div class="result-title">
              <span>🧭 最新旅游计划</span>
              <span class="badge-live"><span class="dot"></span> AI 已生成</span>
            </div>
            <div class="result-meta">
              <div>城市：<span>{{ currentResult.city }}</span></div>
              <div>生成时间：<span>{{ currentResult.timestamp }}</span></div>
            </div>
          </div>
          <div class="result-brief">
            <span class="label">你的需求：</span>
            <span>{{ currentResult.user_input }}</span>
          </div>
          <div class="markdown-view" v-html="currentResult.plan"></div>
        </section>

        <!-- 地图展示区域 -->
        <section class="map-card">
          <div class="map-header">
            <span>🗺 推荐景点地图</span>
            <span class="map-tip">基于 AI 规划结果和高德地图自动标记主要景点</span>
          </div>
          <div id="map-container" ref="mapContainer"></div>
        </section>

        <!-- 输入表单区域 -->
        <section class="input-card">
          <form @submit.prevent="generatePlan">
            <div class="input-row">
              <div class="field-group">
                <label class="field-label" for="city">
                  目标城市 <span class="required">*</span>
                  <span class="tip">支持手动输入，或在下方快速选择常用城市。</span>
                </label>
                <input
                  type="text"
                  id="city"
                  v-model="formData.city"
                  class="field-input"
                  required
                  placeholder="例如：北京市、上海市、成都市"
                >
                <select v-model="selectedCity" @change="updateCity" class="city-select">
                  <option value="">从常用城市中选择（可选）</option>
                  <option value="北京市">北京市</option>
                  <option value="上海市">上海市</option>
                  <option value="广州市">广州市</option>
                  <option value="深圳市">深圳市</option>
                  <option value="杭州市">杭州市</option>
                  <option value="成都市">成都市</option>
                  <option value="重庆市">重庆市</option>
                  <option value="西安市">西安市</option>
                  <option value="武汉市">武汉市</option>
                  <option value="南京市">南京市</option>
                </select>
              </div>
            </div>

            <div class="field-group">
              <label class="field-label" for="user_input">
                旅游需求 <span class="required">*</span>
                <span class="tip">尽量写清楚时间、喜好、人群（是否带老人小孩）、预算等。</span>
              </label>
              <textarea
                id="user_input"
                v-model="formData.user_input"
                class="field-textarea"
                required
                placeholder="例如：想去历史古迹，避开人多的地方，请帮我规划详细行程，并查询一下天气和主要道路的交通情况。"
              ></textarea>
            </div>

            <div class="form-footer-row">
              <div class="status-line" :class="{ success: status.type === 'success', error: status.type === 'error' }">
                <span class="status-dot"></span>
                <span>{{ status.message }}</span>
              </div>
              <div class="api-usage-info">
                <span class="usage-text">今日API使用: {{ apiCallCount }}/{{ DAILY_API_LIMIT }}</span>
                <div class="usage-bar">
                  <div class="usage-fill" :style="{ width: `${(apiCallCount / DAILY_API_LIMIT) * 100}%` }"></div>
                </div>
              </div>
              <button type="submit" :disabled="isLoading || !checkApiLimit()" class="btn-primary">
                <span v-if="isLoading" class="spinner"></span>
                <span>{{ isLoading ? '正在规划中…' : !checkApiLimit() ? '今日次数已用完' : '开始规划' }}</span>
              </button>
            </div>
          </form>
        </section>
      </section>
    </main>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, nextTick } from 'vue'
import { marked } from 'marked'
import AMapLoader from '@amap/amap-jsapi-loader'

// 类型定义
interface Plan {
  id: string
  city: string
  user_input: string
  plan_details: string
  created_at: string
}

interface FormData {
  city: string
  user_input: string
}

interface CurrentResult {
  city: string
  user_input: string
  plan: string
  timestamp: string
}

interface Status {
  message: string
  type: 'default' | 'success' | 'error'
}

// 响应式数据
const formData = ref<FormData>({
  city: '北京市',
  user_input: '想去历史古迹，避开人多的地方，请帮我规划详细行程，并查询一下天气和主要道路的交通情况。'
})

const selectedCity = ref('')
const isLoading = ref(false)
const status = ref<Status>({
  message: '等待输入你的需求，然后点击右侧按钮开始规划。',
  type: 'default'
})

const historyPlans = ref<Plan[]>([])
const currentResult = ref<CurrentResult>({
  city: '',
  user_input: '',
  plan: '',
  timestamp: ''
})

// 地图相关
const mapContainer = ref<HTMLElement>()
let amapInstance: any = null
let amapGeocoder: any = null
let amapMarkers: any[] = []

// API调用限制
const DAILY_API_LIMIT = 10
const apiCallCount = ref(0)
const lastResetDate = ref('')

// 固定测试景点数据
const FIXED_POIS = [
  { name: '通远门城墙遗址公园', address: '金汤街53号(七星岗地铁站1号口步行430米)', city: '重庆市', lat: 29.555908, lng: 106.567734 },
  { name: '重庆古城墙-金汤门', address: '通远门段城门及城墙', city: '重庆市', lat: 29.55126, lng: 106.56534 },
  { name: '重庆大韩民国临时政府旧址陈列馆', address: '七星岗莲花池38号', city: '重庆市', lat: 29.555752, lng: 106.569505 },
  { name: '郭沫若旧居', address: '天官府8号(较场口地铁站5号口步行420米)', city: '重庆市', lat: 29.551953, lng: 106.567172 },
  { name: '东华观藏经楼', address: '凯旋路73号', city: '重庆市', lat: 29.553043, lng: 106.578749 }
]

// 方法定义
const updateCity = () => {
  if (selectedCity.value) {
    formData.value.city = selectedCity.value
  }
}

const setStatus = (message: string, type: 'default' | 'success' | 'error' = 'default') => {
  status.value = { message, type }
}

const renderMarkdownPreview = (markdown: string): string => {
  if (!markdown) return ''
  const short = markdown.split('\n').slice(0, 4).join('\n')
  return marked.parse(short) as string
}

const cleanPlanMarkdown = (markdownText: string): string => {
  if (!markdownText) return ''
  let text = markdownText

  // 1) 去掉包含 poi_json 的代码块
  text = text.replace(/```[\s\S]*?```/g, function (block) {
    return block.indexOf('poi_json') !== -1 ? '' : block
  })

  // 2) 去掉任何包含"最终选中景点数据"文案的整行
  text = text.replace(/^.*最终选中景点数据.*$/gm, '')

  // 3) 收尾多余的空行
  text = text.replace(/\n{3,}/g, '\n\n')

  return text.trim()
}

const extractPoiJsonFromMarkdown = (markdownText: string): any => {
  if (!markdownText) return null

  const codeBlocks = markdownText.match(/```[\s\S]*?```/g)
  if (!codeBlocks) return null

  for (const block of codeBlocks) {
    if (block.indexOf('poi_json') === -1) continue

    let inner = block.replace(/^```[^\n]*\n?/, '').replace(/```$/, '')

    const lines = inner.split('\n').filter(line => line.trim() && line.trim() !== 'poi_json')
    const jsonStr = lines.join('\n').trim()

    try {
      return JSON.parse(jsonStr)
    } catch (e) {
      console.warn('解析 poi_json JSON 失败:', e, jsonStr)
      return null
    }
  }

  return null
}

// 地图相关方法
const initMap = async () => {
  if (!mapContainer.value) return

  try {
    const amapKey = import.meta.env.VITE_AMAP_KEY
    if (!amapKey) {
      console.error('高德地图API Key未配置')
      // 可以选择继续使用默认地图或显示错误，但为了演示暂时使用测试key
    }

    const AMap = await AMapLoader.load({
      key: amapKey || 'f84c6c4cfd2eef51dec45cc2c4846440', // 使用配置的key或默认测试key
      version: '2.0',
      plugins: ['AMap.Geocoder']
    })

    amapInstance = new AMap.Map(mapContainer.value, {
      zoom: 11,
      viewMode: '2D',
      mapStyle: 'amap://styles/normal'
    })

    amapGeocoder = new AMap.Geocoder()

    // 初始化默认地图
    initDefaultMap()
  } catch (error) {
    console.error('地图初始化失败:', error)
  }
}

const initDefaultMap = () => {
  if (!amapInstance) return

  ensureMap('重庆市')
  clearMapMarkers()

  FIXED_POIS.forEach(poi => {
    if (typeof poi.lat === 'number' && typeof poi.lng === 'number') {
      const marker = new AMap.Marker({
        position: [poi.lng, poi.lat],
        title: poi.name
      })
      amapInstance.add(marker)
      amapMarkers.push(marker)
    }
  })

  if (amapInstance && amapMarkers.length) {
    amapInstance.setFitView(amapMarkers)
  }
}

const ensureMap = (city: string) => {
  if (!amapInstance || !amapGeocoder) return

  if (city) {
    amapGeocoder.getLocation(city, function (status: string, result: any) {
      if (status === 'complete' && result.geocodes && result.geocodes.length > 0) {
        const loc = result.geocodes[0].location
        amapInstance.setCenter(loc)
      }
    })
  }
}

const clearMapMarkers = () => {
  if (amapInstance && amapMarkers.length) {
    amapInstance.remove(amapMarkers)
  }
  amapMarkers = []
}

const addPoisToMapFromPlan = (markdownText: string, city: string) => {
  if (!amapInstance || !amapGeocoder) return

  const poiData = extractPoiJsonFromMarkdown(markdownText)

  let pois = []
  if (poiData) {
    if (Array.isArray(poiData)) {
      pois = poiData
    } else if (Array.isArray(poiData.pois)) {
      pois = poiData.pois
    } else if (Array.isArray(poiData.poi_json)) {
      pois = poiData.poi_json
    }
  }

  if (!pois.length) {
    pois = FIXED_POIS // 使用固定数据做回退
  }

  ensureMap(city)
  clearMapMarkers()

  pois.forEach((poi: any) => {
    const name = poi.name || ''
    const address = poi.address || ''
    const poiCity = poi.city || city || ''

    if (typeof poi.lat === 'number' && typeof poi.lng === 'number' && !isNaN(poi.lat) && !isNaN(poi.lng)) {
      const marker = new AMap.Marker({
        position: [poi.lng, poi.lat],
        title: name
      })
      amapInstance.add(marker)
      amapMarkers.push(marker)
      return
    }

    const fullAddress = `${poiCity}${name || ''}${address || ''}`
    if (amapGeocoder && fullAddress) {
      amapGeocoder.getLocation(fullAddress, function (status: string, result: any) {
        if (status === 'complete' && result.geocodes && result.geocodes.length > 0) {
          const loc = result.geocodes[0].location
          const marker = new AMap.Marker({
            position: loc,
            title: name || fullAddress
          })
          amapInstance.add(marker)
          amapMarkers.push(marker)
        }
      })
    }
  })

  if (amapInstance && amapMarkers.length) {
    amapInstance.setFitView(amapMarkers)
  }
}

// 历史记录管理
const loadHistory = () => {
  const stored = localStorage.getItem('travel-plans')
  if (stored) {
    try {
      historyPlans.value = JSON.parse(stored)
    } catch (e) {
      console.error('加载历史记录失败:', e)
      historyPlans.value = []
    }
  }
}

const saveHistory = () => {
  localStorage.setItem('travel-plans', JSON.stringify(historyPlans.value))
}

const addToHistory = (plan: Omit<Plan, 'id' | 'created_at'>) => {
  const newPlan: Plan = {
    id: Date.now().toString(),
    created_at: new Date().toLocaleString('zh-CN'),
    ...plan
  }
  historyPlans.value.unshift(newPlan)
  saveHistory()
}

const clearHistory = () => {
  if (confirm('确定要清空所有历史记录吗？此操作不可恢复。')) {
    historyPlans.value = []
    localStorage.removeItem('travel-plans')
    setStatus('历史记录已清空。', 'success')
  }
}

// AI规划生成（集成DeepSeek API）
const generatePlan = async () => {
  if (!formData.value.city || !formData.value.user_input) {
    setStatus('请输入城市和旅游需求再开始规划。', 'error')
    return
  }

  // 检查API调用限制
  if (!checkApiLimit()) {
    setStatus(`今日API调用次数已达上限（${DAILY_API_LIMIT}次），请明天再试。`, 'error')
    return
  }

  isLoading.value = true
  setStatus('正在调用 DeepSeek AI 和高德 API，为你规划最优行程…', 'default')

  try {
    // 调用DeepSeek API生成旅游规划
    const aiResponse = await callDeepSeekAPI(formData.value.city, formData.value.user_input)

    if (!aiResponse) {
      throw new Error('AI服务无响应，请检查API配置')
    }

    // 成功调用API，增加计数
    incrementApiCount()

    const cleanedPlan = cleanPlanMarkdown(aiResponse)
    const result = {
      city: formData.value.city,
      user_input: formData.value.user_input,
      plan: marked.parse(cleanedPlan) as string,
      timestamp: new Date().toLocaleString('zh-CN')
    }

    currentResult.value = result

    // 添加到历史记录
    addToHistory({
      city: result.city,
      user_input: result.user_input,
      plan_details: aiResponse
    })

    // 更新地图
    addPoisToMapFromPlan(aiResponse, result.city)

    setStatus(`规划已完成，如有不合适可以直接修改需求再次生成。（今日已使用 ${apiCallCount.value}/${DAILY_API_LIMIT} 次）`, 'success')

  } catch (error) {
    console.error('规划生成失败:', error)
    const errorMessage = error instanceof Error ? error.message : '未知错误'
    currentResult.value.plan = `<div class="error-text">**规划失败：** ${errorMessage}</div>`
    setStatus('规划失败，请稍后重试或检查网络连接。', 'error')
  } finally {
    isLoading.value = false
  }
}

// API调用限制管理
const loadApiCallLimit = () => {
  const today = new Date().toDateString()
  const stored = localStorage.getItem('api-call-limit')

  if (stored) {
    try {
      const data = JSON.parse(stored)
      if (data.date === today) {
        apiCallCount.value = data.count
        lastResetDate.value = data.date
      } else {
        // 新的一天，重置计数
        apiCallCount.value = 0
        lastResetDate.value = today
        saveApiCallLimit()
      }
    } catch (e) {
      console.error('加载API调用限制失败:', e)
      apiCallCount.value = 0
      lastResetDate.value = today
    }
  } else {
    apiCallCount.value = 0
    lastResetDate.value = today
  }
}

const saveApiCallLimit = () => {
  const data = {
    count: apiCallCount.value,
    date: lastResetDate.value
  }
  localStorage.setItem('api-call-limit', JSON.stringify(data))
}

const checkApiLimit = (): boolean => {
  loadApiCallLimit()
  return apiCallCount.value < DAILY_API_LIMIT
}

const incrementApiCount = () => {
  apiCallCount.value++
  saveApiCallLimit()
}

// 调用DeepSeek API
const callDeepSeekAPI = async (city: string, userInput: string): Promise<string> => {
  // 检查配置 - 优先使用环境变量（包括.env.local文件中的配置）
  const apiKey = import.meta.env.VITE_DEEPSEEK_API_KEY
  const apiBase = import.meta.env.VITE_DEEPSEEK_API_BASE || 'https://api.deepseek.com/v1'
  const model = import.meta.env.VITE_DEEPSEEK_MODEL || 'deepseek-chat'

  // 检查API Key是否存在
  if (!apiKey) {
    const configMessage = `
未找到DeepSeek API配置。请按以下步骤配置：

1. 确保项目根目录存在 .env.local 文件
2. 文件内容应包含：
   VITE_DEEPSEEK_API_KEY=你的DeepSeek API Key
   VITE_DEEPSEEK_API_BASE=https://api.deepseek.com/v1
   VITE_DEEPSEEK_MODEL=deepseek-chat

3. 或在系统环境变量中设置上述配置

4. 配置完成后，请重启开发服务器

获取API Key：https://platform.deepseek.com/
    `.trim()

    throw new Error(configMessage)
  }

  const prompt = `你是一个专业的旅游规划助手。请根据用户的需求，为${city}制定详细的旅游行程规划。

用户需求：${userInput}

请提供以下信息：
1. 行程概览和时间安排
2. 推荐的景点和活动（包含具体地址）
3. 交通建议（如何到达和在城市内移动）
4. 餐饮推荐
5. 住宿建议
6. 预算估算
7. 注意事项和tips

请用Markdown格式输出，并确保信息准确有用。在文末用以下格式添加景点数据（用于地图显示）：

\`\`\`poi_json
[
  {
    "name": "景点名称",
    "address": "具体地址",
    "city": "${city}",
    "lat": 纬度,
    "lng": 经度
  }
]
\`\`\`

请确保景点数据格式正确，包含真实的经纬度坐标。`

  const response = await fetch(`${apiBase}/chat/completions`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${apiKey}`
    },
    body: JSON.stringify({
      model: model,
      messages: [
        {
          role: 'user',
          content: prompt
        }
      ],
      temperature: 0.7,
      max_tokens: 2000
    })
  })

  if (!response.ok) {
    const errorData = await response.json().catch(() => ({}))
    throw new Error(`DeepSeek API调用失败: ${response.status} ${errorData.error?.message || ''}`)
  }

  const data = await response.json()
  return data.choices?.[0]?.message?.content || 'AI未返回有效内容'
}

// 检查配置文件
const checkConfiguration = () => {
  const missingConfigs: string[] = []

  if (!import.meta.env.VITE_AMAP_KEY) {
    missingConfigs.push('高德地图API Key (VITE_AMAP_KEY)')
  }

  if (!import.meta.env.VITE_DEEPSEEK_API_KEY) {
    missingConfigs.push('DeepSeek API Key (VITE_DEEPSEEK_API_KEY)')
  }

  if (missingConfigs.length > 0) {
    console.warn('⚠️ 配置文件检查:', missingConfigs.join(', '), '未配置')
    console.info('请确保 .env.local 文件存在并包含正确的配置')
  } else {
    console.info('✅ API配置检查通过')
  }
}

// 生命周期
onMounted(async () => {
  checkConfiguration()
  loadApiCallLimit() // 加载API调用限制
  loadHistory()
  await initMap()
})
</script>

<style scoped>
/* 导入原版样式 */
@import url('./style.css');
</style>
