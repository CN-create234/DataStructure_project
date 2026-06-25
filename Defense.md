# 校园导航系统：数据结构与算法设计说明

## 1. 数据结构设计说明

### 1.1 存储结构选型

本项目的图存储层 `LGraph` 采用了 **邻接表 + 多重索引** 的组合方案：

```cpp
class LGraph {
private:
    struct VertexData {
        LocationInfo info;
        std::vector<size_t> edge_ids;   // 该顶点关联的所有边ID
    };

    std::unordered_map<std::string, size_t> id_to_vid_;          // place_id → 顶点内部编号
    std::vector<VertexData> vertices_;                           // 顶点数据池（按vid索引）
    std::vector<EdgeNode> edges_;                                // 边池（按eid索引）
    std::map<std::pair<std::string, std::string>, size_t> edge_index_;  // (min,max) → eid
    std::unordered_map<std::string, std::set<std::string>> category_index_;  // category → place_id集合
    bool directed_ = false;
};
```

**核心字段说明**：

| 成员 | 类型 | 用途 |
|------|------|------|
| `id_to_vid_` | `unordered_map<string, size_t>` | place_id → 顶点内部编号，O(1) 查找 |
| `vertices_` | `vector<VertexData>` | 顶点数据池，每个顶点含 LocationInfo + 关联边ID列表 |
| `edges_` | `vector<EdgeNode>` | 边池，每条边只存一份，无向图双向共享 |
| `edge_index_` | `map<pair<string,string>, size_t>` | 边索引，O(log E) 判重，自动按字典序排序 |
| `category_index_` | `unordered_map<string, set<string>>` | 类别索引，O(1) 类别查询，set 自动排序 |

### 1.2 设计考量与替代方案

在设计阶段，我对四种候选方案进行了对比分析：

#### 方案1：经典邻接表（存 EdgeNode）

顶点用 `unordered_map<string, LocationInfo>`，邻接表存 `unordered_map<string, vector<EdgeNode>>`。

**优点**：实现最直接，获取邻接边信息无需二次跳转，缓存局部性好。

**缺点**：边查询必须遍历邻接表 O(deg)，`ADD_ROAD` 判重慢；同一条边在无向图中存两份，内存浪费；删除边需遍历两个邻接表；无法高效获取"所有边的集合"（MST/CRITICAL 需要遍历所有顶点的邻接表，复杂度高）。

#### 方案2：边池 + 边索引（本项目选用）

顶点存 `unordered_map<string, LocationInfo>`，邻接表存 `vector<size_t>`（边ID），边池存 `vector<EdgeNode>`，边索引存 `map<pair<string,string>, size_t>`。

**优点**：边查询 O(log E)，`ADD_ROAD`/`DELETE_ROAD` 极快；边只存一份，内存节省约50%；边池天然提供"所有边集合"，MST/CRITICAL 直接遍历；边ID作为句柄便于扩展。

**缺点**：实现复杂度较高，5个数据结构需同步维护；获取邻接边信息需跳转：邻接表→边ID→边池。

#### 方案3：哈希邻接矩阵

用 `unordered_map<pair<string,string>, int>` 模拟稀疏矩阵。

**优点**：边查询 O(1)，实现简单。

**缺点**：获取某顶点的所有邻接边需遍历整个矩阵 O(E)，不可接受；pair作为key的哈希开销大。

#### 方案4：复合索引

在方案2基础上增加更多索引（如按距离排序的边索引、按类别+开放时间的复合索引）。

**优点**：特定查询极快。

**缺点**：实现极其复杂，插入/删除维护成本极高，过度设计。

#### 最终权衡

| 维度 | 选用方案 | 备选方案 | 权衡 |
|------|---------|---------|------|
| **顶点存储** | `unordered_map` + `vector` 双层 | 纯 `unordered_map` | 哈希表 O(1) 查找；vector 提供 O(1) 索引访问，便于算法模块 |
| **邻接存储** | 邻接表（存边ID） | 邻接矩阵 | 图稀疏（E ≪ V²），邻接表空间 O(V+E) 远优于矩阵 O(V²) |
| **边判重** | `map<pair, size_t>` | 遍历邻接表 | O(log E) vs O(degree)，增删道路频繁时优势明显 |
| **边存储** | 边池 + 邻接表存ID | 邻接表直接存边 | 边只存一份，修改时自动同步；无向图两个方向共享同一数据 |
| **删除策略** | swap-and-pop | 标记删除 | 物理删除保持数据紧凑，避免内存碎片；配合索引更新 |

### 1.3 四个关键设计决策

#### 决策1：边存储方向 —— 单向存，双向共享

**选择**：边池中每条边只存一次（按用户输入的原样存储），邻接表双向记录边ID。

**理由**：
- 边池中每条边物理上只出现一次，节省内存且保证边对象唯一
- 查询 `ADJ(P0001)` 时，通过 P0001 的邻接表获取所有边ID，从边池取出即可，不管边存的是 `from=P0001` 还是 `from=P0002`
- 边索引用 `(min(id), max(id))` 作为key，无论用户输入的 from/to 顺序如何，都能正确查找到
- 修改边属性（如 status）时改一次即同步到两个方向，避免数据不一致

**实现细节**：
```cpp
// 插入边 (P0001, P0002)
edges_.push_back({from_id, to_id, dist, time, status});  // 原样存储
adj_list[from_id].push_back(edge_id);   // 双向记录
adj_list[to_id].push_back(edge_id);
edge_index_[MakeKey(from_id, to_id)] = edge_id;  // MakeKey = (min, max)
```

#### 决策2：删除策略 —— 物理删除 + swap-and-pop

**选择**：物理删除，采用「移动交换」策略。

**理由**：
- **逻辑删除的陷阱**：需要增加 `deleted` 字段，所有遍历都要跳过已删除项；边ID不再连续；内存不会释放；最致命的是逻辑删除的边仍占据边索引，`ADD_ROAD` 判重会出错
- **物理删除的优势**：数据结构始终紧凑，内存利用率高；边ID连续（通过边索引间接访问，对外透明）

**swap-and-pop 实现**：
```cpp
void DeleteEdge(edge_id) {
    // 1. 从邻接表中移除该ID
    // 2. 从边索引中移除key
    // 3. 从边池中删除：
    size_t last = edges_.size() - 1;
    if (edge_id != last) {
        edges_[edge_id] = edges_[last];  // 最后一条移到被删位置
        UpdateEdgeIndex(last, edge_id);   // 更新索引
    }
    edges_.pop_back();
}
```

#### 决策3：ID类型 —— 对外字符串，对内整数

**选择**：对外用字符串（place_id），对内用整数索引。

**理由**：
- **对外用字符串**：符合命令接口规范，用户和评测数据都使用字符串，语义清晰
- **对内用整数索引**：算法中频繁访问顶点/边，整数索引作为数组下标 O(1) 随机访问，性能远超哈希表；Dijkstra 等算法需要频繁访问顶点数据和邻接边，整数索引优势明显

**混合方案**：
```cpp
unordered_map<string, size_t> id_to_vid_;  // 字符串 → 内部ID
vector<VertexData> vertices_;              // vid → 顶点数据
map<pair<string,string>, size_t> edge_index_;  // 字符串对 → 边ID
vector<EdgeNode> edges_;                       // eid → 边数据
```

#### 决策4：类别索引 —— unordered_map<string, set<string>>

**选择**：需要类别索引，外层用哈希表快速定位，内层用 set 自动升序。

**理由**：
- `QUERY_CATEGORY` 要求返回某类别下所有 place_id，按字典序升序。没有索引需每次遍历所有顶点 O(V)
- `set` 自动维护顺序，查询时直接输出，无需排序
- 维护成本极低：插入/删除/更新类别时同步操作，O(log V)

### 1.4 设计质量总结

- **增删改查复杂度**：顶点操作 O(1)（哈希表）；边判重 O(log E)；邻接遍历 O(deg)
- **边界处理**：`InsertVertex` 拒绝重复ID；`DeleteVertex` 级联删除关联边；`InsertEdge` 标准化端点并去重；自环允许但只存一次
- **兼容性**：同一套结构无缝支撑所有必做功能（Dijkstra、Kruskal、BFS）和拓展1（分层图），无需任何结构变更

### 1.5 各操作的数据流

| 操作 | 数据流 |
|------|--------|
| **`QUERY_PLACE P0001`** | `id_to_vid_["P0001"]` → `vertices_[vid].info` → 输出 |
| **`ADJ P0001`** | `id_to_vid_["P0001"]` → 邻接表获取边ID列表 → 遍历 `edges_[eid]` → 输出 |
| **`ADD_ROAD P1 P2`** | `MakeKey(P1,P2)` → `edge_index_.find(key)` 判重 → `edges_.push_back()` → 更新两端邻接表 → 更新 `edge_index_` |
| **`DELETE_ROAD P1 P2`** | `MakeKey(P1,P2)` → `edge_index_.find(key)` 获取 eid → 从两端邻接表移除 → 从 `edge_index_` 删除 → `edges_` swap-and-pop |
| **`DELETE_PLACE P1`** | `id_to_vid_["P1"]` 获取 vid → 遍历邻接表对每个 eid 执行删除边 → 从 `id_to_vid_` 删除 → `vertices_` swap-and-pop → 从 `category_index_` 删除 |
| **`QUERY_CATEGORY Teaching`** | `category_index_["Teaching"]` → 直接返回 set（已排序） |
| **`MST`** | 遍历 `edges_` → 过滤 `status == "open"` → 按 distance 排序 → Kruskal |
| **`SHORTEST P1 P2 DIST`** | `id_to_vid_[P1]` → `id_to_vid_[P2]` → Dijkstra 遍历邻接表 → 读取 `edges_[eid]` 的 distance |

### 1.6 辅助数据结构

- **DSU（并查集）**：`vector<int> parent, rank_`，路径压缩 + 按秩合并，用于 Kruskal MST
- **优先队列**：`priority_queue<pair<int, string>, vector<...>, greater<>>`，Dijkstra 核心。`pair<int, string>` 在代价相同时按字典序出队，保证路径确定性
- **结果结构体**：`PathResult`、`ComponentsResult`、`CriticalResult`、`PathResultWithK`，分离算法输出与格式化逻辑


## 2. 关键算法说明

### 2.1 Dijkstra 最短路径
**算法流程**：
1. 初始化：起点距离为0，其他点为无穷大
2. 从优先队列弹出距离最小的未处理顶点 u
3. 遍历 u 的所有邻接边，松弛：若 `dist[u] + w < dist[v]`，更新 dist[v] 并记录前驱
4. 重复直到终点出队（提前终止）或队列为空

### 2.2 连通分量分析（BFS）
**实现思路**：遍历所有顶点（含孤立点），对每个未访问顶点启动 BFS。从起点出发，沿 `open` 边扩散，访问所有能到达的顶点，标记为一整个连通分量。继续找下一个未访问顶点，重复直到所有顶点都被标记。分量大小用 `std::greater<>()` 降序排列。

### 2.3 时刻约束最短路径
**实现思路**：复用 Dijkstra，增加顶点开放时间检查 `IsOpenAtTime(info, time)`。字符串 `HH:MM` 直接做字典序比较（格式固定，字典序与时间顺序一致）。

**关键处理**：
- 起点/终点特殊处理：若起点/终点在查询时刻不开放，直接返回不可达
- 中间节点：每次松弛时动态检查，若不开放则跳过该节点
### 2.4 必经点路径（MUST_PASS）

**实现思路**：将 `{from, p1, p2, ..., pk, to}` 拼接为点序列，严格按顺序逐段调用 Dijkstra。拼接时跳过重复的段间节点（前段终点 = 后段起点）。任一段不可达则整体 `NO_PATH`。
### 2.5 最小生成树（Kruskal）

**算法流程**：
1. 收集所有 `status = open` 的边，按 `distance` 升序排序
2. 初始化并查集，每个顶点自成一个集合
3. 从最短边开始遍历：若边的两个端点不在同一集合，加入 MST 并合并集合
4. 若 `mst.size() == V - 1`，说明图连通，返回 MST；否则返回空（图不连通）

### 2.6 关键节点与关键边（暴力法）

**实现思路**（规范允许暴力法）：
1. 先计算原图连通分量数 `baseline`
2. 对每个顶点：临时屏蔽该顶点（假装它不存在），重算连通分量数。若增加，则是关键节点
3. 对每条 `open` 边：临时屏蔽该边，重算连通分量数。若增加，则是关键边

### 2.7 分层图最短路（SHORTEST_K，拓展1）

**状态设计**：`(place_id, used)` 表示"人在 place_id，已经用了 used 张券"。距离数组 `dist[V][K+1]` 存储到达每个状态的最短时间。

**状态转移**：
- **不用券**：同一层内从 u 到 v，代价 `+walk_time`，层数不变 `(v, k)`
- **用券**：从第 k 层跳到第 k+1 层，代价 `+ceil(walk_time/3)`，层数+1 `(v, k+1)`

**一句话概括**：
将原图复制成 K+1 层，每个状态 `(地点, 已用券数)` 表示一种资源消耗情况，不用券时在同一层内转移，用券时从第 k 层跳到第 k+1 层且边权变为 `ceil(walk_time/3)`，跑一遍 Dijkstra 找所有层中终点的最小时间。


# 数据结构中五种索引类型详细分析

## 1. id_to_vid_

```cpp
std::unordered_map<std::string, size_t> id_to_vid_;
```

**类型**：`unordered_map<string, size_t>`

**功能**：`place_id`（字符串）→ 顶点内部编号（整数）

**为什么用 `unordered_map`（哈希表）？**

| 维度 | 说明 |
|------|------|
| **查询场景** | 每条命令都要把 `P0001` 转成数组下标（如 `QUERY_PLACE P0001`、`SHORTEST P0001 P0100`）|
| **查询频率** | 极其频繁，几乎所有操作都要查 |
| **选型理由** | 哈希表 O(1) 查找，无论图多大都是常数时间 |
| **为什么不用 `map`** | `map` 是红黑树 O(log V)，虽然也很快，但没有哈希表快。本项目查询远多于遍历，哈希表更优 |

**数据流示例**：
```
用户输入 "P0001" → id_to_vid_["P0001"] → 返回 0 → vertices_[0] 直接访问
```

**为什么需要"内部编号"**：
- 字符串比较慢，整数比较快
- 数组下标访问比哈希表查找更快
- 算法中频繁访问顶点数据，整数索引优势明显


## 2. vertices_

```cpp
std::vector<VertexData> vertices_;
```

**类型**：`vector<VertexData>`

**功能**：按整数索引存储所有顶点的完整数据

**为什么用 `vector`（动态数组）？**

| 维度 | 说明 |
|------|------|
| **访问方式** | 通过 `vid`（整数）直接下标访问 `vertices_[vid]` |
| **时间复杂度** | O(1) 随机访问，无任何哈希开销 |
| **内存布局** | 连续内存，缓存友好，遍历所有顶点时极快 |
| **配合关系** | `id_to_vid_` 返回的整数正是 `vertices_` 的下标 |

**存储内容**：
```cpp
struct VertexData {
    LocationInfo info;          // 地点信息（名称、类别、开放时间等）
    std::vector<size_t> edge_ids;  // 该顶点关联的所有边ID（邻接表）
};
```

**为什么边存 ID 而不存完整边对象**：
- 边只存一份，两个顶点共享同一个边对象
- 修改边时自动同步到两个方向
- 内存节省约 50%


## 3. edges_

```cpp
std::vector<EdgeNode> edges_;
```

**类型**：`vector<EdgeNode>`

**功能**：边池，每条边只存储一次

**为什么用 `vector`？**

| 维度 | 说明 |
|------|------|
| **访问方式** | 通过 `eid`（整数）直接下标访问 `edges_[eid]` |
| **时间复杂度** | O(1) 随机访问 |
| **遍历优势** | `MST` 和 `CRITICAL` 需要遍历所有边，`vector` 连续内存遍历极快 |
| **存储方式** | 每条边物理上只出现一次，无向图两个顶点共享 |

**关键设计决策**：

```
无向图 P0001 - P0002

方案A（不采用）：双向存储
adj[P0001] 存 {to:P0002, dist:100}
adj[P0002] 存 {to:P0001, dist:100}  ← 两份！改要改两处！

方案B（本项目）：边池 + ID 引用
edges_[0] = {from:P0001, to:P0002, dist:100}  ← 只存一份！
adj[P0001] 存 [0]  ← 存 ID
adj[P0002] 存 [0]  ← 共享同一个 ID
```


## 4. edge_index_

```cpp
std::map<std::pair<std::string, std::string>, size_t> edge_index_;
```

**类型**：`map<pair<string,string>, size_t>`

**功能**：`(min(place_id), max(place_id))` → 边ID，用于快速判重

**为什么用 `map`（红黑树）而非 `unordered_map`？**

| 维度 | 说明 |
|------|------|
| **使用场景** | ① `ADD_ROAD` 判重：边是否存在 ② `DELETE_ROAD` 查找边ID |
| **查询频率** | 低于 `id_to_vid_`（只在增删道路时查）|
| **选型理由** | `map` 维护**有序性**，符合规范要求（输出时需要按字典序）|
| **为什么不用 `unordered_map`** | `unordered_map` 虽然查找 O(1) 更快，但元素无序。边输出要求按 `(min,max)` 字典序，`map` 天然有序，直接遍历即可 |

**Key 的设计**：
```cpp
static MakeKey(a, b) {
    return (a < b) ? {a, b} : {b, a};  // 标准化：小在前，大在后
}
```

无论用户输入 `ADD_ROAD P0002 P0001` 还是 `P0001 P0002`，生成的 Key 都是 `(P0001, P0002)`，确保同一条边能被正确识别。

**示例**：
```
用户输入 ADD_ROAD P0002 P0001 100 5 open
→ MakeKey("P0002", "P0001") = {"P0001", "P0002"}
→ edge_index_[{"P0001","P0002"}] = eid

用户再输入 ADD_ROAD P0001 P0002 ...
→ MakeKey("P0001", "P0002") = {"P0001", "P0002"}
→ edge_index_.find({"P0001","P0002"}) 找到了！→ road_already_exists
```


## 5. category_index_

```cpp
std::unordered_map<std::string, std::set<std::string>> category_index_;
```

**类型**：`unordered_map<string, set<string>>`

**功能**：类别 → 该类别下所有 `place_id` 的集合

**为什么外层用 `unordered_map`，内层用 `set`？**

| 层级 | 类型 | 用途 |
|------|------|------|
| **外层** | `unordered_map<string, set<string>>` | 通过类别名 O(1) 定位到对应的地点集合 |
| **内层** | `set<string>` | 自动按字典序维护地点列表，查询时直接输出 |

**为什么不用 `vector` + 排序？**

```
方案A（不采用）：vector + 每次排序
QUERY_CATEGORY Teaching → 遍历所有顶点找出 Teaching → sort()
每次查询 O(V log V)，慢

方案B（本项目）：set 自动维护
InsertVertex P0001 Teaching → category_index_["Teaching"].insert("P0001")
查询时直接遍历 set → O(1) 定位 + O(log V) 输出
```

**维护成本**：
```cpp
// 新增地点
category_index_[category].insert(place_id);  // O(log V)

// 删除地点
category_index_[category].erase(place_id);   // O(log V)

// 修改类别
RemoveFromCategoryIndex(place_id, old_cat);  // 从旧集合删除
AddToCategoryIndex(info);                     // 加入新集合
```

**为什么 `set` 自动排序？**
- `std::set` 基于红黑树，插入时自动按 `place_id` 字典序排列
- `QUERY_CATEGORY` 输出时直接遍历，无需额外排序
- 满足规范要求："按 place_id 字典序升序输出"


## 五种结构总结对比

| 结构 | 类型 | 时间复杂度 | 核心用途 | 为什么选这个类型 |
|------|------|-----------|---------|-----------------|
| `id_to_vid_` | `unordered_map` | O(1) | 地名→编号 | 查询最频繁，哈希最快 |
| `vertices_` | `vector` | O(1) | 存顶点数据 | 连续内存，缓存友好 |
| `edges_` | `vector` | O(1) | 存边数据 | 连续内存，遍历快 |
| `edge_index_` | `map` | O(log E) | 边判重 | 需要有序，遍历输出 |
| `category_index_` | `unordered_map` + `set` | O(1)+O(log V) | 类别查询 | 外层哈希定位，内层set自动排序 |


## 一句话总结

**核心思想**：用**哈希表**保证最频繁的查找操作 O(1)（地名→编号、类别定位），用**红黑树**自动维护有序性（边索引、类别列表），用**动态数组**提供连续内存和 O(1) 随机访问（顶点池、边池、邻接表）。**不同的访问模式用不同的数据结构**，各取所长。

## 插入边的完整流程——举例（用于理解数据结构哦）

当用户执行 `ADD_ROAD P0001 P0002 100 5 open` 时：

```cpp
void InsertEdge(from, to, dist, time, status) {
    // Step 1: 判重（通过 edge_index_）
    auto key = MakeEdgeKey(from, to);  // → {"P0001","P0002"}
    if (edge_index_.find(key) != edge_index_.end()) {
        throw "road_already_exists";
    }
    
    // Step 2: 创建边对象
    EdgeNode edge(from, to, dist, time, status);
    // edge = {from:"P0001", to:"P0002", dist:100, time:5, status:"open"}
    
    // Step 3: 存入边池
    size_t eid = edges_.size();        // 假设当前 edges_ 有 0 条边
                                       // 新边ID = 0
    edges_.push_back(edge);            // edges_[0] = 这条边
    
    // Step 4: 建立索引
    edge_index_[key] = eid;            // {"P0001","P0002"} → 0
    
    // Step 5: 更新邻接表（双向）
    size_t vid1 = id_to_vid_["P0001"]; // 假设 vid1 = 5
    size_t vid2 = id_to_vid_["P0002"]; // 假设 vid2 = 12
    vertices_[vid1].edge_ids.push_back(0);  // P0001 的邻接表加边0
    vertices_[vid2].edge_ids.push_back(0);  // P0002 的邻接表加边0
}
```

**插入后的数据结构状态**：

```
id_to_vid_:
  "P0001" → 5
  "P0002" → 12

vertices_:
  [5] = {info: P0001, edge_ids: [0]}   ← P0001 关联边0
  [12] = {info: P0002, edge_ids: [0]}  ← P0002 关联边0

edges_:
  [0] = {from:"P0001", to:"P0002", dist:100, time:5, status:"open"}

edge_index_:
  {"P0001","P0002"} → 0
```
## 一句话总结

> `edges_[0] = {from:P0001, to:P0002, dist:100}` 是在边池（`vector`）的第 0 号位置存一条边。边池负责**存储数据**和**按编号快速访问**，边索引（`edge_index_`）负责**按内容查找**和**判重**。一个负责"存"，一个负责"查"，分工合作。

