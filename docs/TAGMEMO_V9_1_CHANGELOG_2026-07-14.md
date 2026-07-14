# TagMemo V9.1 更新日志：枢纽免疫、软非回溯与有限时域场

> 日期：2026-07-14  
> 版本标识：对外仍使用 `v9`，内部算法版本升级为 `v9.1`  
> 对照方式：直接复跑既有 350 组 V8.3/V9 数据，与历史 V9 结果比较  
> 回退路径：`v8_3` 资产和执行路径保持不变

## 1. 发布策略

本次不增加第三个版本标识，也不修改 LightMemo A/B 协议。

现有 `v9` 原位升级为 V9.1。历史 350 组 V9 结果作为旧基线，新运行结果由新的 artifact signature 和 `algorithmVersion: v9.1` 区分。

本次只改变传播侧，不改变 Potential Field 候选读出公式，以缩短验证路径并避免同时修改传播与重排造成归因混乱。

## 2. 入流枢纽校正

V9 已通过行归一化约束单个源节点的总出流，但通用标签仍可能从大量源节点持续吸积质量。

V9.1 首先构造未归一化传导率：

\[
g_{ij}
=
\log(1+\lambda W_{ij})
\cdot
G_{\mathrm{wormhole}}(i,j)
\]

统计目标节点总入流：

\[
s_j^{in}
=
\sum_i g_{ij}
\]

以全图正入流中位数为基准，对目标节点实施夹逼后的幂律校正：

\[
h_j
=
\operatorname{clip}
\left(
\left(
\frac{s_j^{in}}
{\operatorname{median}(s^{in})+\epsilon}
\right)^{-\eta},
h_{\min},
h_{\max}
\right)
\]

\[
\tilde g_{ij}=g_{ij}h_j
\]

最后仍统一归一化到固定节点出流预算：

\[
P_{ij}
=
m_{\mathrm{out}}
\frac{\tilde g_{ij}}
{\sum_k\tilde g_{ik}+\epsilon}
\]

因此，枢纽校正只重分配行内选择，不突破节点总出流预算。

默认参数：

- `hubPenaltyExponent = 0.30`
- `hubPenaltyFloor = 0.55`
- `hubPenaltyCeiling = 1.80`
- `hubSmoothingRatio = 0.10`

## 3. 软非回溯传播

V9.1 的传播状态由节点状态升级为带前驱记忆的边状态。

当传播出现：

\[
u\rightarrow i\rightarrow u
\]

立即回流不再获得完整质量，而是乘以：

\[
\rho_{\mathrm{return}}=0.15
\]

普通汇聚路径和非立即回流路径不受影响。虫洞继续保留动量豁免，但不豁免立即回流抑制。

默认参数：

- `v91ReturnFlowFactor = 0.15`
- `v91MaxPropagationStates = 2000`

传播状态超过上限时，仅保留能量最高的状态，避免高分支图造成状态爆炸。

## 4. 归一化有限时域场

旧 V9 场直接累计每一跳能量：

\[
E=\sum_{t=0}^{T}h^{(t)}
\]

V9.1 使用归一化几何有限脉冲响应：

\[
w_t
=
\frac{\gamma^t}
{\sum_{r=0}^{T}\gamma^r}
\]

\[
E
=
\sum_{t=0}^{T}w_t h^{(t)}
\]

默认：

\[
\gamma=0.60
\]

该方法仍保持有限跳，不求全局稳态，不引入 PPR、heat kernel 或额外模型调用。

## 5. 不变项

以下行为保持不变：

- `v8_3` 资产与查询路径；
- V9 对数证据压缩；
- 节点固定出流上限；
- 虫洞预算内竞争；
- V9 residual anchor；
- 最大跳数和动量机制；
- 涌现节点上限；
- Potential Field 公式及硬门控；
- LightMemo A/B 输入与输出协议；
- embedding、pairwise similarity 和 IR 数据格式。

## 6. Artifact 与启动行为

V9.1 参数已经进入 `rag_params.json`，因此 artifact signature 会与历史 V9 不同。

服务器启动时会从现有事实矩阵和残差资产构造新的 V9.1 内存传播核。此升级不要求重新生成 chunk/tag embedding，也不要求为了算法升级强制重算 pairwise similarity 或 intrinsic residual。

若服务器需要确保全部派生事实同步到最新数据库，可按原流程主动执行完整训练；这不是 V9.1 核生效的必要条件。

## 7. 诊断字段

V9 bundle 增加：

- `algorithmVersion: v9.1`
- `kernelDiagnostics.algorithmVersion`
- `kernelDiagnostics.medianInflow`
- `kernelDiagnostics.targetCount`
- 枢纽校正实际参数

查询 TagMemo 信息增加：

- `propagation.algorithmVersion`
- `propagation.returnFlowSuppressedMass`
- `propagation.stateTruncations`
- `propagation.hopInFlightMass`

## 8. 最短验证方式

直接使用与历史 350 组数据相同的：

- 查询文本；
- 搜索范围；
- K；
- BM25 开关；
- tag boost；
- core tags；
- Potential Field 开关。

复跑现有 V8.3/V9 A/B 即可。新的 V9 侧即为 V9.1。

优先观察：

1. 高频主题与通用标签污染是否减少；
2. V9 独占结果的人工有效率是否继续提高；
3. 跨域桥是否因回流抑制而损失；
4. Top-3 是否稳定；
5. Top-K 尾部噪声是否下降；
6. `stateTruncations` 是否经常大于零；
7. P50/P95 延迟是否可接受。

## 9. 回退

代码回退可撤销 V9.1 提交。

运行时保命回退仍可将：

```json
"activeVersion": "v8_3"
```

作为活动版本。V8.3 bundle 未被 V9.1 修改。