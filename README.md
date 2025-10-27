# MuJoCo（个人分支）：Tracy 性能分析集成与示例

本分支基于 [google-deepmind/mujoco](https://github.com/google-deepmind/mujoco)，加入了 Tracy 客户端并在关键路径进行了插桩，便于在实际仿真中进行端到端性能分析与定位热点。

## 主要改动
- 集成 Tracy C 客户端并启用 `TRACY_ENABLE` + `TRACY_ON_DEMAND`（仅在 Viewer 连接时采集）。
- 在引擎前向流程 `engine_forward.c` 内添加 CPU 区段（函数入口与关键计算片段）。
- 设置线程名：主线程、rollout 线程、线程池 worker 线程，便于在 Viewer 中区分。
- 在 `testspeed` 示例中修复参数校验逻辑，允许传入第 6 个参数以启用内部线程池。
- 在每步 `mj_step` 后添加 `FrameMark`，方便在帧视图中定位与筛选区段。

## 快速开始（Windows 示例）
- 构建：
  - `cmake -S . -B build`
  - `cmake --build build --config Release -j 8`
- 单独构建示例：
  - `cmake --build build --config Release --target testspeed -j 8`

> 说明：已在 CMake 接入 Tracy 的头文件与客户端源码；无需额外配置即可编译，并默认按需采集（Viewer 连接时）。

## Tracy 子模块与构建
- 本仓库包含 Tracy 作为 Git 子模块（`third_party/tracy`）。克隆方式建议：
  - 直接递归克隆：`git clone --recurse-submodules <repo-url>`
  - 或在已有克隆中初始化：`git submodule update --init --recursive`
- 构建 Tracy Viewer（Windows）：
  - 配置：`cmake -S third_party/tracy/profiler -B third_party/tracy/build/profiler`
  - 构建：`cmake --build third_party/tracy/build/profiler --config Release -j 8`
  - 生成路径：`third_party\tracy\build\profiler\Release\tracy-profiler.exe`
- 注意：`third_party/tracy/build/` 已被 `.gitignore` 排除，不会进入仓库；请在本地按需构建。

## 运行示例
- 不启用内部线程池：
  - `build\bin\Release\testspeed.exe model\humanoid\humanoid.xml 500 2 0.01 0.1`
- 启用内部线程池（第 6 个参数为线程池大小）：
  - `build\bin\Release\testspeed.exe model\humanoid\humanoid.xml 500 2 0.01 0.1 2`

`testspeed` 会输出每线程统计与内部分析器数据；在 Viewer 连接情况下，也会同步采集并展示区段与线程信息。

## Tracy Viewer
- 使用与客户端版本一致的 Viewer（已从仓库源码构建）：
  - 路径：`third_party\tracy\build\profiler\Release\tracy-profiler.exe`
- 推荐流程：
  - 先启动 Viewer 并点击 “Connect”。
  - 再运行 `testspeed.exe`，可在 Viewer 中看到：
    - 线程名：`main`、`rollout-<id>`、`worker-<id>`。
    - 帧标记：每步 `mj_step` 的 `FrameMark`。
    - CPU 区段：`mj_fwdPosition`、`mj_fwdVelocity`、`mj_fwdActuation`、`mj_fwdConstraint`、`mj_forwardSkip` 等。

## 版本兼容与锁定
- 本分支未在代码中指定固定的 Tracy 版本号；Tracy 客户端来自子模块当前提交，并通过 `TracyClient.cpp` 编译嵌入。
- 协议兼容性要求：Viewer 与客户端必须版本匹配。若出现 “incompatible protocol version”，说明两者不匹配。
- 推荐做法：
  - 保持子模块提交固定（仓库已记录子模块的 commit），并从该提交构建 Viewer。
  - 更新 Tracy 时：先更新子模块到目标提交/标签（例如 `git -C third_party/tracy checkout v0.12.4` 并提交 gitlink），随后重建 Viewer 与本项目。
  - 若仅更新了 Viewer 或仅更新了子模块客户端，可能产生协议不兼容；务必两者同步。

## 插桩约定与维护
- 头文件：在需要的 C 源文件顶部包含 `#include "tracy/TracyC.h"`（受 `TRACY_ENABLE` 控制）。
- 区段模板：
  - 进入区段：`TracyCZoneN(zName, "section_name", 1);`
  - 退出区段：`TracyCZoneEnd(zName);`（注意所有早返回路径都要关闭）
- 注意避免手动写 `TracyCZoneCtx zName;`（宏内部已声明，重复会导致“重定义”错误）。
- 无需调用 `___tracy_init_thread()`（C++ 客户端内部符号）；线程命名用 `TracyCSetThreadName` 即可。

## 与上游差异（摘要）
- `src/engine/engine_forward.c`：添加区段宏以定位热点。
- `src/thread/thread_pool.cc`：为 worker 线程设置 Tracy 线程名。
- `sample/testspeed.cc`：修复参数检查并添加主线程与 rollout 线程命名、帧标记。
- CMake：接入 Tracy 头与客户端源码，启用按需采集。

## License
- 本仓库为上游 MuJoCo 的衍生分支，保留其原有开源许可证与致谢信息；具体参见 `LICENSE` 与上游文档。

## 致谢
- 感谢 [Google DeepMind](https://www.deepmind.com/) 维护的开源项目 [mujoco](https://github.com/google-deepmind/mujoco)。
