# VibeKey — 流星雨增强 + 性能优化 + 首屏过渡 专业提示词

> 角色：资深前端工程师 / Canvas 渲染优化专家  
> 目标：完成三项核心改进，提升视觉与体验至专业水准

---

## 一、密集流星雨增强

### 1.1 当前状态
```
- 流星每 0.8-1.8 秒生成一颗
- 同时最多 8 颗
- 方向：右上→左下，斜 30°-50°
```

### 1.2 目标效果
```
画面右上角区域应持续有流星划过的视觉密度。
不是"偶尔一颗"，而是"流星雨"的观感。

- 流星密度翻倍：每 0.3-0.8 秒生成一颗
- 同时最多 15 颗（从 8 提升）
- 批量生成机制：每 3-5 秒有一次"爆发"——
  同时生成 2-4 颗流星，角度略有差异（±8°），
  模拟真实的流星雨辐射点效果
- 流星间角度差异：3°-10°（从同一辐射点散开）
- 辐射点位置：画面右上角（x: 75%-95%, y: 0%-5%）
- 尾迹长度：180-400px（加长）
- 颜色：
  - 45% 暖金 #c4a87c（主流星群）
  - 25% 青白 #a0e8e0（快流星，速度 +30%）
  - 20% 淡紫 #d4b8f0（慢流星，尾迹更长）
  - 10% 纯白 #ffffff（超亮火流星，每 10-15 秒一颗）
- 火流星特殊效果：
  - 宽度 ×3（4-6px）
  - 尾迹长 400-600px
  - 头部光晕扩散到 8px
  - 飞过时整个画面亮度短暂 +5%
```

### 1.3 流星渲染优化（性能关键）
```
不要每颗流星用 createLinearGradient（开销大）：
→ 改用多段半透明线段叠加模拟拖尾
→ 头部用简单的 arc + fill
→ 拖尾用 5-8 段递减 opacity 的短线段
```

---

## 二、Canvas 性能优化（核心）

### 2.1 当前性能瓶颈诊断
```
主要问题：
1. createRadialGradient 调用过多（当前优化后仍有部分残留）
   - 月亮每帧 ~8 个径向渐变
   - 流星拖尾每帧 ~15 个线性渐变
   
2. 每帧全量清空+重绘
   - clearRect 清空整个 Canvas（w×h 像素）
   - 204 个活动对象每帧全部绘制
   
3. CSS 动画层与 Canvas 叠加
   - cosmos-deep / cosmos-nebula / cosmos-milkyway 三层的 CSS 动画
   - 每层触发 GPU 合成

4. devicePixelRatio 最大 2x
   - 4K 屏幕上 Canvas 分辨率 = 3840×2160×2 = 16.6M 像素
   - 每帧清除+绘制 16.6M 像素
```

### 2.2 优化方案（分优先级）

#### P0 — 关键路径（立即见效）
```
1. 彻底消灭 createRadialGradient / createLinearGradient
   - 月亮光晕：预计算 4 张离屏 Canvas（offscreen canvas），
     只在 resize 时重新渲染，draw 时直接 drawImage
   - 流星拖尾：用循环绘制 5-8 个递减 alpha 的小圆点
     替代 createLinearGradient
   - 萤火虫/光粒光晕：用 2-3 层嵌套的 arc + fillStyle 替代
     （已部分完成，检查是否有残留）

2. devicePixelRatio 限制为 1.5x（从 2x 降低）
   - 视觉效果几乎无差异
   - Canvas 像素量减少 44%

3. 离屏 Canvas 缓存月亮
   - 创建 moonCache（offscreen canvas）
   - 仅在 resize 时重绘月亮本体+光晕到 moonCache
   - 每帧只需 ctx.drawImage(moonCache, 0, 0)
   - 每帧节省 ~8 个 createRadialGradient 调用
```

#### P1 — 重要优化
```
4. 视口裁剪（Viewport Culling）
   - 跳过绘制画面外的粒子
   - 当前所有 204 个粒子每帧全量绘制
   - 加入简单边界检查：粒子坐标超出 viewport 就跳过

5. 减少 CSS 动画层
   - cosmos-deep 的 hue-rotate 120s → 移除动画（肉眼不可见的变化）
   - cosmos-nebula 的 30s transform → 改为 60s（减半 CPU 占用）
   - cosmos-milkyway 的 300s rotate → 保持不变（已经很慢）
```

#### P2 — 长期优化
```
6. 帧率自适应
   - 监控实际 FPS
   - 若 FPS < 45，自动降低粒子数：
     光粒 120→80，萤火虫 70→50，流星 max 15→10

7. 使用 OffscreenCanvas + Worker（如浏览器支持）
   - 将粒子更新逻辑移到 Worker 线程
   - 主线程只负责 drawImage
```

### 2.3 具体实现：离屏 Canvas 月亮缓存

```javascript
// 创建离屏 canvas
const moonCache = document.createElement('canvas');
const moonCtx = moonCache.getContext('2d');

function renderMoonToCache() {
  moonCache.width = w; moonCache.height = h;
  // 在 moonCtx 上绘制完整的月亮+光晕（使用径向渐变，但仅 resize 时调用）
  // ... 月亮绘制逻辑 ...
}

function drawMoon() {
  // 每帧只需一行：
  ctx.drawImage(moonCache, 0, 0);
}
```

### 2.4 具体实现：流星拖尾优化

```javascript
// 旧方案（昂贵）：
// const grad = ctx.createLinearGradient(...);
// grad.addColorStop(...); ctx.strokeStyle = grad; ctx.stroke();

// 新方案（廉价）：
function drawMeteorTrail(mx, my, vx, vy, life, color, segments = 6) {
  for (let i = 0; i < segments; i++) {
    const t = i / segments;
    const sx = mx - vx * t * 8;
    const sy = my - vy * t * 8;
    const alpha = life * (1 - t) * 0.7;
    const r = (1 - t * 0.7) * 1.5;
    ctx.fillStyle = `rgba(${color},${alpha})`;
    ctx.beginPath(); ctx.arc(sx, sy, r, 0, Math.PI * 2); ctx.fill();
  }
}
```

---

## 三、首屏 → 第二页 动态渐变过渡

### 3.1 需求描述
```
当用户从 Hero 视频首屏向下滚动，进入第二页（品牌定位区）时：
- Hero 内容不是突然消失
- 而是以动态渐变的方式优雅溶解
- 过渡过程中 Hero 视频逐渐被深色覆盖
- 第二页内容同时从下方浮现
```

### 3.2 实现方案

#### 方案 A：CSS 渐变遮罩 + IntersectionObserver（推荐）
```
1. 在 Hero section 底部添加一个渐变遮罩层
   - position: sticky / fixed（底部对齐）
   - height: 35vh
   - background: linear-gradient(to bottom, transparent 0%, #020208 100%)
   - 初始 opacity: 0

2. IntersectionObserver 监听 Hero 的可见比例
   - 当 Hero 可见比例从 100% 降到 20% 时
   - 渐变遮罩 opacity 从 0 线性过渡到 1
   - 根边距: rootMargin: '0px 0px -70% 0px'
   
3. Hero 的 scale/opacity 同时变化
   - Hero 整体 scale: 1 → 0.97（微缩）
   - Hero 整体 opacity: 1 → 0.3
   - 制造"远去"的感觉

4. 第二页内容同时做入场动画
   - 从下方 40px 淡入上浮
   - 与 Hero 的消失同步
```

#### CSS 代码
```css
/* Hero dissolve overlay */
.hero-dissolve {
  position: absolute;
  bottom: 0; left: 0; right: 0;
  height: 40vh;
  background: linear-gradient(
    to bottom,
    transparent 0%,
    rgba(2,2,8,.3) 30%,
    rgba(2,2,8,.7) 60%,
    #020208 100%
  );
  z-index: 6;
  pointer-events: none;
  opacity: 0;
  transition: opacity 0.6s cubic-bezier(0.4, 0, 0.2, 1);
}
.hero-dissolve.active {
  opacity: 1;
}

/* Hero scale-down */
.hero.scaling-down {
  transform: scale(0.97);
  opacity: 0.4;
  transition: transform 0.8s cubic-bezier(0.4, 0, 0.2, 1),
              opacity 0.8s cubic-bezier(0.4, 0, 0.2, 1);
}
```

#### JS 逻辑
```javascript
const heroObserver = new IntersectionObserver((entries) => {
  const e = entries[0];
  const ratio = e.intersectionRatio; // 1 = fully visible, 0 = fully gone
  
  // Dissolve effect proportional to scroll-out
  const dissolveProgress = 1 - Math.max(0, Math.min(1, ratio * 2));
  dissolveOverlay.style.opacity = dissolveProgress;
  
  // Scale down hero
  hero.style.transform = `scale(${1 - dissolveProgress * 0.03})`;
  hero.style.opacity = 1 - dissolveProgress * 0.6;
  
}, { threshold: [0, 0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1] });
```

---

## 四、完整执行清单

```
□ 1. 流星密度翻倍
     - max 8 → max 15
     - spawn interval 0.8-1.8s → 0.3-0.8s
     - 批量爆发：每 3-5s 同时生成 2-4 颗
     - 辐射点：右上角 75-95% x, 0-5% y
     - 火流星：每 10-15s 一颗（白色超亮）
     - 流星拖尾改用多段圆点渲染（省去 createLinearGradient）

□ 2. Canvas 性能优化
     - dpr 限制 1.5x（从 2x）
     - 月亮用离屏 Canvas 缓存（drawImage 替代每帧 8 个径向渐变）
     - 萤火虫/光粒确认无 createRadialGradient 残留
     - 流星拖尾改用圆点叠加（已包含在第 1 项）
     - 画面外粒子跳过绘制
     - CSS 动画减速：deep hue-rotate 移除，nebula 改为 60s

□ 3. 首屏过渡效果
     - Hero 底部添加 .hero-dissolve 渐变遮罩
     - IntersectionObserver 多阈值监听
     - Hero scale + opacity 随滚动降低
     - 第二页内容同步入场
     - 移除 Hero 旧的 IntersectionObserver（视频暂停逻辑保留）

□ 4. 验证
     - Chrome DevTools Performance 面板录制 5 秒滚动
     - 确认 FPS > 55
     - 确认内存无泄漏（粒子数量稳定）
```

---

## 五、验收标准

| 指标 | 当前 | 目标 |
|------|------|------|
| 流星视觉密度 | 偶尔一颗 | 持续流星雨，肉眼可见频繁划过 |
| Canvas 帧率 | ~30-40 FPS（卡顿） | ≥ 55 FPS（流畅） |
| createRadialGradient 调用 | ~15/帧 | 0/帧（仅在 resize 时调用） |
| 首屏过渡 | 生硬切换 | 渐变溶解，Hero 远去，第二页浮现 |
| Canvas 像素量 | w×h×2 | w×h×1.5 |
