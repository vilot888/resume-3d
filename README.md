# resume-3d

郭伟涛的 HTML 简历站点。**一个仓库、一个站点、一个链接** —— 打开就是完整简历，
页面里的 3D 可以直接拖拽旋转、也可以切到高斯喷溅点云渲染。

- 线上地址（GitHub Pages）：<https://vilot888.github.io/resume-3d/>
- 打印 / 导出 PDF 版：<https://vilot888.github.io/resume-3d/resume-print.html>

## 目录结构

```
resume-3d/
├── index.html              ← 站点入口：简历主页 + 内嵌「3D 展区」
├── resume-print.html       ← 打印友好版（浅色，供浏览器另存为 PDF）
├── assets/
│   ├── model.glb           ← Python 程序化生成的多层古塔模型（约 1,800 面）
│   ├── scene.splat         ← 在 model.glb 表面采样 40 万个点得到的喷溅数据
│   ├── model-preview.jpg   ← 静态渲染图，WebGL 不可用时降级显示
│   ├── model-viewer.min.js ← @google/model-viewer 3.5.0 本地副本
│   ├── gsplat.min.js       ← antimatter15/splat 渲染器本地副本（已改造，见下）
│   └── splat.html          ← 喷溅渲染的同源宿主页（index.html 以 iframe 嵌入）
├── UPGRADE-SPLAT.md        ← 如何换成手机拍摄的照片级高斯喷溅
├── README.md
└── .nojekyll               ← 让 GitHub Pages 跳过 Jekyll 处理
```

## 硬性约束的落实情况

| 约束 | 落实方式 |
|---|---|
| 一个站点、同源 | 所有资源都在本目录内，全部用**相对路径**引用；页面与 JS 里没有任何 CDN / 跨域地址 |
| 单文件 < 15 MB | 最大文件 `assets/scene.splat` = 12.8 MB |
| 整站 < 30 MB | 约 13.5 MB |
| 加载进度 | A 档用 `model-viewer` 的 `progress` 事件；B 档由 iframe 通过 `postMessage` 回传解析进度 |
| 加载失败降级 | WebGL/WebGL2 不可用、脚本 404、超时（20 s / 45 s）→ 自动切到 `model-preview.jpg` 静态渲染图并给出原因 |
| 低配模式 | 按钮切换。喷溅侧用 HTTP Range 只取前 14 万个点，并把 `devicePixelRatio` 固定为 1；模型侧关掉阴影与自动旋转 |
| 移动端 | 响应式栅格 + 触屏交互遮罩：手机上先让页面正常滚动，点一下才把指针事件交给 3D，避免"手指一放就滚不动" |
| 不改原件 | 微信收到的原始 HTML 只作参考，站点内是打磨后的新文件 |

## 3D 部分是怎么来的

**A 档 · 可交互模型**

`trimesh` + `numpy` 用 box / cylinder / cone / icosphere 拼出五层塔身、八角攒尖屋面、
斗拱点缀、朱红立柱、台基踏道与塔刹；按面拆点写入顶点法线，得到硬边平直的风格化着色，
导出为 `model.glb`（含 `POSITION` / `NORMAL` / `COLOR_0`）。

**B 档 · 高斯喷溅（网格转喷溅）**

在 `model.glb` 的三角面上按面积加权均匀采样 **400,000** 个点，取面颜色、沿法线方向压扁
（切向尺度约为法向的 2.9 倍），写出 32 字节/条的 `.splat`：

```
float32 × 3  位置 xyz
float32 × 3  缩放 sx sy sz   （世界单位，线性）
uint8   × 4  RGBA
uint8   × 4  旋转四元数，编码为 round(q*128)+128
```

> **如实说明**：这一档是**网格转喷溅**，属于风格化点云渲染，**不是**多视角照片重建的照片级结果。
> 想换成真实重建，看 [`UPGRADE-SPLAT.md`](UPGRADE-SPLAT.md)。

## 第三方资源与许可

| 资源 | 版本 | 许可 | 说明 |
|---|---|---|---|
| [`@google/model-viewer`](https://github.com/google/model-viewer) | 3.5.0 | Apache-2.0 | `assets/model-viewer.min.js`，原样本地化 |
| [`antimatter15/splat`](https://github.com/antimatter15/splat) | main | MIT | `assets/gsplat.min.js`，保留着色器与排序 Worker，仅改造 IO 与事件（文件头列出了全部改动） |

`gsplat.min.js` 相对上游的改动（构建脚本见工作区 `tools/patch-splat.mjs`）：

1. 资源路径由远程改成同站相对路径 `assets/scene.splat`，并支持 `window.SPLAT_OPTS` 覆盖
2. 尺寸以 canvas 容器为准（上游写死全屏 `innerWidth/innerHeight`）
3. 滚轮 / 键盘 / 拖拽事件收归 canvas，不再劫持整页滚动与键盘
4. 低配模式：用 `Range` 请求只取前 N 个点
5. 兼容缺失 `Content-Length`、206 分片响应与缓冲增长
6. 暴露 `SPLAT_ON_PROGRESS` / `SPLAT_ON_READY` / `SPLAT_ON_ERROR` 给页面，默认机位改为本模型

## 本地预览

站点全部使用相对路径，但 `.glb` / `.splat` 必须经 HTTP 读取（`file://` 会被 CORS 拦住），
所以在目录内起一个静态服务器即可：

```bash
cd resume-3d
python -m http.server 8899
# 打开 http://127.0.0.1:8899/
```

## 部署

**GitHub Pages**（`main` 分支根目录发布，已启用）

```bash
git add -A
git commit -m "feat: 简历主页 + 内嵌 3D 展区"
git push origin main
```

推送后约 1 分钟生效：<https://vilot888.github.io/resume-3d/>

**Surge**

```bash
npx surge .        # 在 resume-3d 目录内执行
```

同一个目录、同一份内容，两个平台完全一致。

## 更新照片

把照片存成 `assets/photo.jpg`，然后打开 `index.html` 里那段被注释掉的
`<img class="photo-img" src="assets/photo.jpg" …>` 即可（搜索 `photo-img`）。
