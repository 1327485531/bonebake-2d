# bonebake-2d
Bake 3D skeletal animations into skinned 2D bones. Swap mesh/weights/textures to make multiple characters 。。with one rig, and export DragonBones
# 3D 骨骼动画 → 2D 骨骼动画（蒙皮 + 换素材做多角色）
<img width="1900" height="955" alt="chrome_jPYkalldGp" src="https://github.com/user-attachments/assets/18d11365-182e-4d89-a3ec-3704dbf288e8" />
<img width="1913" height="964" alt="chrome_VeQRxba4Iz" src="https://github.com/user-attachments/assets/aba5dec0-a084-4915-8d35-70eeec59e61d" />
<img width="1915" height="985" alt="3V8zjVESGh" src="https://github.com/user-attachments/assets/908745cf-f361-499d-b672-a43beb4a00d5" />

把 3D 的骨骼动画烘焙成 **2D 骨骼动画**：2D 网格做线性混合蒙皮，胳膊肘、膝盖会真正弯起来；
动画按骨长归一化，**换一套素材（网格 + 权重 + 贴图）就是另一个角色**，骨架和动画完全不动。
效果上等同于龙骨 / DragonBones 那种"骨架 + 蒙皮网格"。而且**能真的导出成 DragonBones 5.5 项目**：
`tools/export-dragonbones.mjs` 生成 JSON + 贴图集，官方 DragonBones 5.7 + Pixi 8 运行时直接解析播放，
逐帧骨骼矩阵/蒙皮顶点与 M2D 的误差 < 1.5px（300px/m 下约 5mm，`tools/verify-dragonbones.mjs` 实测）。

> UI 截图与 AI 生成图都不入库；仓库只保留代码、CC0 动画/模型、模板和文档。

素材来自 [Mesh2Motion](https://github.com/Mesh2Motion/mesh2motion-app) 的 CC0 模型与动画（MIT 代码 / CC0 素材）。
技术路线与源码分析见 **[docs/研究报告.md](docs/研究报告.md)**。

## 快速开始

```bash
# 依赖：Node.js（无 npm 依赖）。素材需要先克隆 mesh2motion-app：
git clone --depth 1 https://github.com/Mesh2Motion/mesh2motion-app.git

# 烘焙 2D 数据（侧面视角）
node tools/convert.mjs --view side --fps 30 \
  --anim mesh2motion-app/static/animations/human-base-animations.glb \
  --clips Idle_A,Walk,Jog,Sprint,Jump_air,Sword_Attack,Punch_Jab,Dance_Simple,Roll,Zombie_Walk,Pistol_Shoot,Crouch_Walk \
  --char male=mesh2motion-app/static/models-variation/human/male.glb \
  --char monster=mesh2motion-app/static/models-variation/human/monster.glb \
  --char swat_male=mesh2motion-app/static/models-variation/human/swat_male.glb \
  --char killer_4=mesh2motion-app/static/models-variation/human/killer_4.glb \
  --char doctor_m=mesh2motion-app/static/models-variation/human/doctor_m.glb \
  --out out/side

# 导出 DragonBones（每个角色一个 JSON，贴图路径相对 JSON）
node tools/export-dragonbones.mjs --data out/side --out out/side/dragonbones

# 用官方运行时校验（解析 + 逐帧骨骼求值 + 蒙皮顶点 + 贴图集）
node tools/verify-dragonbones.mjs --db out/side/dragonbones/male.json --data out/side --char male

# 看演示：必须用 HTTP 服务器，不能直接双击 index.html（浏览器会拦 file:// 的 module 和 fetch）
node tools/serve.mjs                 # 零依赖，推荐；端口被占用会自动换到 8900/8901…并打印 URL
# 或者： python3 tools/serve.py       # Windows 下别用 python -m http.server，.js 可能被识别成 text/plain
# Windows 可以直接双击根目录 start-server.bat；Linux/macOS 用 ./start-server.sh
# M2D 演示：  http://127.0.0.1:8899/web/index.html（以服务器打印的端口为准）
# 服务器自检：http://127.0.0.1:8899/__server
# 龙骨实测：  http://127.0.0.1:8899/web/dragonbones.html
```

`tools/bake-all.sh` 会把正/侧两个视角的转换和龙骨导出一起做完；
`tools/verify-all-dragonbones.sh side` 逐个角色跑官方运行时校验。

演示页可以切角色 / 切动画 / 正侧面视角、显示骨架与网格叠加、开关深度排序；
底部**角色墙**用同一套动画的同一帧同时驱动 4 个不同角色 —— 这就是"一个动画做很多角色"。

两个特别值得点的角色：

- **naked** —— Mesh2Motion 的裸模型，**一个蒙皮权重都没有**。转换器用「最近骨段反距离加权 +
  邻接平滑」现算权重，0.5 秒绑好 7399 个顶点，胳膊腿照样弯。
- **zombie** —— 只有 **42 根骨**（比人形少了中/无名/小指共 24 节）。转换器按「语义槽位」
  把它和人形 66 骨对上，链长不等时按比例重采样，同一套动画照样能驱动。

## 目录

| 路径 | 说明 |
|---|---|
| `docs/研究报告.md` | **技术路线 + Mesh2Motion 源码分析 + 踩坑记录 + 龙骨导出与实测** |
| `docs/AI皮肤接入.md` | **GPT Image 2 接入**：提示词、平面 UV、龙骨导出、局限与密钥安全 |
| `tools/glb.mjs` | 零依赖 GLB/glTF 读取器 |
| `tools/inspect.mjs` | 查看 GLB 的骨骼 / 网格 / 动画结构 |
| `tools/convert.mjs` | 转换器：3D 骨架+动画 → 2D 骨架 + 2D 蒙皮网格 + 权重 + 深度 |
| `tools/autoskin.mjs` | **自动蒙皮**：给没有权重的裸模型现算权重（0.5s 绑 7399 顶点 / 66 骨） |
| `tools/bonemap.mjs` | **骨骼语义映射**：骨骼数量/命名不同的骨架也能吃同一套动画 |
| `tools/render-ref.mjs` | 独立软件光栅化参考渲染器（z-buffer）；支持 `bind` 渲染绑定姿态，给 AI 皮肤当参考图 |
| `tools/ai-skin.mjs` | **AI 皮肤（整身平面版）**：gpt-image-2 重绘 + 去背景/裁边 + 平面投影 UV → `characters-ai.json` |
| `tools/ai-skin-parts.mjs` | **AI 皮肤（精细分块版）**：6 块独立生成/裁块 + 图集打包 + 单 part 网格；支持 `--pose` 纹理姿态、`--style chibi`、`--head-scale` 大头 |
| `tools/fix-shoulder-weights.mjs` | **修肩膀被右臂扯动**：torso 块上臂权重按离肩距离衰减到 spine_03；已接入 bake-all |
| `tools/generate-character-sets.mjs` | **批量主题角色**：幻想法师/蒸汽朋克/水墨仙侠/废土/森林精灵，5×6 部位；自动构建 |
| `tools/make-studio-data.mjs` | 生成 `web/studio-data.js`（base/pose/ref 内嵌，供 file:// 用） |
| `tools/patch-shoulder-socket.mjs` | **补侧视躯干肩窝黑洞**：跑步手臂前摆时露出的黑色肩窝改成机械内衬+金属圈；可 `--rebuild` |
| `tools/fix-shoulder-panel.mjs` | **修右后肩肩皮前移/变形**：把右后肩 panel 的 clavicle_r（v5）/ clavicle_r+upperarm_r（Q版 `--both`）冻结到 spine_03 |
| `tools/pose2d.mjs` | **纹理姿态求值**：按 clip/frame 做 2D LBS，给侧视手臂等退化投影的块求 UV |
| `tools/make-templates.mjs` | **手绘模板包**：6 块品红轮廓 guide + 透明 canvas + 3x2 可导入 sheet |
| `tools/import-part-drawing.mjs` | **手绘导入**：吃 3x2 sheet 或 6 块部件图，剥 guide、抠图并直接构建角色 |
| `tools/import-ui-server.mjs` | **可视化导入器服务**：零依赖本地 HTTP，上传/预览/构建（默认 `127.0.0.1:8898`） |
| `web/import.html`、`web/import.js` | 可视化导入器前端：3x2 sheet / 6 块上传、预览 6 块、构建结果 |
| `tools/ai-image.mjs`、`tools/aruhub.mjs` | 分块/图集/PNG 工具与 AruHub 图片接口封装 |
| `tools/inspect-skin.mjs` | 检查角色贴图是语义 UV 图集还是调色板（UV 占用率 / 颜色数） |
| `tools/bake-ai-atlas.mjs` | **原 UV 烘焙**：把 AI 图烤进角色原有 UV 图集，网格/UV/权重不变 |
| `tools/export-dragonbones.mjs` | **导出 DragonBones 5.5**：骨架 / 蒙皮网格 / 权重 / 抽帧动画 / 贴图集 |
| `tools/verify-dragonbones.mjs` | **用官方运行时校验导出**：逐骨逐帧对比全局矩阵、逐顶点对比蒙皮结果 |
| `web/runtime.js` | M2D 运行时：WebGL2 + 2D 线性混合蒙皮 + 逐帧深度排序 |
| `web/index.html`、`web/demo.js` | M2D 演示页 |
| `web/import.html`、`web/import.js` | 可视化手绘导入器前端（上传/预览/构建） |
| `web/studio.html` | **皮肤工作室**：AI 生成 + 导入/构建两个 tab，生成 6 块可一键送到导入 |
| `web/studio-data.js` | 单文件模式内置数据：base/pose/12 张 ref（约 1.85MB，脚本生成） |
| `web/generate.html`、`web/generate.js` | **AI 生成器页面**：填 API 地址 + Key + 角色描写，选风格/视角/部位直接调 OpenAI 兼容接口 |
| `web/offline-build.js` | 纯浏览器离线构建：拆块/UV/图集/角色 JSON（无 Node） |
| `web/dragonbones.html`、`web/dragonbones-demo.js` | **龙骨实测页**：只用官方 DragonBones + Pixi 运行时渲染导出的 JSON |
| `tools/vendor/`、`web/vendor/` | 官方 DragonBones / Pixi 运行时（MIT，仅校验和演示用，见 `tools/vendor/README.md`） |
| `out/<view>/` | 烘焙产物（不入库）：`characters.json`、`anim.json`、`tex/*.png` |
| `out/<view>/dragonbones/` | 龙骨导出产物（不入库）：每角色一个 `<char>.json` |
| `templates/side`、`templates/front` | **手绘模板包（入库）**：6 块 `template-*.png` / `canvas-*.png` / `ref-*.png` + 3x2 `sheet.png` + 离线 `base-male.json` / `pose-walk0.json` + README |
| `out/<view>/ai/` | AI 皮肤产物（不入库）：参考图、原图、抠图、纹理 |
| `out/<view>/characters-ai.json` | AI 皮肤角色（不入库，重新生成即可） |
| `out/<view>/ai/parts/` | 精细版的每块参考图 / 原图 / 纹理（不入库） |
| `mesh2motion-app/` | 上游参考仓库（不入库） |

## 数据格式

每根骨头每帧只存 4 个数：`平移比x`、`平移比y`、`局部旋转`、`深度`。
平移按骨长归一化 → 跨角色通用；深度从 3D 一起烘焙下来 → 运行时用于三角面排序，
解决"胳膊在身前还是身后"以及"躯干正反面"的遮挡。详见研究报告第三节。

## 皮肤工作室（生成 + 导入，无服务）

打开 [`web/studio.html`](web/studio.html)（直接双击 `file://` 也行）：

- **AI 生成**：填 Key + 角色描写，生成单块/全部 6 块；
- **导入 / 构建**：上传 sheet/6 块，预览后构建并下载；
- 生成 6 块后点“用这 6 块去导入/构建”，自动填好槽位；
- 导入/构建不需要任何服务：base/pose/ref 已内嵌在 `web/studio-data.js`；
- 生成只需要联网，直接连你填的 OpenAI 兼容 API，不启动本地服务；
- 改了模板/base 后运行 `node tools/make-studio-data.mjs` 重新生成内嵌数据。

详细说明见 docs 7.18。

## AI 皮肤（GPT Image 2，已实测）
骨架/权重/动画不变，用 gpt-image-2 按绑定姿态重绘，再贴回同一套网格。现在有这几版：

- **`ai_ronin`（单张平面投影）**：`tools/ai-skin.mjs` 生成一张整身图，按整个人物包围盒做平面 UV。
  一张图、两次调用，最省事。
- **`ai_ronin_hd`（精细分块）**：`tools/ai-skin-parts.mjs` 按主导骨把头 / 躯干 / 左右臂 / 左右腿
  分成 6 块，正视图每块单独生成（脸、夹克、机械臂、战术裤各有细节），再打包成一张图集。
  网格复制块边界顶点、仍保持**单 part**，所以 M2D 的逐三角深度排序不受影响。
- **`ai_ronin_v5`（修正版，推荐看侧视）**：侧视绑定姿态是 T-pose，手臂正对镜头，屏幕投影会把整条手臂
  压成肩部一个圆——旧版拿这个 bbox 做平面 UV，手臂纹理必然糊。v5 改用 **Walk 第 0 帧的纹理姿态**求
  `arm_l` / `arm_r` 的 UV，躯干 / 腿也从**单独部件参考图**重新生成（不再从整身图裁块，避免把手/手臂裁进
  躯干和腿的贴图）。近侧机械臂和远侧深色夹克袖子都显示，远侧由深度排序挡在躯干后面。
  `tools/pose2d.mjs` + `ai-skin-parts.mjs --pose` 实现，详见 docs 7.8。
- **`ai_chibi`（Q 版卡通）**：`--style chibi` 生成大头、大眼睛、粗描边的卡通分块贴图，
  `--head-scale 2` 只把 `head` 组顶点放大 2 倍（骨骼/权重/动画不变）；侧视两条手臂都显示，
  远侧米白袖子单独生成、由深度排序挡在躯干后面。正视图是大头 + 琥珀大眼 + 薄荷绿刺猬头 + 机械右臂，详见 docs 7.9。
- **`ai_police`（原 UV 烘焙）**：`tools/bake-ai-atlas.mjs` 把 AI 图按三角形的绑定姿态屏幕位置，
  直接烘焙进角色**原本的 512 UV 图集**（输出 1024），网格 / UV / 权重完全不动，没有分块接缝。
  适合 `police_male`、`female_8`、`monster` 这类本来就有语义 UV 图的角色；
  `male` / `naked` 的 UV 只是两块调色板，不适合这条路（见 `tools/inspect-skin.mjs`）。
```bash
# Key 只放环境变量，绝不入库
export ARUHUB_API_KEY='...'

# 1) 先有整身侧视参考（精细版侧视裁块要用）
node tools/ai-skin.mjs --char male --name ai_ronin --views side,front

# 2) 精细分块版：正视图 6 块逐块生成，侧视图从整身图裁块
node tools/ai-skin-parts.mjs --char male --name ai_ronin_hd --views side,front

# 刷新后打开：
# http://127.0.0.1:8899/web/index.html?view=front&char=ai_ronin_hd&clip=Walk&frame=0&play=0&bones=0
```

修正版 `ai_ronin_v5`（侧视两条手臂用纹理姿态；独立部件图先按 docs 7.8 生成/抠好，放进 `out/<view>/ai/parts/`）：

```bash
# 侧视：arm_l / arm_r 都用 Walk 0 的纹理姿态，其余块仍用 bind
# （远侧深色袖子也要先单独生成 ai_ronin_v5-arm_l-side.png，见 docs 7.8）
node tools/ai-skin-parts.mjs --char male --name ai_ronin_v5 --views side --source custom \
  --pose Walk --pose-frame 0 --pose-groups arm_l,arm_r

# 正视图：全部 bind（可复用 v2 的正视分块图）
node tools/ai-skin-parts.mjs --char male --name ai_ronin_v5 --views front --source custom

# http://127.0.0.1:8899/web/index.html?view=side&char=ai_ronin_v5&clip=Walk&frame=0&play=0&bones=0
```

Q 版卡通角色 `ai_chibi`（大头 + 卡通贴图；侧视 6 张、正视图 6 张）：

```bash
# 侧视：Q 版贴图 + 两条手臂都用 Walk 0 纹理姿态 + 2x 大头
node tools/ai-skin-parts.mjs --char male --name ai_chibi --views side \
  --source generated --style chibi --pose Walk --pose-frame 0 --pose-groups arm_l,arm_r \
  --head-scale 2

# 正视图
node tools/ai-skin-parts.mjs --char male --name ai_chibi --views front \
  --source generated --style chibi --head-scale 2

# http://127.0.0.1:8899/web/index.html?view=front&char=ai_chibi&clip=Walk&frame=0&play=0&bones=0
```

手绘模板 / 导入（模板已入库在 `templates/`，不需要调 API）：

```bash
# 打开 templates/side/sheet.png 或 template-<group>.png 描线，然后：
node tools/import-part-drawing.mjs --view side --sheet templates/side/sheet.png --name mychar

# 也可以 6 张分块图
node tools/import-part-drawing.mjs --view side --dir mydrawings --prefix mychar --name mychar

# 正视图 / Q 版大头
node tools/import-part-drawing.mjs --view front --sheet templates/front/sheet.png --name mychar --head-scale 2
```

模板是 6 块 **品红轮廓 guide**（1024×1024）+ 全透明 canvas + 一张 3×2 sheet；
导入工具会自动剥掉品红、抠图，并直接构建角色。完整说明见 [`templates/README.md`](templates/README.md) 和 docs 7.11。

可视化导入器（上传 sheet / 6 块 → 预览 → 构建）：

```bash
# 服务模式：预览/构建直接写 out/
node tools/import-ui-server.mjs
# 手绘皮肤导入器: http://127.0.0.1:8898/
# 演示页工具栏也有“手绘导入 ↗”按钮

# 离线模式：不用 Node 服务，任意静态服务器打开页面
python3 -m http.server 8898
# http://127.0.0.1:8898/web/import.html
# 构建后下载 atlas.png + characters-ai.json，手动放进 out/<view>/
```

服务模式只绑定 `127.0.0.1`，不调 AI 接口；默认不允许覆盖同名角色。
离线模式在浏览器里完成抠图/图集/JSON，不需要后端（`file://` 会被 CORS 拦住，要用静态服务器）。
完整说明见 [`templates/README.md`](templates/README.md)、[`web/import.html`](web/import.html) 和 docs 7.11 / 7.12。

**AI 生成器页面**（只填 Key + 角色描写，其余提示词内置）：

```text
http://127.0.0.1:8898/generate.html               # Node 服务
http://<host>:8000/.../web/generate.html          # 任意静态服务器
```

- Q版 / 写实是两套独立风格提示词；6 个部位各有独立 PART_PROMPT；
- 骨骼/姿势由 `templates/<view>/ref-<group>.png` 锁定，不在文字里；
- API 地址由用户填，兼容 OpenAI `/images/edits`；不是 Google；
- 单个部位或“全部 6 块（依次）”；生成后下载透明图，去手绘导入页构建。
详见 docs 7.13 和 [`web/generate.html`](web/generate.html)。

原 UV 烘焙版（`ai_police`）：

```bash
# 先看看这个角色的贴图是语义图集还是调色板
node tools/inspect-skin.mjs --data out/side --char police_male

# 给 police_male 生成整身参考图（不写入演示角色，只留图/元数据）
node tools/ai-skin.mjs --char police_male --name ai_police_src --views side,front --no-char

# 烘焙进 police_male 的原始 UV 图集
node tools/bake-ai-atlas.mjs --data out/front --char police_male --ai ai_police_src --name ai_police
node tools/bake-ai-atlas.mjs --data out/side  --char police_male --ai ai_police_src --name ai_police

# http://127.0.0.1:8899/web/index.html?view=front&char=ai_police&clip=Walk&frame=0&play=0&bones=0
```

演示页会自动发现 `characters-ai.json`（角色名带 `· AI`），也能继续导出 Cocos 3.8.4 包；
`ai_ronin_hd` 的 2014x2008 图集在官方 DragonBones 5.7 + Pixi 8 里同样通过了逐帧校验。
> 上一版 `ai_ronin_v2` 把侧视近侧机械臂单独生成、远侧手臂隐藏，但手臂 UV 仍按绑定姿态求，
> 所以手指会跑到肘部；腰胯处还会有从整身图裁块带进来的“幽灵手”。v5 修掉了这两个问题：
> 手臂 UV 换到 Walk 0 纹理姿态，躯干/腿改成独立部件图。正视图脚趾的“碎裂”经纯色测试确认是
> 高模脚部网格自交/破洞，不是贴图问题；2×3 部件 sheet 直贴的实验也做过，部件姿态对不上，详见 docs 7.5–7.8。
>
> 接线细节、提示词、分块/图集原理和局限见 **[docs/AI皮肤接入.md](docs/AI皮肤接入.md)**；
> 生成图在哪、是一块一张还是整张、怎么拿 `ref-*` 图当模板自己画或用图生图换角色，见 **docs 7.10**。
注意：AI 只换"皮"，网格和蒙皮权重仍是原角色（普通模型来自 GLB，`naked` 由 `tools/autoskin.mjs` 现算）。

> **肩膀被右臂扯动**：源模型的上臂权重覆盖到了 torso 肩部；已用
> [`tools/fix-shoulder-weights.mjs`](tools/fix-shoulder-weights.mjs) 对 torso 块做“离肩距离衰减”，
> 削掉的权重转给 `spine_03`，手臂姿势/动画不变。`bake-all.sh` 已自动接入：
> `node tools/fix-shoulder-weights.mjs --views side,front`。原理与 12 个动画的验证见 **docs 7.14**。
> 跑步时侧视 `ai_ronin_v5` 肩窝还有一块贴图黑洞（torso 生成时没有手臂），已用
> [`tools/patch-shoulder-socket.mjs`](tools/patch-shoulder-socket.mjs) 补成机械肩窝；
> 预览见 docs 7.15。跑步时右后肩肩皮向前移动/变形是另一层：已用
> [`tools/fix-shoulder-panel.mjs`](tools/fix-shoulder-panel.mjs) 把 panel 的 `clavicle_r` 权重冻结到 `spine_03`，
> 胸口/脖子不动；预览见 docs 7.16。Q版 `ai_chibi` 的右肩衣服扭曲用 `--both`（锁骨+大臂）冻结，
> 见 docs 7.17。

## 导出 DragonBones（Cocos Creator 3.8.4 / Pro）

演示页工具栏的 **“导出 Cocos 3.8.4 ⤓”** 会从当前 `characters.json + anim.json` 现场生成
**Cocos Creator 3.8.4 可直接导入的包**，并打成一个 ZIP：

```text
<char>-cocos384/
  README.md
  assets/dragonbones/<char>/
    <char>_ske.json
    <char>_tex.json
    <char>_tex.png
  assets/scripts/DragonBonesPlayer.ts
```

Cocos Creator 3.8.x 的 `dragonBones.ArmatureDisplay` 支持 DragonBones 5.6.3 及以下，这份是 5.5。
把 `assets/dragonbones` 和 `assets/scripts` 拷进 Cocos 项目的 `assets/`，编辑器会按 `_ske` / `_tex` 后缀
自动识别成 **DragonBonesAsset** / **DragonBonesAtlasAsset** / 纹理。

用法（详见 [docs/cocos-creator-3.8.4.md](docs/cocos-creator-3.8.4.md)）：

1. Canvas 下新建空节点，添加 **DragonBones → ArmatureDisplay**；
2. `Dragon Asset` 选 `<char>_ske`，`Dragon Atlas Asset` 选 `<char>_tex`；
3. `Armature Name` 填 `<char>`，`Animation` 选 `Idle_A`（或 Walk / Jog / …），`Play Times` 填 `0`。

数据按 **300px = 1 米** 放大并居中，`male` 约 549px 高；2D Canvas 里太大就把节点 Scale 调到 0.5。
正视导出会生成 `<char>_front` 文件夹和文件前缀，armature 仍是 `<char>`。

**Pro / 其他引擎**：`_ske/_tex/png` 是标准 DragonBones 5.5 数据，Pixi / Phaser 等运行时可直接用。
Cocos 包里不带 Pro 专属的 `edges`；如果要在 DragonBones Pro 里编辑，Pro 的网格导入还需要额外的
slot/display/SubTexture 同名、`edges`、`width/height` 处理，见 [docs/研究报告.md](docs/研究报告.md) 第 6 节。
`web/dragonbones.html` 用官方 DragonBones 5.7 + Pixi 8 运行时逐帧对照。

### 坐标 / 蒙皮约定

- **坐标系**：M2D 是 y 向上，龙骨是 y 向下。导出时统一 `(x, y) → (x, -y)`、`rotation → -rotation`，再乘 300。
- **蒙皮空间**：`vertices` 存绑定姿态的骨架空间坐标，`slotPose` 是单位矩阵，
  `bonePose[骨]` 是该骨的绑定姿态世界矩阵。官方解析会算 `bonePose⁻¹ · vertex`，
  播放时再乘当前骨骼矩阵，正好抵消成标准 LBS。
- 龙骨骨骼时间轴的 `x/y/rotate` 是相对绑定姿态的**增量**，导出时做了差。

### 关键帧不再是“每帧全满”

源 `anim.json` 是 30fps 逐帧烘焙的；如果原样写，65 根骨 × 95 帧 × 12 个动画会把时间轴塞满。
导出时在 **0.25px / 0.05°** 容差内做折线抽帧（RDP，旋转先展开避免 ±180° 跳变），
实测 `male` 从 81018 个关键帧降到 25651 个（约 31%），逐帧渲染误差 < 1.1px。
文件从 ~4.5MB 降到 ~1.8MB；明显不动的骨不再生成时间轴。

`web/dragonbones.html` 就是同一套数据的官方运行时渲染，可以和 M2D 页逐帧对照
（两页都支持 `?view=side&char=zombie&clip=Sword_Attack&frame=20&play=0` 参数）。

**已知取舍**：每个角色导出成**一个蒙皮网格槽**，三角形顺序固定，没有 M2D 运行时那种逐三角深度排序，
肢体与躯干交错时偶尔会有遮挡差异；下一步是按骨骼分桶成多个 slot，再用龙骨 5.5 的 slot zIndex
时间轴把每帧深度写进去。

## 许可

- 代码：MIT，见 [LICENSE](LICENSE)
- 动画/模型/骨架：CC0 1.0（Mesh2Motion）
- 字体：SIL OFL 1.1；DragonBones/Pixi：MIT
- **仓库不包含 AI 生成图片**；AI 生成脚本是 MIT 代码，生成物由用户自行处理

完整分层说明见 [ASSETS-LICENSE.md](ASSETS-LICENSE.md) 与 [THIRD_PARTY_LICENSES.md](THIRD_PARTY_LICENSES.md)。

## 发布/打包

见 [PACKAGING.md](PACKAGING.md)：`out/`、`mesh2motion-app/`、`shots/` 不入库；
发 GitHub 前先确认样例动画/CMU/AI 贴图的许可边界。
