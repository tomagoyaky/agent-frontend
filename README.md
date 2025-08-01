# Agent Frontend - 智能AI助手前端界面

基于Vue 3 + Element Plus的现代化AI聊天前端界面。

## 🚀 特性

- **💬 实时聊天界面**: 流畅的对话体验
- **🌊 流式响应**: 实时显示AI回复过程
- **🔍 搜索集成**: 显示搜索结果和引用来源
- **🧠 思考过程**: 展示AI思考过程（支持的模型）
- **📱 响应式设计**: 适配桌面和移动设备
- **⚙️ 设置面板**: 动态配置LLM和搜索选项
- **💾 会话管理**: 完整的对话历史管理
- **🔄 错误重试**: 支持重试失败的消息

## 🛠️ 技术栈

- **Vue 3**: 现代化前端框架
- **Element Plus**: 优雅的UI组件库
- **Vite**: 快速的构建工具
- **JavaScript**: ES6+语法

## 📁 项目结构

```
agent-frontend/
├── src/
│   ├── components/        # Vue组件
│   ├── styles/           # 样式文件
│   ├── utils/            # 工具函数
│   ├── App.vue           # 主应用组件
│   └── main.js           # 应用入口
├── index.html            # HTML模板
├── package.json          # 项目配置
└── vite.config.js        # Vite配置
```

## 🛠️ 安装和运行

### 环境要求
- Node.js >= 18.0.0
- npm >= 8.0.0

### 安装依赖
```bash
cd agent-frontend
npm install
```

### 开发模式
```bash
npm run dev
```

### 生产构建
```bash
npm run build
```

### 预览构建结果
```bash
npm run preview
```

## 🔧 配置说明

### API端点配置
前端默认连接到后端服务：
- 开发环境: `http://localhost:3000`
- 生产环境: 根据部署配置调整

### 环境变量
可以通过环境变量配置：
```env
VITE_API_BASE_URL=http://localhost:3000
```

## 🎨 界面特性

- **深色/浅色主题**: 自适应系统主题
- **消息气泡**: 清晰的对话界面
- **搜索结果展示**: 引用来源链接
- **状态指示**: 消息发送/接收状态
- **错误处理**: 友好的错误提示
- **加载动画**: 流畅的交互反馈

## 🚀 部署建议

### 静态部署
- 构建后将`dist/`目录部署到静态服务器
- 支持Nginx、Apache、CDN等部署方式

### 容器化部署
- 可与后端一起打包为Docker镜像
- 支持Kubernetes部署

## 📱 浏览器兼容性

- Chrome (推荐)
- Firefox
- Safari
- Edge

## 🔗 相关链接

- [Vue 3 文档](https://vuejs.org/)
- [Element Plus 文档](https://element-plus.org/)
- [Vite 文档](https://vitejs.dev/)
