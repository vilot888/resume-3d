# 把「网格转喷溅」升级成手机拍摄的照片级高斯喷溅

现在页面 B 档用的 `assets/scene.splat`，是把 A 档模型（`model.glb`）的网格表面
均匀采样 40 万个点得到的 —— 它**没有真实照片**，属于**风格化点云渲染**。

要拿到真正的照片级结果，只能靠**多视角照片重建**：绕实物拍一圈照片，由算法反解出
每个点的位置、颜色、各向异性尺度和朝向。下面是从拍摄到替换的完整流程。

---

## 一、拍摄（最关键的一步，拍砸了后面全白搭）

**工具**：手机 + [Polycam](https://polycam.ai/)（iOS / Android，免费额度够用）或
[Luma AI](https://lumalabs.ai/)（iOS / Android）。

**对象**：选一个**静止、不反光、有丰富纹理**的东西。古建筑题材可选：小塔模型、
砖雕摆件、木构模型、石狮子小件。**别拍**玻璃、镜面不锈钢、纯白墙面、会动的物体。

**拍法**：

1. 光线用**阴天散射光**或室内多点布光；避免阳光直射（高光会被当成颜色烤进去）和闪光灯。
2. 围绕物体**走两到三圈**：
   - 第一圈：中高度，水平环绕，每步约 15°–20°，**总张数 30–60 张**；
   - 第二圈：抬高俯拍（约 30°–45° 俯角），补屋顶、塔檐这些俯视面；
   - 第三圈（可选）：降低仰拍，补台基、底面边缘。
3. 每张照片**相邻重叠 70% 以上**，走一步拍一张，别跳着拍。
4. 每张都要**对上焦**；关掉人像/美颜/广角畸变矫正（除非 App 明确要求）。
5. 物体在画面里占 **60%–80%**，四周留一点环境——背景里有纹理反而更容易对齐。
6. 全程**别改焦距、别开数码变焦**，用同一个镜头。
7. 拍摄期间物体**不能动**；如果放在转盘上转，那转的时候人别动，二选一，不要同时动。

**数量参考**：30–100 张。少于 30 张容易失败，多于 150 张手机端处理会很慢。

---

## 二、重建与导出（Polycam 为例）

1. 打开 Polycam → 右下角 `+` → 选 **Photo Mode**（不是 LiDAR，网页端要的是照片重建结果）。
2. 导入或现场拍摄全部照片 → **Upload** → 等待云端处理（几分钟到十几分钟）。
3. 处理完成后进入模型详情页：
   - **先看 Polycam 里的预览有没有破面、糊掉的区域**；有就回去补拍再重跑。
   - 依次导出需要的格式：
     - **Export → Gaussian Splat → `.ply`**（推荐，信息最全，颜色是球谐系数）
     - 或 **Export → Gaussian Splat → `.splat`**（能直接导出就用这个，跳到第四节）
4. Luma AI 同理：`Capture` → 环绕拍摄 → 完成后 `Export` → 选 Gaussian Splat / `.ply`。

> ⚠️ Polycam 免费账号对导出格式有限制，导出按钮灰掉就说明当前套餐不含该格式。

---

## 三、`.ply`（3DGS 格式）→ `.splat`

3DGS 的 `.ply` 里每个点有 **59 个 float**：位置 3、法线 3、球谐系数 48
（`f_dc_0..2` + `f_rest_0..44`）、不透明度 1、缩放 3、旋转四元数 4。

`.splat` 是它的紧凑化版本（32 字节/点）：

```
偏移  0..11  float32 × 3   位置 xyz
偏移 12..23  float32 × 3   缩放 sx sy sz   ← 线性尺度，不是 log
偏移 24..27  uint8   × 4   RGBA           ← 颜色取球谐的直流分量，alpha = sigmoid(opacity)
偏移 28..31  uint8   × 4   旋转四元数      ← round(q*128)+128
```

> **注意**：`.splat` 里的缩放是**线性**值（渲染器直接拿它算协方差），
> 而 `.ply` 里存的是 **log 尺度**，所以必须 `exp()` 还原。忘了 `exp()` 会让点云缩成一小团。
> 四元数顺序是 `(w, x, y, z)`，与上游 antimatter15/splat 的着色器一致。

把下面这段保存成 `ply2splat.py`，与导出的 `.ply` 放同一目录后运行：

```python
"""3DGS 的 .ply 转 antimatter15/splat 的 .splat（32 字节/点）。

用法:
    python ply2splat.py input.ply output.splat [最大点数]
"""
import sys
import numpy as np

SH_C0 = 0.28209479177387814          # 球谐 0 阶系数


def sigmoid(x):
    return 1.0 / (1.0 + np.exp(-x))


def read_ply_header(path):
    with open(path, "rb") as fh:
        if fh.readline().strip() != b"ply":
            raise SystemExit("不是 ply 文件")
        fmt = fh.readline().strip()
        if b"binary_little_endian" not in fmt:
            raise SystemExit("只支持 binary_little_endian 的 ply")
        count, props, reading = 0, [], False
        while True:
            line = fh.readline()
            if not line:
                raise SystemExit("ply 头不完整")
            t = line.strip()
            if t.startswith(b"element vertex"):
                count = int(t.split()[-1]); reading = True
            elif t.startswith(b"element"):
                reading = False
            elif t.startswith(b"property") and reading:
                props.append(t.split()[-1].decode())
            elif t == b"end_header":
                break
        return count, props, fh.tell()


def main():
    src, dst = sys.argv[1], sys.argv[2]
    cap = int(sys.argv[3]) if len(sys.argv) > 3 else 0

    n, props, offset = read_ply_header(src)
    dtype = np.dtype([(p, "<f4") for p in props])
    data = np.fromfile(src, dtype=dtype, count=n, offset=offset)
    if cap and n > cap:                                   # 均匀抽稀，保持空间分布
        pick = np.linspace(0, n - 1, cap).astype(np.int64)
        data = data[pick]
    n = len(data)

    xyz = np.stack([data["x"], data["y"], data["z"]], axis=1).astype("<f4")

    scale = np.exp(np.stack([data["scale_0"], data["scale_1"], data["scale_2"]], axis=1)).astype("<f4")

    # 颜色取球谐直流分量：rgb = clamp(0.5 + SH_C0 * f_dc, 0, 1)
    rgb = np.stack([data["f_dc_0"], data["f_dc_1"], data["f_dc_2"]], axis=1)
    rgb = np.clip(0.5 + SH_C0 * rgb, 0.0, 1.0)
    alpha = sigmoid(data["opacity"])
    rgba = np.concatenate([(rgb * 255).astype(np.uint8),
                           (alpha * 255).astype(np.uint8)[:, None]], axis=1)

    q = np.stack([data["rot_0"], data["rot_1"], data["rot_2"], data["rot_3"]], axis=1)
    q /= np.maximum(np.linalg.norm(q, axis=1, keepdims=True), 1e-9)
    q_u8 = np.clip(np.rint(q * 128.0) + 128.0, 0, 255).astype(np.uint8)

    rec = np.zeros((n, 32), dtype=np.uint8)
    rec[:, 0:12] = xyz.view(np.uint8).reshape(n, 12)
    rec[:, 12:24] = scale.view(np.uint8).reshape(n, 12)
    rec[:, 24:28] = rgba
    rec[:, 28:32] = q_u8
    rec.tofile(dst)

    print(f"写入 {dst}：{n} 个点，{n * 32 / 1048576:.2f} MB")
    print("模型包围盒:",
          np.round(xyz.min(axis=0), 3).tolist(), "→", np.round(xyz.max(axis=0), 3).tolist())


if __name__ == "__main__":
    main()
```

```bash
# 点数太多会卡手机，建议压到 40 万以内（与当前口径一致）
python ply2splat.py my_capture.ply scene.splat 400000
```

---

## 四、替换进站点

```bash
cp scene.splat resume-3d/assets/scene.splat     # 覆盖同名文件即可
```

站点代码不需要改，但**相机会因为尺度不同而需要重新对位**。真实重建的模型通常不在
原点、朝向也不一致，所以按下面的顺序调一次就行：

1. **看包围盒**。上一步脚本会打印 `模型包围盒`。记下三个方向的实际尺寸。
2. **重新居中**（可选但强烈建议）。写个一次性小脚本把点云平移到「底面在 y=0、水平居中」：

   ```python
   import numpy as np
   a = np.fromfile("assets/scene.splat", dtype=np.uint8).reshape(-1, 32)
   pos = a[:, 0:12].copy().view("<f4").reshape(-1, 3)
   pos[:, 0] -= (pos[:, 0].min() + pos[:, 0].max()) / 2
   pos[:, 2] -= (pos[:, 2].min() + pos[:, 2].max()) / 2
   pos[:, 1] -= pos[:, 1].min()
   a[:, 0:12] = pos.astype("<f4").view(np.uint8).reshape(-1, 12)
   a.tofile("assets/scene.splat")
   ```

3. **改默认机位**。打开 `assets/gsplat.min.js`，找到这一行：

   ```js
   let defaultViewMatrix = [...];
   ```

   它是一个**列主序**的 4×4 视图矩阵，等价于「相机放在哪儿、朝哪儿看」。
   在 `index.html` 里可以看到预览图机位是「方位角 34°、俯角 14°、距离 ≈ 1.10 × 模型高度」，
   对应的矩阵就是这么算出来的。最省事的做法是直接改这两个常量并重算：

   ```python
   # 与站点保持一致的机位生成（列主序，直接粘贴进 gsplat.min.js）
   import numpy as np, math, json
   height = 16.36        # ← 换成你的模型高度
   az, el, dist = math.radians(34), math.radians(14), height * 1.10
   center = np.array([0.0, height * 0.5, 0.0])
   eye = center + np.array([math.cos(el)*math.sin(az), math.sin(el), math.cos(el)*math.cos(az)]) * dist
   f = (center - eye) / np.linalg.norm(center - eye)
   up = np.array([0.0, 1.0, 0.0])
   s = np.cross(f, up); s /= np.linalg.norm(s)
   u = np.cross(s, f)
   V = np.array([[s[0], s[1], s[2], -s @ eye],
                 [u[0], u[1], u[2], -u @ eye],
                 [-f[0], -f[1], -f[2], f @ eye],
                 [0, 0, 0, 1]])
   print(json.dumps([round(float(x), 4) for x in V.T.reshape(-1)]))
   ```

4. **如果模型上下颠倒或侧躺**：3DGS 重建出来的坐标系经常和 glTF 的 Y-up 不一致。
   在同一个文件里找 `defaultViewMatrix` 时，把 `up` 改成 `[0,0,1]` 重算一次，
   或者干脆在点云阶段对坐标做一次轴交换（`y, z = z, -y`）。

5. **点数与低配模式**。`assets/splat.html` 里的 `maxPoints: lowSpec ? 140000 : 0`
   就是低配档位的取点数。真实重建如果点数很多（比如 200 万），
   建议把高配档位也加个上限：

   ```js
   maxPoints: lowSpec ? 140000 : 600000
   ```

   这样高配档位只会请求前 60 万个点（靠 `Range` 请求），首屏不会因为 60 MB 的
   `.splat` 而卡住。**同时注意单文件 < 15 MB 的约束**：400 万个点就是 128 MB，
   必须先用 `ply2splat.py` 的第三个参数抽稀。

6. **清缓存再看**。浏览器对 `.splat` 会缓存，改完记得 `Ctrl + F5` 强刷，
   或者在地址栏后面加 `?v=2`。

---

## 五、常见问题

| 现象 | 原因 | 处理 |
|---|---|---|
| 点云缩成一小团 | `.ply` 的 log 尺度忘了 `exp()` | 用本文脚本，别自己手搓 |
| 一大片模糊的雾 | 照片重叠不足 / 有运动模糊 | 回去补拍，重叠拉到 70% 以上 |
| 模型上下颠倒 | 重建坐标系与 Y-up 不一致 | 见第四步第 4 点 |
| 页面白屏，控制台报 `Unable to load` | `.splat` 路径不对或服务器不支持 `Range` | 静态托管（GitHub Pages / Surge）都支持；本地 `python -m http.server` 不支持 `Range`，会退化成整包下载，属正常 |
| 手机很卡 | 点数过多 | 抽稀到 40 万以内，或打开页面上的「低配模式」 |
| 颜色发灰、发暗 | 只取了球谐直流分量，没做色调映射 | `.splat` 格式本身只存 RGB，属正常；可在导出前对 `rgb` 做一次 gamma 调整 |

---

## 六、当前 B 档数据是怎么生成的（对照用）

工作区里的 `tools/build_assets.py` 做了三件事，可以拿来和上面的真实流程对照：

1. `trimesh` + `numpy` 拼出古塔网格 → `assets/model.glb`
2. 按面积加权在网格表面采样 **400,000** 个点，取面颜色、沿法线方向压扁
   （切向 0.0435、法向 0.0161，切向/法向 ≈ 2.9），写出 `assets/scene.splat`（12.8 MB）
3. 自己写的软件光栅化渲染器输出 `assets/model-preview.jpg`，作为 3D 加载失败时的降级图

换成真实重建后，第 2 步的产物被替换掉，第 1、3 步可以保留，也可以一起换成
用重建模型的网格重新导出的 `.glb`。
