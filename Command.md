# 校园导航系统 —— 命令接口完整说明

> 很重要，做之前一定通读一下，易于后面辨析是否输出有错误！

## 1. LOAD —— 加载数据

从 CSV 文件加载地点和道路数据，替换当前图的所有内容。

```
LOAD <places_file> <roads_file>
```

| 参数 | 说明 |
|------|------|
| `places_file` | 地点 CSV 文件路径 |
| `roads_file` | 道路 CSV 文件路径 |

**输出**：`OK` 或 `ERROR <reason>`

**示例**：
```
输入：LOAD places.csv roads.csv
输出：OK
```


## 2. SAVE —— 保存数据

将当前图的所有地点和道路保存到 CSV 文件。

```
SAVE <places_out_file> <roads_out_file>
```

**输出**：`OK`

**示例**：
```
输入：SAVE output_places.csv output_roads.csv
输出：OK
```


## 3. QUERY_PLACE —— 查询地点信息

查询指定地点的完整信息。

```
QUERY_PLACE <place_id>
```

**输出格式**：
```
PLACE <place_id> <display_name> <category> <stay_time> <open_time> <close_time>
```

**错误**：`ERROR place_not_found`

**示例**：
```
输入：QUERY_PLACE P0001
输出：PLACE P0001 Library Teaching 30 08:00 22:00
```


## 4. QUERY_CATEGORY —— 按类别查询

查询某个类别下的所有地点，按 `place_id` 字典序升序输出。

```
QUERY_CATEGORY <category>
```

**输出格式**：
```
CATEGORY <category> <count> <id1> <id2> ...
```

**示例**：
```
输入：QUERY_CATEGORY Teaching
输出：CATEGORY Teaching 2 P0001 P0005

输入：QUERY_CATEGORY NonExistent
输出：CATEGORY NonExistent 0
```


## 5. ADJ —— 查询邻接道路

查询某个地点的所有相邻道路，按邻接点 `place_id` 字典序升序输出。

```
ADJ <place_id>
```

**输出格式**：
```
ADJ <place_id> <count> <neighbor1>:<distance>:<walk_time>:<status> <neighbor2>:...
```

**错误**：`ERROR place_not_found`

**示例**：
```
输入：ADJ P0001
输出：ADJ P0001 3 P0002:200:3:open P0003:150:2:open P0005:400:5:open
```


## 6. ADD_PLACE —— 新增地点

向图中添加一个新地点。

```
ADD_PLACE <place_id> <display_name> <category> <stay_time> <open_time> <close_time>
```

**输出**：`OK` 或 `ERROR place_already_exists`

**示例**：
```
输入：ADD_PLACE P0007 Cafe Dining 20 07:00 22:00
输出：OK
```


## 7. DELETE_PLACE —— 删除地点

删除指定地点，同时自动删除所有与该地点相连的道路。

```
DELETE_PLACE <place_id>
```

**输出**：`OK` 或 `ERROR place_not_found`


## 8. UPDATE_PLACE —— 修改地点信息

修改地点的某个字段。

```
UPDATE_PLACE <place_id> <field> <value>
```

**支持的字段**：`display_name`、`category`、`stay_time`、`open_time`、`close_time`

**输出**：`OK`、`ERROR place_not_found` 或 `ERROR invalid_field`

**示例**：
```
输入：UPDATE_PLACE P0001 stay_time 45
输出：OK
```


## 9. ADD_ROAD —— 新增道路

向图中添加一条新道路。

```
ADD_ROAD <from_id> <to_id> <distance> <walk_time> <status>
```

**输出**：`OK`、`ERROR place_not_found` 或 `ERROR road_already_exists`

**示例**：
```
输入：ADD_ROAD P0001 P0007 120 2 open
输出：OK
```


## 10. DELETE_ROAD —— 删除道路

从图中删除一条道路。

```
DELETE_ROAD <from_id> <to_id>
```

**输出**：`OK` 或 `ERROR road_not_found`


## 11. UPDATE_ROAD —— 修改道路信息

修改道路的某个字段。

```
UPDATE_ROAD <from_id> <to_id> <field> <value>
```

**支持的字段**：`distance`、`walk_time`、`status`

**输出**：`OK`、`ERROR road_not_found` 或 `ERROR invalid_field`


## 12. CLOSE_ROAD —— 关闭道路

将道路状态设为 `closed`，使其不可通行。

```
CLOSE_ROAD <from_id> <to_id>
```

**输出**：`OK` 或 `ERROR road_not_found`


## 13. OPEN_ROAD —— 打开道路

将道路状态设为 `open`，恢复通行。

```
OPEN_ROAD <from_id> <to_id>
```

**输出**：`OK` 或 `ERROR road_not_found`


## 14. COMPONENTS —— 连通分量分析

计算当前图的连通分量（只考虑 `status = open` 的道路）。

```
COMPONENTS
```

**输出格式**：
```
COMPONENTS <count> SIZES <s1> <s2> ...
```

规模按降序排列。

**示例**：
```
输入：COMPONENTS
输出：COMPONENTS 2 SIZES 5 3
```


## 15. SHORTEST —— 最短路径

计算两点间的最短路径（按距离或按时间）。

```
SHORTEST <from_id> <to_id> <DIST|TIME>
```

| 参数 | 说明 |
|------|------|
| `DIST` | 按道路长度（distance）最短 |
| `TIME` | 按步行时间（walk_time）最短 |

**输出格式**：
```
PATH <DIST|TIME> <total_cost> NODES <id1> <id2> ...
```

**错误**：
- 不可达：`NO_PATH`
- 地点不存在：`ERROR place_not_found`

**示例**：
```
输入：SHORTEST P0001 P0006 DIST
输出：PATH DIST 680 NODES P0001 P0003 P0005 P0006
```


## 15'. TIMED_SHORTEST —— 时刻约束最短路径

在指定时刻下计算最短路径，只走当时开放的地点。

```
TIMED_SHORTEST <from_id> <to_id> <time> <DIST|TIME>
```

| 参数 | 说明 |
|------|------|
| `time` | 查询时刻，HH:MM 格式（如 19:00）|
| `DIST` | 按距离最短 |
| `TIME` | 按时间最短 |

**约束条件**：
- 只走 `status = open` 的道路
- 路径上所有地点（含起点和终点）必须在 `time` 时开放，即 `open_time ≤ time ≤ close_time`
- 起点或终点未开放直接返回不可达

**输出**：同 `SHORTEST` 格式

**示例**：
```
输入：TIMED_SHORTEST P0001 P0005 19:00 DIST
输出：NO_PATH
（P0005 在 19:00 已关闭）
```


## 16. MUST_PASS —— 必经点路径

必须按指定顺序经过若干地点的最短路径。

```
MUST_PASS <from_id> <to_id> <DIST|TIME> <k> <p1> <p2> ... <pk>
```

| 参数 | 说明 |
|------|------|
| `k` | 必经点个数 |
| `p1`~`pk` | 必经点列表，**必须按此顺序依次经过** |

**输出**：同 `SHORTEST` 格式

**示例**：
```
输入：MUST_PASS P0001 P0007 DIST 2 P0003 P0006
输出：PATH DIST 760 NODES P0001 P0002 P0003 P0005 P0006 P0007
```


## 17. MST —— 最小生成树

计算图的最小生成树（按 `distance`，只考虑 `open` 边）。

```
MST
```

**输出格式**：
```
MST <total_distance> EDGES <u1>-<v1>:<w1> <u2>-<v2>:<w2> ...
```

- 边按 `(min(u,v), max(u,v))` 字典序排序
- 每条边中 `u` 和 `v` 取字典序较小者在前

**错误**：图不连通时输出 `DISCONNECTED`

**示例**：
```
输入：MST
输出：MST 980 EDGES P0001-P0002:200 P0001-P0003:150 P0003-P0005:180 P0004-P0005:100 P0005-P0006:350
```


## 18. CRITICAL —— 关键节点与关键边

找出图中的关键节点和关键边（只考虑 `open` 边）。

- **关键节点**：删除该节点后，连通分量数增加的顶点
- **关键边**：删除该边后，连通分量数增加的边

```
CRITICAL
```

**输出格式**：
```
CRITICAL NODES <node_count> <id1> <id2> ... EDGES <edge_count> <u1>-<v1> <u2>-<v2> ...
```

- 关键节点按 `place_id` 字典序排序
- 关键边按 `(min(u,v), max(u,v))` 字典序排序

**示例**：
```
输入：CRITICAL
输出：CRITICAL NODES 2 P0005 P0006 EDGES 1 P0005-P0006
```


## 19. QUIT —— 退出

退出程序。

```
QUIT
```

**无输出**（程序终止）


## 命令速查表

| 命令 | 格式 | 核心输出 |
|------|------|----------|
| LOAD | `LOAD <places> <roads>` | `OK` |
| SAVE | `SAVE <places> <roads>` | `OK` |
| QUERY_PLACE | `QUERY_PLACE <id>` | `PLACE ...` |
| QUERY_CATEGORY | `QUERY_CATEGORY <cat>` | `CATEGORY ...` |
| ADJ | `ADJ <id>` | `ADJ ...` |
| ADD_PLACE | `ADD_PLACE <id> <name> <cat> <stay> <open> <close>` | `OK` |
| DELETE_PLACE | `DELETE_PLACE <id>` | `OK` |
| UPDATE_PLACE | `UPDATE_PLACE <id> <field> <value>` | `OK` |
| ADD_ROAD | `ADD_ROAD <from> <to> <dist> <time> <status>` | `OK` |
| DELETE_ROAD | `DELETE_ROAD <from> <to>` | `OK` |
| UPDATE_ROAD | `UPDATE_ROAD <from> <to> <field> <value>` | `OK` |
| CLOSE_ROAD | `CLOSE_ROAD <from> <to>` | `OK` |
| OPEN_ROAD | `OPEN_ROAD <from> <to>` | `OK` |
| COMPONENTS | `COMPONENTS` | `COMPONENTS <cnt> SIZES ...` |
| SHORTEST | `SHORTEST <from> <to> <DIST\|TIME>` | `PATH ...` |
| TIMED_SHORTEST | `TIMED_SHORTEST <from> <to> <time> <DIST\|TIME>` | `PATH ...` |
| MUST_PASS | `MUST_PASS <from> <to> <DIST\|TIME> <k> <p1>...` | `PATH ...` |
| MST | `MST` | `MST <total> EDGES ...` |
| CRITICAL | `CRITICAL` | `CRITICAL NODES ... EDGES ...` |
| QUIT | `QUIT` | （程序退出）|
