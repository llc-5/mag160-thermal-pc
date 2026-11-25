# MAG160 / MG160 热成像仪 · Windows 客户端

把手机端 `MagViewer-2.8-alpha3-beta` 的原厂测温引擎搬到了 Windows 上跑。
双击 `MAG160 热像仪.bat` 就能用，界面全中文深色，鼠标点哪儿读哪儿的温度。

> 原厂 .so 里的 ARM64 机器码由 Unicorn 在本地内存中仿真执行，
> **不刷机、不改固件、不往设备写任何东西**。程序对相机只发 7 条命令：
> 4 条读参数、快门开/关、开始/停止传图。

## 目录内容

| 文件 / 目录 | 说明 | 是否入库 |
|---|---|---|
| `MAG160 热像仪.bat` | 启动器，双击即用 | 是 |
| `start_app.bat` | 启动器（同上，等价） | 是 |
| `check_device.bat` | USB 诊断：列出本机 USB 设备，判断相机是否被识别 | 是 |
| `zadig.exe` | 驱动工具。相机在设备管理器里报错 28 时，用它装 WinUSB | 是 |
| `mag160-thermal-pc.bundle` | **完整源码仓库**（含 14 次提交历史、原厂二进制） | 是 |
| `MagViewerPC_v7/` | 主程序本体（412MB，含 Python 运行时） | 否（构建产物） |
| `MAG160热像仪-单文件.exe` | 便携版，单个 exe（157MB） | 否（超 GitHub 100MB 限制） |
| `CheckDevice/` | USB 诊断工具本体（63MB） | 否（构建产物） |

## 怎么用

1. 双击 `MAG160 热像仪.bat`
2. 程序自动找相机（VID `0x833C`）。找不到时：
   - 先双击 `check_device.bat` 看设备列表里有没有 `833C`
   - 有 `833C` 但程序仍连不上 → 用 `zadig.exe` 把它换成 **WinUSB** 驱动
3. 连上后状态栏显示「热像仪已连接 · 160×120 · 原厂自动测温」
4. 画面点一下 → 实时显示该点温度

## 从 bundle 拿源码

`mag160-thermal-pc.bundle` 是 `git bundle` 打包的裸仓库，把它当远程拉下来即可：

```bash
git clone mag160-thermal-pc.bundle mag160-pc
cd mag160-pc
git log --oneline        # 应有 14 条提交
```

## 怎么重建未入库的两项

```bash
cd mag160-pc
python release.py
```

`release.py` 一条命令完成：PyInstaller 打包 → `cfg_patch.py` 清除 PE 的 CFG 标志位
→ 清理日志 → 部署到本目录 → 重写 `start_app.bat`。

> **`cfg_patch.py` 这一步不能省。** PyInstaller 的 bootloader 在 PE 头里带
> `IMAGE_DLLCHARACTERISTICS_GUARD_CF`，Windows 因此给进程开启 Control Flow Guard；
> 而 Unicorn 是 JIT，运行时生成的机器码不在 CFG 合法目标表内，进程被 `__fastfail`
> 直接杀掉——表现为静默退出、退出码 `0xC0000409`、没有任何 traceback，
> 用 `faulthandler` 也只能看到一条无关的 first-chance 异常。

单文件版重建：

```bash
python -m PyInstaller --noconfirm --distpath dist_onefile --workpath build_onefile MagViewerPC_onefile.spec
python cfg_patch.py "dist_onefile/MAG160热像仪.exe"
```

## 已知限制

- **160×120 是硬件分辨率**。ESPCN 2× 超分和 Lanczos 只能改善观感，变不出真实细节。
- **镜头定焦、无电动对焦马达**。软件做不了物理对焦。手离开 30cm 以上变模糊是光学决定的，
  若镜头筒有滚花可以手动拧。
- **单点平场在参考温度附近最准**。探测器温度漂移大时，点一次「快门校正」重建平场。
- 帧率 8~12 fps。设备自报 15fps，整帧测温 48ms 是主要开销，已做出图/测温隔帧错开。

## 实测数据

| 项目 | 数值 |
|---|---|
| 原厂 `PushFrame` 出图 | ~31 ms |
| 原厂 `GetTemperaturemC` 整帧测温 | ~48 ms |
| ESPCN 2× 超分（CPU） | 2.2 ms |
| 平场前低频阴影 std | 182.75 counts（范围 660） |
| 平场后低频阴影 std | **14.50 counts（范围 80）** |
