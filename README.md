# resume-3d

郭伟涛的 HTML 简历站点。**一个仓库、一个站点、一个链接** —— 打开就是完整简历，
页面里还内嵌了两个可以直接玩的小游戏。

- 线上地址（GitHub Pages）：<https://vilot888.github.io/resume-3d/>
- Surge：<https://guoweitao-resume.surge.sh/>
- 打印 / 导出 PDF 版：<https://vilot888.github.io/resume-3d/resume-print.html>

## 目录结构

```
resume-3d/
├── index.html            ← 站点入口：简历主页 + 游戏实验室
├── resume-print.html     ← 打印友好版（浅色，供浏览器另存为 PDF）
├── README.md
└── .nojekyll
```

**整站不到 100 KB，只有一个 HTML 里写的样式与脚本 —— 没有图片、没有字体、没有 JS 库、
没有任何外部请求。** 断网、拷到 U 盘、直接双击（除打印版外）都能跑。

## 游戏实验室

两个游戏都是原生 Canvas 2D + 原生 JS 手写，共用同一个 `requestAnimationFrame` 循环写法，
并且都做到了：只在滚进视口时运行（`IntersectionObserver`）、切到后台自动停、
支持鼠标与触屏（Pointer Events）、不在任何地方使用第三方库。

### A · 引力弹弓

- **玩法**：从发射台按住画布往反方向拖拽（像弹弓），虚线给出**预测轨迹**，松手发射。
  要按顺序穿过所有光环，最后飞进终点；撞上恒星或飞出边界算失败。
- **物理**：`a = Σ Gm·Δ/(|Δ|²+soften)^{3/2}`，每帧 4 个子步（`dt = 1/240`），半隐式欧拉积分。
  预测轨迹就是拿同一套积分器空跑 2.6 秒得到的点列。
- **关卡**：4 关，从单星绕行到三星收束。**每一关都保证可解** —— 见下文验证方式。
- 代码里留了一个 `window.__slingshot.solve(i)`，用暴力网格搜索找出一条通关弹道。

### B · 梯度下降大冒险

- **玩法**：你是一个优化器。在**看不见深浅**的损失曲面上，靠梯度下降滚到最低点。
  学习率调大一步飞出去，调小走不到底，还得判断自己是不是掉进了局部最优。
- **曲面**：`f = 0.42(x²+y²) − Σ aᵢ·exp(−d²/2σᵢ²)`，四个高斯井，其中一个最深（全局最优）。
  梯度是解析式，不是数值差分。
- **机制**：**迷雾探索** —— 曲面只在你滚过的地方揭开；每局跑完才显示真实深浅、
  全局最优位置与你的差距。还有带动量的更新（`v = μv − lr·∇f`）、限速防爆、loss 实时曲线、
  步数预算 600。历史最优 loss 存 `localStorage`。
- **门槛**：`gap = loss − globalMin`，`<0.02` 判到全局最优、`<0.15` 判接近、否则判掉进局部最优。

## 验证方式（不是"看着没问题"）

两个游戏都暴露了给自动化用的调试接口（`window.__slingshot` / `window.__descent`），
构建时用 Chrome DevTools Protocol 真的跑了一遍：

- 逐个关卡用 `__slingshot.solve(i)` 做暴力搜索，**确认 4 关都有通关弹道**（否则关卡不可解）
- 用 `__descent.setLr()` + `setStart()` 跑完整局，确认 loss 单调下降到全局最优附近
- 检查控制台无报错、无外部网络请求

复现命令见工作区 `tools/cdp-verify.mjs` 与 `tools/shots.mjs`。

## 本地预览

```bash
cd resume-3d
python -m http.server 8899
# 打开 http://127.0.0.1:8899/
```

深链参数：`?theme=light|dark` 指定主题。

## 部署

**GitHub Pages**（`main` 分支根目录发布，已启用）

```bash
git add -A && git commit -m "..." && git push origin main
```

**Surge**（同一份目录）

```bash
surge . guoweitao-resume.surge.sh
```

> 注：surge CLI ≥ 0.44 的内部请求层改用 undici `fetch`，在本机连 `surge.surge.sh` 会
> `UND_ERR_CONNECT_TIMEOUT`（原生 `https` 正常）。本项目实际使用 `surge@0.30.0`（axios 传输）
> 部署，结果一致。

## 更新照片

把照片存成 `assets/photo.jpg`，打开 `index.html` 里被注释掉的
`<img class="photo-img" src="assets/photo.jpg" …>` 即可（搜索 `photo-img`）。
