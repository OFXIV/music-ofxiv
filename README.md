# Music OFXIV
- 每天两次更新歌单内容(UTC 00:00,UTC 12:00)，`XIV`14首歌
- 基于 Vue 3 + Vite 构建的音乐播放器应用

## 技术栈

- Vue 3
- Vite
- Vue Composition API
- 现代化 CSS

## 快速开始

### 安装依赖

```bash
npm install
```

### 开发环境运行

```bash
npm run dev
```

### 构建生产版本

```bash
npm run build
```

### 预览生产版本

```bash
npm run preview
```

## 数据源

歌曲列表通过远程 API 加载：
```
https://raw.githubusercontent.com/OFXIV/resources/refs/heads/main/json/music.json
```

## 开发指南

项目使用 Vue 3 的 `<script setup>` 语法，推荐查看 [Vue 3 文档](https://v3.vuejs.org/) 了解更多细节。