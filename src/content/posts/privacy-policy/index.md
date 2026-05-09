---
title: 隐私政策
published: 2026-05-09T00:00:00
description: 本隐私政策适用于 AcoFork Blog（以下简称"我们"）。
image: "img/privacy-policy.png"
draft: false
hide: true
lang: ""
---

<script>
  import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '$lib/components/ui/table';
</script>

## Cookie 与本地存储

### 必要（始终加载）

以下服务在所有页面加载时运行，无法通过 Cookie 设置关闭：

<Table>
  <TableHeader>
    <TableRow>
      <TableHead>服务</TableHead>
      <TableHead>说明</TableHead>
    </TableRow>
  </TableHeader>
  <TableBody>
    <TableRow>
      <TableCell><a href="https://u.2x.nz">Umami</a>（自托管）</TableCell>
      <TableCell>收集匿名访问数据，用于统计浏览量</TableCell>
    </TableRow>
    <TableRow>
      <TableCell><a href="https://www.cloudflare.com/analytics/">Cloudflare Web Analytics</a></TableCell>
      <TableCell>收集匿名访问数据，无 Cookie、无指纹追踪</TableCell>
    </TableRow>
    <TableRow>
      <TableCell>CF Umami（辅助统计）</TableCell>
      <TableCell>用于全站浏览量计数，同时作为浏览量 API 端点</TableCell>
    </TableRow>
  </TableBody>
</Table>

### 功能（需用户同意）

<Table>
  <TableHeader>
    <TableRow>
      <TableHead>服务</TableHead>
      <TableHead>说明</TableHead>
    </TableRow>
  </TableHeader>
  <TableBody>
    <TableRow>
      <TableCell><a href="https://giscus.app">Giscus</a></TableCell>
      <TableCell>基于 GitHub Discussions 的评论系统，用户交互时受 <a href="https://docs.github.com/en/site-policy/privacy-policies/github-privacy-statement">GitHub 隐私政策</a>约束</TableCell>
    </TableRow>
  </TableBody>
</Table>

### 分析（需用户同意）

<Table>
  <TableHeader>
    <TableRow>
      <TableHead>服务</TableHead>
      <TableHead>说明</TableHead>
    </TableRow>
  </TableHeader>
  <TableBody>
    <TableRow>
      <TableCell><a href="https://tongji.baidu.com/">百度统计</a></TableCell>
      <TableCell>收集站点访问情况</TableCell>
    </TableRow>
    <TableRow>
      <TableCell><a href="https://analytics.google.com">Google Analytics (GA4)</a></TableCell>
      <TableCell>收集站点访问情况</TableCell>
    </TableRow>
    <TableRow>
      <TableCell><a href="https://clarity.microsoft.com">Microsoft Clarity</a></TableCell>
      <TableCell>收集用户行为分析数据</TableCell>
    </TableRow>
  </TableBody>
</Table>

### 本地存储

我们使用浏览器 `localStorage` 存储以下数据：

<Table>
  <TableHeader>
    <TableRow>
      <TableHead>键名</TableHead>
      <TableHead>用途</TableHead>
    </TableRow>
  </TableHeader>
  <TableBody>
    <TableRow>
      <TableCell><code>cookie-consent-preferences</code></TableCell>
      <TableCell>Cookie 同意偏好设置</TableCell>
    </TableRow>
    <TableRow>
      <TableCell><code>theme</code></TableCell>
      <TableCell>用户主题偏好（亮色/暗色/跟随系统）</TableCell>
    </TableRow>
    <TableRow>
      <TableCell>论坛相关键名</TableCell>
      <TableCell>论坛登录凭证及环境配置</TableCell>
    </TableRow>
  </TableBody>
</Table>

## 其他第三方服务

### 论坛验证

论坛页面（`/forum/*`）使用 [Cloudflare Turnstile](https://www.cloudflare.com/products/turnstile/) 进行人机验证，用于登录、注册、发帖和评论操作。Turnstile 会收集浏览器信息用于机器人检测。

### 按需加载的 CDN 资源

以下资源仅在页面包含相关内容时按需加载：

<Table>
  <TableHeader>
    <TableRow>
      <TableHead>资源</TableHead>
      <TableHead>CDN</TableHead>
      <TableHead>用途</TableHead>
    </TableRow>
  </TableHeader>
  <TableBody>
    <TableRow>
      <TableCell><a href="https://mermaid.js.org">Mermaid.js</a></TableCell>
      <TableCell>jsdelivr</TableCell>
      <TableCell>渲染流程图、时序图等图表</TableCell>
    </TableRow>
    <TableRow>
      <TableCell><a href="https://highlightjs.org">Highlight.js</a></TableCell>
      <TableCell>cdnjs</TableCell>
      <TableCell>代码块语法高亮</TableCell>
    </TableRow>
    <TableRow>
      <TableCell><a href="https://dashjs.org">dash.js</a></TableCell>
      <TableCell>cdn.dashjs.org</TableCell>
      <TableCell>DASH 视频播放</TableCell>
    </TableRow>
    <TableRow>
      <TableCell><a href="https://iconify.design">Iconify</a></TableCell>
      <TableCell>api.iconify.design</TableCell>
      <TableCell>图标资源</TableCell>
    </TableRow>
  </TableBody>
</Table>

### 第三方图片源

页面中可能加载来自以下来源的图片：

- QQ 头像 API（`q2.qlogo.cn`）
- 阿里云 CDN（`img.alicdn.com`）

## 自托管服务

以下服务由我们自行托管，数据不经过第三方：

<Table>
  <TableHeader>
    <TableRow>
      <TableHead>服务</TableHead>
      <TableHead>地址</TableHead>
      <TableHead>用途</TableHead>
    </TableRow>
  </TableHeader>
  <TableBody>
    <TableRow>
      <TableCell>博客主站</TableCell>
      <TableCell><code>2x.nz</code></TableCell>
      <TableCell>文章发布与展示</TableCell>
    </TableRow>
    <TableRow>
      <TableCell>Umami 统计</TableCell>
      <TableCell><code>u.2x.nz</code></TableCell>
      <TableCell>匿名访问统计</TableCell>
    </TableRow>
    <TableRow>
      <TableCell>论坛后端</TableCell>
      <TableCell><code>i.2x.nz</code></TableCell>
      <TableCell>论坛数据存储与 API</TableCell>
    </TableRow>
  </TableBody>
</Table>

## 数据控制

您可以通过页面底部的「Cookie 设置」按钮随时修改 Cookie 同意偏好。清除浏览器 `localStorage` 可删除本地存储的所有数据。
