# AI 旅游智能助手

基于 Vue 3 + TypeScript + 高德地图 API 的智能旅游规划应用

## 📢 项目声明

本项目由阿里云ESA提供加速、计算和保护

![阿里云ESA](https://img.alicdn.com/imgextra/i1/O1CN01H1UU3i1Cti9lYtFrs_!!6000000000139-2-tps-7534-844.png)
## ✨ 主要功能

- 🧠 **AI 智能规划**：基于大模型生成个性化旅游行程
- 🗺️ **实时地图展示**：集成高德地图，自动标记推荐景点
- 📝 **Markdown 输出**：结构化行程方案，支持复制到 Notion/飞书
- 🌤️ **实时数据查询**：天气、交通状况实时获取
- 📚 **历史记录管理**：localStorage 本地存储，支持清空操作
- 🎨 **现代化 UI**：暗色主题设计，支持多主题切换

## 🚀 快速开始

### 环境要求

- Node.js 16+
- npm 或 yarn

### 安装依赖

```bash
npm install
```

### 配置 API 密钥

#### 1. 高德地图 API 配置

1. 前往 [高德地图开放平台](https://lbs.amap.com/) 注册账号
2. 创建应用并获取以下 Key：
   - Web JS API Key（用于前端地图显示）
   - 安全密钥（如果控制台开启了安全码）

#### 2. DeepSeek API 配置

1. 前往 [DeepSeek 官网](https://platform.deepseek.com/) 注册账号
2. 获取 API Key

#### 3. 创建环境变量文件

在项目根目录创建 `.env` 文件，并填入以下配置：

```env
# 高德地图API配置
VITE_AMAP_KEY=
VITE_AMAP_SECURITY_JS_CODE=

# DeepSeek API配置
VITE_DEEPSEEK_API_KEY=
VITE_DEEPSEEK_API_BASE=https://api.deepseek.com/v1
VITE_DEEPSEEK_MODEL=deepseek-chat
```


**配置优先级**: `.env.local` → 系统环境变量 → 默认值（仅测试用）

### 启动开发服务器

```bash
npm run dev
```

访问 http://localhost:5173 查看应用。

### 构建生产版本

```bash
npm run build
```

## 🏗️ 项目结构

```
src/
├── components/
│   ├── TravelAssistant.vue  # 主组件（完整功能集成）
│   └── style.css           # 样式文件
├── App.vue                 # 应用入口
└── main.ts                 # Vue 应用初始化
```

## 🔧 技术栈

- **前端框架**: Vue 3 + TypeScript
- **构建工具**: Vite
- **地图服务**: 高德地图 JS API
- **Markdown 渲染**: Marked.js
- **状态管理**: Vue Composition API
- **数据存储**: localStorage



## 🎯 使用说明

1. 在右侧输入目标城市和旅游需求
2. 点击"开始规划"按钮
3. AI 将生成包含景点、路线、天气等信息的详细行程
4. 左侧可查看历史规划记录
5. 地图会自动标记推荐景点位置

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

## 📄 许可证

MIT License
