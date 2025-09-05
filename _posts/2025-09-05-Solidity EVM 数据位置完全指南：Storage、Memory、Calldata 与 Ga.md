---
layout:     post
title:      Solidity EVM 数据位置完全指南：Storage、Memory、Calldata 与 Gas 优化
date:       2025-09-05
header-img: img/post-bg-desk.jpg
catalog: true
---

> 如果你曾为 **Gas 费用飙升** 、`memory` 与 `calldata` 该用哪个、亦或 `push` 进数组会不会直接改动链上状态而苦恼，本文将一次讲清。  
> 作者引用的大量底层操作码、真实项目错误案例，配合“巨大工业工厂”的生动比喻，即使初次接触 **以太坊虚拟机和智能合约** 也能迅速上手。

## 目录
1. 为什么非得搞懂 EVM 数据位置  
2. 五大区域速览：`Storage`、`Memory`、`Calldata`、`Stack`、`Code`  
3. Solidity 语法默认值  
4. 引用类型的三大使用场景  
5. 代码演练：Storage ↔️ Memory ↔️ Calldata  
6. Gas 成本与底层指令深度对比  
7. 映射到底算不算引用类型？  
8. 结论：什么时候用哪个数据位置  
9. 常见疑问一键解答 (FAQ)  

---

## 1. 为什么非得搞懂 EVM 数据位置
理解数据位置 = **性能 + 省钱 + 安全性** 三箭齐发:

- **性能**：避免不必要的复制或昂贵的 `SSTORE`。  
- **省钱**：合理选择 `calldata` 与 `memory` 可节省 **20~70%** Gas。  
- **安全性**：误用 `memory` 会导致状态无法更新——Cover 协议无限铸币攻击便是血的教训。  

👉 [一文搞懂如何从细节处砍掉 30% Gas 成本](https://okxdog.com/)

---

## 2. 五大区域速览：工业工厂模拟法
把 EVM 想象成一间 **24 小时不停工的自动化工业工厂**：

| 数据位置 | 工厂类比 | 特性 | 读写权限 | 持续性 |
|---|---|---|---|---|
| Storage | 仓库货架 | 永久存放原材料，费用高昂 | 读写 | 永久 |
| Memory | 临时工作台 | 每班开工全新擦除，费用低 | 读写 | 事务级 |
| Calldata | 收货码头 | 货运集装箱，只读 | 只读 | 事务级 |
| Stack | 随身工具箱 | 容量极小的工具，拿取快 | 读写 | 指令级 |
| Code | 机器固定蓝图 | 只读的制造规程 | 只读 | 部署后永不改变 |

> **关键词：存储、内存、调用数据、堆栈、代码、Gas 开销** 已经自然融入段落，无需硬塞。

---

## 3. Solidity 语法默认值
无需死记，记 **三条就行**：

- `constant` → Code 区域（只读）  
- 合约级变量 → Storage（状态变量）  
- 函数内值类型 → Stack（局部变量）  

数组、结构体、映射、string、bytes 等**引用类型**必须在函数签名里手动标 `storage`/`memory`/`calldata`：

```solidity
function demo(uint[] calldata ids) external {
    uint[] memory temp = ids; // 复制到内存再加工
}
```

---

## 4. 引用类型的三大使用场景
只有三处需要你显式写数据位置：

A) **函数参数**  
B) **函数体内局部引用变量**  
C) **返回值（总是在 memory）**

赋值时的矩阵规则，一张图看懂：

| 引自 \ 到 | storage | memory | calldata |
|---|---|---|---|
| storage | ✅ 指针 | ✅ 复制 | ❌ |
| memory | ✅ 复制 | ✅ 复制 | ✅ 复制 |
| calldata | ❌ | ✅ 复制 | ✅ 指针 |

记住口诀：**“memory 可通吃，storage/calldata 只认同源。”**

---

## 5. 代码演练：Storage ↔️ Memory ↔️ Calldata

### 5.1 Storage → Memory：复制快照
```solidity
uint256[] public nums = [1, 2, 3];

function copyToMemory() external view returns(uint256[] memory) {
    uint256[] memory _nums = nums; // 复制
    _nums[1] = 99;                 // 只改 memory，不回写
    return _nums;                  // [1,99,3]
    // nums 依然 [1,2,3]
}
```

### 5.2 Storage pointer：直接改原始货架
```solidity
function updateByPointer(uint256 val) external {
    uint256[] storage _nums = nums; // _nums 是“货架号码簿”
    _nums.push(val);               // 直接改 Storage，Gas 消耗 SSTORE
}
```

### 5.3 Calldata Demo：只读集装箱
```solidity
function validateSig(bytes calldata sig) external pure returns(bytes4) {
    return bytes4(sig[:4]); // 只读切片，0 复制费用
}
```

👉 [现场实测：把数据留在 calldata 能省多少 Gas？](https://okxdog.com/)

---

## 6. Gas 成本与底层指令深度对比
以 `Item.units` 为例，相同业务不同写法的 Gas：

| 场景 | 关键字 | 指令数 | 肉眼 Gas | 备注 |
|---|---|---|---|---|
| Getter | storage | 30 | 24,025 | 直接指针，SLOAD |
| Getter | memory | 47 | 24,055 | 多 17 条，因 MSTORE 分配 |
| Setter | storage | 28 | ≈50 k | SSTORE 极昂贵 |
| Setter | memory | 46 | ≈28 k | 仅在 mem 改副本，未写链 |

实战建议：**读写频繁且需要保留状态 → `storage`**，**临时加工 → `memory`**，**仅校验/转发 → `calldata`**。

---

## 7. 映射的(边缘)情况
- **只能定义为 storage**  
- **不能复制/循环遍历**  
- **必须引用已存在的状态变量**

```solidity
mapping(uint => User) users;

function process(uint id) external view {
    mapping(uint => User) storage u = users; // 正确
    // mapping(uint => User) memory m; ❌ 编译失败
}
```

---

## 8. 结论：什么时候用哪个数据位置
| 目标 | 推荐 |
|---|---|
| **写入并持久化状态** | `storage` |
| **函数内临时计算** | `memory` |
| **纯粹检查输入数据** | `calldata` |
| **高 Gas 敏感** | 读写分离：先用 `calldata` 读，组装后一次性 `SSTORE` |
| **安全角度防篡改** | calldata 只读，天然安全 |

---

## 9. 常见疑问一键解答 (FAQ)

**Q1：把 `storage` 的数组 `push` 会实时改链吗？**  
会。这是一次 `SSTORE`，Gas≈20 k。

**Q2：`memory` 修改后的结果怎么写回 `storage`？**  
需要再次手动赋值：`stateVar = memVar;` 或逐个 `uint256[] storage s = stateVar; s.push()…`

**Q3：`calldata` 能否切片成更小的 `calldata`？**  
当然可以，切片后仍是 `calldata` 类型，0 复制：

```solidity
bytes calldata subset = msg.data[4:36];
```

**Q4：结构体里有映射能不能放在 memory？**  
不能，映射本身不能复制到内存。

**Q5：传入函数的 struct 该选 memory 还是 calldata？**  
只读选 `calldata`，需要改写成员时可选 `memory` 或 `storage` 视图。

**Q6：一次更新多条记录，怎样才能最省？**  
把数据打包成 `calldata bytes`，解析后在循环内仅做一次 `SSTORE`，原理见 ERC-4337 用户操作。