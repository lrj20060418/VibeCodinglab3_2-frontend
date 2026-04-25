# Frontend（Vue 3 + Vite）

## 安装与运行

```bash
cd frontend
npm install
npm run dev
```

默认开发地址通常是 `http://localhost:5173/`。

## V4：高德地图配置（地点管理）

1. 复制一份环境变量文件：

```bash
copy .env.local.example .env.local
```

2. 编辑 `.env.local`，填写你的高德 **Web 端（JS API）Key** 与 **安全密钥（securityJsCode）**：

- `VITE_AMAP_KEY`
- `VITE_AMAP_SECURITY_JS_CODE`

> 注意：`.env.local` 不要提交到仓库。

## 常见问题

如果安装依赖时出现 `ENOSPC: no space left on device`，请先清理磁盘空间后重试（例如清理临时文件、删除不必要的下载/缓存等）。

# Vue 3 + Vite

This template should help get you started developing with Vue 3 in Vite. The template uses Vue 3 `<script setup>` SFCs, check out the [script setup docs](https://v3.vuejs.org/api/sfc-script-setup.html#sfc-script-setup) to learn more.

Learn more about IDE Support for Vue in the [Vue Docs Scaling up Guide](https://vuejs.org/guide/scaling-up/tooling.html#ide-support).
