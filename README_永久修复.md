# APCEMM v1.2.1 永久可复现性修复副本

本目录是独立的永久修复副本，不修改原来的
`/Users/maoxinyi/Desktop/MD/dannie/APCEMM/APCEMM v1.2.1`，
也不影响临时修复副本 `APCEMM修复- 临时`。

修复日期：2026-09-29

---

## 1. 修复的竞态与问题

### 1.1 `Grid_Aerosol::Moment(UInt n)` 数据竞争

文件：`Code.v05-00/src/AIM/Aerosol.cpp`

原实现：

```cpp
#pragma omp parallel for ... schedule(dynamic, 1)
for (iBin = 0; iBin < nBin; iBin++)
    for (jNy = 0; jNy < Ny; jNy++)
        for (iNx = 0; iNx < Nx; iNx++)
            moment[jNy][iNx] += ...;   // 多线程写同一元素
```

修复：

- 并行维度改为空间行 `jNy`；
- 每个线程只写自己负责的 `moment[jNy][iNx]`；
- 每个格点的粒径 bin 累加在本地 `acc` 中按固定顺序完成。

这既消除了数据竞争，也保证了与单线程相同的求和顺序。

### 1.2 `localND` 共享变量竞争

文件：`Code.v05-00/src/Core/LAGRIDPlumeModel.cpp`

原实现在两个单位换算循环中把 `localND` 声明在并行循环外部，导致多个线程
同时写同一个变量，并把别的行的高度换算系数用于当前行。

修复：

- 把 `localND` 移入循环体内部，声明为 `const double`；
- 两处换算循环（`/ localND` 与 `* localND`）都已修复。

该竞态会影响 H₂O、contrail 示踪物和所有冰晶 bin 的每一步换算，是完整算例
结果严重发散的主要原因之一。

### 1.3 归约求和调度

文件：`Code.v05-00/src/AIM/Aerosol.cpp`

以下四个归约从 `schedule(dynamic, 1)` 改为 `schedule(static)`：

- `TotalNumber_sum`
- `TotalIceMass_sum`
- `Moment(UInt n, const Vector_1D& PDF)`
- `Moment(UInt n, UInt jNy, UInt iNx)`

这样各线程的分块固定，减少浮点求和顺序的不确定性。

### 1.4 `CreateREADME()` 空指针崩溃

文件：`Code.v05-00/src/Core/Main.cpp`

`SLURM_JOB_USER` / `SLURM_NODELIST` 为空时改为写 `unknown`，不再需要
手动设置假环境变量。

---

## 2. 编译

```bash
cd "/Users/maoxinyi/Desktop/MD/dannie/APCEMM/APCEMM修复- 永久"
mkdir -p rundirs/build-permfix
cd rundirs/build-permfix

"/Users/maoxinyi/Desktop/MD/dannie/APCEMM/.tools/cmake-3.28.6-macos-universal/CMake.app/Contents/bin/cmake" \
  ../../Code.v05-00 \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_TOOLCHAIN_FILE= \
  -DCMAKE_PREFIX_PATH="/Users/maoxinyi/Desktop/MD/dannie/APCEMM/APCEMM修复- 永久/rundirs/cmake-pkgs;/opt/homebrew/opt/boost@1.85;/opt/homebrew" \
  -DBOOST_ROOT=/opt/homebrew/opt/boost@1.85 \
  -DOPENSSL_ROOT_DIR=/opt/homebrew/opt/openssl@1.1 \
  -DCMAKE_C_FLAGS="-I/opt/homebrew/opt/libomp/include" \
  -DCMAKE_CXX_FLAGS="-I/opt/homebrew/opt/libomp/include" \
  -DCMAKE_EXE_LINKER_FLAGS="-L/opt/homebrew/opt/openssl@1.1/lib" \
  -DOpenMP_CXX_FLAGS="-Xpreprocessor -fopenmp" \
  -DOpenMP_C_FLAGS="-Xpreprocessor -fopenmp" \
  -DOpenMP_CXX_LIB_NAMES="omp" \
  -DOpenMP_C_LIB_NAMES="omp" \
  -DOpenMP_omp_LIBRARY=/opt/homebrew/opt/libomp/lib/libomp.dylib

"/Users/maoxinyi/Desktop/MD/dannie/APCEMM/.tools/cmake-3.28.6-macos-universal/CMake.app/Contents/bin/cmake" --build . -- -j8
```

可执行文件：`rundirs/build-permfix/APCEMM`

运行示例：

```bash
cd "/Users/maoxinyi/Desktop/MD/dannie/APCEMM/APCEMM修复- 永久/examples/issl_rhi140"
OMP_NUM_THREADS=8 ../../rundirs/build-permfix/APCEMM input.yaml > APCEMM_run_log.txt 2>&1
```

---

## 3. 验证结果

### 3.1 快速重复测试（8 线程）

统一使用输入文件中的 8 线程配置，连续运行 10 次：

- `Initial Ice Mass`：10/10 全部为 `0.0521689`；
- 不再需要设置 SLURM 环境变量；
- `README` 正常生成。

### 3.2 完整算例重复测试（8 线程）

两次完整运行，均使用 8 线程，只改输出目录：

| 指标 | 运行 A | 运行 B |
|---|---:|---:|
| Initial Ice Mass | 0.0521689 | 0.0521689 |
| Peak Ice Mass | 38.7735 | 38.7735 |
| Peak step | 228 | 228 |
| Last step | 720 | 720 |
| Final particles | 9.78265e10 | 9.78265e10 |
| Final ice mass | 15.0376 | 15.0376 |
| Runtime | 807301 ms | 805888 ms |
| netCDF frames | 73 | 73 |
| status | Incomplete | Incomplete |

逐帧对比 73 个 netCDF 文件，以下变量的最大绝对差值全部为 **0**：

- `Ice Mass`
- `Number Ice Particles`
- `IWC`
- `Temperature`
- `Ice aerosol particle number`
- `RHi`
- `Extinction`
- `Effective radius`

### 3.3 与临时单线程基线的一致性

永久修复版（8 线程）与临时修复版（强制单线程）的完整算例也做了逐帧对比：

- 73/73 帧全部对齐；
- 上述所有变量最大绝对差值全部为 **0**。

这证明：

1. 永久修复没有改变物理结果；
2. 多线程结果与单线程基准完全一致；
3. 运行速度恢复到多线程水平（本机完整算例约 13.5 分钟，
   明显快于单线程的约 27 分钟）。

---

## 4. 后续注意事项

1. 本副本只审计并修复了主循环实际调用的并行路径。
2. `Grid_Aerosol::Coagulate()` 当前未被 v1.2.1 主循环调用；将来启用
   凝并或加入化学模块时，需要重新审计其并行路径。
3. 温度扰动 `Meteorology::updateTempPerturb()` 在并行循环中调用
   `fRand()`；当前基线关闭温度扰动，因此不受影响。若后续启用该功能，
   需要改为线程安全的随机数生成方式。
