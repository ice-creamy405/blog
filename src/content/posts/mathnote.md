---
title: 数学建模笔记分享
published: 2026-07-11
pinned: true
description: 暑假准备数学建模笔记分享
image: ./images/cover2.avif
tags:
  - learning
  - note
category: learning
---
# 遗传优化算法
1. 十进制->二进制
2. 二进制编码，随机生成染色体 （种群初始化）
3. 选择（适者生存）->基因重组（交叉）->基因突变（随机选择部分基因变异）
4. 最大迭代次数
5. 解码

种群初始化：$X_0 = rand \times (UB-LB) +LB$  (rand = [0,1])
### 核心操作
- 选择：根据个体适应度值选择每个个体在子代中出现的概率， $$P(i) (选择概率) = \frac {f_i(适应度)}{\sum_{j=1}^N f_i}$$ /锦标赛，随机采样s个，选择最优的进入下一轮
- 交叉：
	- 单点交叉：两条染色体随机一个位置分割，交换右侧基因，得到两个不同的子染色体
	- 随机两个，交换中间
	- 部分匹配：2+冲突检测（根据交换部分的映射关系修改）
- 变异
# 模拟退火算法
'''mermaid
flowchart TD
    A([开始]) --> B[设置算法参数<br/>随机产生初始值]
    B --> C[外层循环: 控制温度下降]
    C --> D[内层循环: 每个温度L次状态转移]
    D --> E[根据当前解加随机扰动生成新解]
    E --> F[根据Metropolis准则接受新解]
    F --> G[更新历史最优解]
    G --> H[降温操作]
    H --> I{温度是否达到阈值或<br/>适应度差值达到阈值}
    I -- N --> C
    I -- Y --> J[输出最优解]
    note right of B: 设定取值范围、温度下降系数、马尔可夫链长度等
    note right of C: 控制算法循环: 前期下降速度快, 后期慢
    note right of D: 寻找当前温度下的最优解
    note right of E: 在当前解附近随机生成新解
    note right of F: 接受更优解; 概率接收劣解
    note right of G: 记录全局最优解
    note right of H: 降低温度, 进入下一轮循环
    note right of I: 温度降低至常温(达到稳定态)<br/S>新解与当前解差值较小(几乎不更新)
'''

<script src="https://giscus.app/client.js"
        data-repo="ice-creamy405/blog"
        data-repo-id="R_kgDOTsZHBg"
        data-category="General"
        data-category-id="DIC_kwDOTsZHBs4DC7cK"
        data-mapping="title"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="1"
        data-input-position="bottom"
        data-theme="preferred_color_scheme"
        data-lang="zh-CN"
        data-loading="lazy"
        crossorigin="anonymous"
        async>
</script>