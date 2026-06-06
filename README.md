# VibeKey — 为 AI 编程而生

> 一把专为 AI 编程打造的机械键盘。兼容 Cursor · Claude Code · GitHub Copilot · Codex · Windsurf 等主流 AI 编程工具。

## 📁 项目结构

```
VibeKey/
├── sales.html              # 🌟 主销售页（全屏日月星辰特效）
├── index.html              # 🎬 产品介绍视频页
├── server.js               # 🔧 本地开发服务器
├── VibeKey_photos/         # 🖼️ 产品照片（6 款配色）
│   ├── vibe_hero_cream.jpg
│   ├── vibe_blue_morandi.jpg
│   ├── vibe_hacker_black.jpg
│   ├── vibe_mint_minecraft.jpg
│   ├── vibe_transparent_limited.jpg
│   └── vibe_lineup_5color.jpg
└── docs/prompts/           # 📝 设计提示词文档
    ├── VibeKey_销售页_设计提示词.md
    ├── VibeKey_video_脚本_完整方案.md
    ├── VibeKey_video_AI提示词卡片_即用版.md
    ├── VibeKey_流星雨_性能优化_过渡效果_专业提示词.md
    ├── VibeKey_粒子密度升级_资深前端提示词.md
    └── VibeKey_星云代码粒子_提示词.md
```

## 🚀 本地运行

```bash
# 启动开发服务器
node server.js

# 浏览器访问
# 销售页：http://localhost:8765/sales.html
# 视频页：http://localhost:8765/
```

## 🎨 设计特性

- **全局日月星辰特效**：单层 fixed Canvas，450+ 动态粒子，密集流星雨，圆月呼吸光晕
- **香槟金 + 灰蓝弥散星云**：CSS 多层渐变叠加
- **四角呼吸柔光**：4s 周期脉动
- **代码粒子**：浅绿色微型光点沿星轨漂移
- **首屏渐变溶解过渡**：Hero 滚动到第二页的动态淡出效果
- **6 款配色展示**：奶油白 / 莫兰迪蓝 / 极客黑 / 薄荷绿 / 透明限定 / 五色阵列
- **预售弹窗**：用户信息采集 + 配色选择

## 🛠️ 技术栈

- 纯 HTML/CSS/JS（零依赖）
- Canvas 2D 粒子引擎
- IntersectionObserver 滚动动画
- CSS 多层动画（星云漂移、银河旋转、呼吸光晕）
