# resume-3d

郭伟涛的 HTML 简历站点。**一个仓库、一个站点、一个链接** —— 打开就是完整简历，
页面里的 3D 可以直接拖拽旋转、也可以切到点云渲染。

- 线上地址（GitHub Pages）：<https://vilot888.github.io/resume-3d/>
- Surge：<https://guoweitao-resume.surge.sh/>
- 打印 / 导出 PDF 版：<https://vilot888.github.io/resume-3d/resume-print.html>

## 3D 部分是怎么来的

### A 档 · 真实摄影测量扫描

模型是 **珠海渔女的真实摄影测量扫描件**，不是手工建模：

- 原始数据：Agisoft PhotoScan 照片重建，**590,079 面 + 4096² 照片贴图**
- 来源：[Sketchfab · 珠海渔女](https://sketchfab.com/3d-models/none-192abbc700d743ed99910bdd109bd975)，作者 [一升VR](https://sketchfab.com/laisenglee32)
- 许可：**CC BY 4.0**（署名保留在页面与本文档中）

处理链路（`tools/from_scan.py`）：

1. **把 4K 照片贴图烘焙成逐顶点色** —— 363,036 个顶点各自从贴图里取一次真实像素色
2. **丢掉贴图**，用二次误差边坍缩减面到 150,000 面（`fast-simplification`），
   再用体素哈希把顶点色搬到新顶点上（减面**会移动顶点**，坐标精确匹配对不上）
3. **补一个石材材质** —— 必须有：glTF 在没有 material 时默认 `metallicFactor = 1.0`，
   石头会被渲染成抛光金属、颜色被高光冲淡；改成 `metallic 0 / roughness 0.82` 才像石头
4. 摆正坐标（扫描件是 Agisoft 的 Z-up → glTF 的 Y-up）、水平居中、底面落在 `y = 0`

结果：**24 MB → 4.4 MB，观感仍是照片级**。

### B 档 · 照片色点云

在减面后的扫描件表面按面积加权采样 **200,000** 个点，颜色直接取自照片贴图，
写出 32 字节/条的 `.splat`：

```
float32 × 3  位置 xyz
float32 × 3  缩放 sx sy sz   （世界单位，线性；切向 ≈ 法向的 2.6 倍）
uint8   × 4  RGBA
uint8   × 4  旋转四元数，编码为 round(q*128)+128
```

> **如实说明**：这一档是**照片色点云**（surface splatting），颜色是真实照片色，
> 但**不是**真正的 3D Gaussian Splatting —— 没有视角相关的球谐外观，
> 也没有从多视角照片拟合出的各向异性高斯。要换成真·3DGS，见 [`UPGRADE-SPLAT.md`](UPGRADE-SPLAT.md)。

### 电影感是怎么做出来的

1. **自建 HDR 环境**（`tools/make_hdr.py`）—— 程序化生成暖主光（方位 38°/仰角 34°）
   + 冷轮廓光（205°/14°）+ 弱补光的三点光环境贴图，写成 Radiance `.hdr`。
   默认的 `neutral` 环境是一张均匀灰图，出来就是商品展示的塑料感。
2. **ACES 色调映射**（`tone-mapping="aces"`）+ 曝光与阴影参数
3. **页面后期层**（纯 CSS）：宽银幕黑边、暗角、颗粒、轻微提饱和加对比，可一键开关
4. **静态降级图**不是另画的：用 `model-viewer.toDataURL()` 从 WebGL 画布取出真实渲染帧，
   再在 Python 里合成舞台背景 + 接触阴影 + 分离调色 + 高光泛光

## 目录结构

```
resume-3d/
├── index.html              ← 站点入口：简历主页 + 内嵌「3D 展区」
├── resume-print.html       ← 打印友好版（浅色，供浏览器另存为 PDF）
├── assets/
│   ├── model.glb           ← 珠海渔女真实扫描件（减面 + 顶点色烘焙，4.4 MB）
│   ├── scene.splat         ← 从扫描件表面采样的照片色点云（20 万点 / 6.1 MB）
│   ├── model-preview.jpg   ← 电影感静态渲染图（也是 WebGL 不可用时的降级图）
│   ├── studio.hdr          ← 程序化搭的三点光 HDR 环境
│   ├── model-viewer.min.js ← @google/model-viewer 3.5.0 本地副本
│   ├── gsplat.min.js       ← antimatter15/splat 渲染器本地副本（已改造）
│   └── splat.html          ← 点云渲染的同源宿主页（index.html 以 iframe 嵌入）
├── UPGRADE-SPLAT.md
├── README.md
└── .nojekyll
```
## 硬性约束的落实情况

| 约束 | 落实方式 |
|---|---|
| 一个站点、同源 | 所有资源都在本目录内，全部用**相对路径**引用；页面与 JS 里没有任何 CDN / 跨域地址 |
| 单文件 < 15 MB | 最大文件 `assets/scene.splat` = 6.1 MB（`model.glb` 4.4 MB、`studio.hdr` 2.0 MB） |
| 整站 < 30 MB | 约 13.8 MB |
| 加载进度 | A 档用 `model-viewer` 的 `progress` 事件；B 档由 iframe 通过 `postMessage` 回传解析进度 |
| 加载失败降级 | WebGL/WebGL2 不可用、脚本 404、**有进度就续期**的看门狗超时 → 自动切到 `model-preview.jpg` 并给出原因 |
| 低配模式 | 按钮切换。点云侧用 HTTP Range 只取前 14 万个点（不支持 Range 的托管方本地截断兜底），`devicePixelRatio` 固定为 1；模型侧关掉阴影与自动旋转 |
| 移动端 | 响应式栅格 + 触屏交互遮罩：手机上先让页面正常滚动，点一下才把指针事件交给 3D，避免"手指一放就滚不动" |
| 不改原件 | 微信收到的原始 HTML 只作参考，站点内是打磨后的新文件 |

## 第三方资源与许可

| 资源 | 版本 | 许可 | 说明 |
|---|---|---|---|
| 珠海渔女扫描件 | — | **CC BY 4.0** | 作者 [一升VR](https://sketchfab.com/laisenglee32)（Sketchfab），已减面 + 顶点色烘焙；署名保留在页面与本文档 |
| [`@google/model-viewer`](https://github.com/google/model-viewer) | 3.5.0 | Apache-2.0 | `assets/model-viewer.min.js`，原样本地化 |
| [`antimatter15/splat`](https://github.com/antimatter15/splat) | main | MIT | `assets/gsplat.min.js`，保留着色器与排序 Worker，仅改造 IO 与事件（文件头列出了全部改动） |

`gsplat.min.js` 相对上游的改动（构建脚本见工作区 `tools/patch-splat.mjs`）：

1. 资源路径由远程改成同站相对路径 `assets/scene.splat`，并支持 `window.SPLAT_OPTS` 覆盖
2. 尺寸以 canvas 容器为准（上游写死全屏 `innerWidth/innerHeight`）
3. 滚轮 / 键盘 / 拖拽事件收归 canvas，不再劫持整页滚动与键盘
4. 低配模式：用 `Range` 请求只取前 N 个点；托管方忽略 Range 时本地截断兜底
5. 兼容缺失 `Content-Length`、206 分片响应与缓冲增长
6. **数据下载完成即视为「可看」**，不等排序 Worker 追完（否则低端设备会一直卡在加载遮罩里）
7. 暴露 `SPLAT_ON_PROGRESS` / `SPLAT_ON_READY` / `SPLAT_ON_ERROR` 给页面，默认机位改为本模型

## 本地预览

站点全部使用相对路径，但 `.glb` / `.splat` / `.hdr` 必须经 HTTP 读取（`file://` 会被 CORS 拦住），
所以在目录内起一个静态服务器即可：

```bash
cd resume-3d
python -m http.server 8899
# 打开 http://127.0.0.1:8899/
```

深链参数：`?tab=b` 直接打开点云档，`?low=1` 低配模式，`?theme=light|dark` 指定主题。

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

把照片存成 `assets/photo.jpg`，然后打开 `index.html` 里那段被注释掉的
`<img class="photo-img" src="assets/photo.jpg" …>` 即可（搜索 `photo-img`）。
