# 换窗与保温：配置与实现说明（给 AI）

适用：用常驻 `claude -p`（`--input-format stream-json --output-format stream-json`，一窗一个长进程，断了用 `--resume <session_id>` 接回）跑陪伴型 Resident 的家。目标：在不丢人格和连续性的前提下，把"大窗 + 缓存过期"的整窗重写降到最少。

先读第 1 节理解计价，照第 2 节把前缀理顺，再按第 5 节在换窗 / 保温 / 混合里选一种，第 6 节每个坑都要自测一遍，最后用第 8 节验收。

---

## 1. 计价模型

记基础输入价为 1：

| 项 | 倍率 | 说明 |
|---|---|---|
| 缓存读 | 0.1 | 命中的前缀 |
| 缓存写（1h） | 2.0 | 订阅登录下 Claude Code 默认用 1h 缓存 |
| 缓存写（5m） | 1.25 | API key 默认是 5m；要 1h 须设 `ENABLE_PROMPT_CACHING_1H=1` |
| 未缓存输入 | 1.0 | 一般很小 |

- TTL 1 小时，**每读一次刷新一次**。
- 缓存按前缀匹配：从第一个不同的字节往后全部重写。
- 缓存按**账号**隔离：换账号 = 冷缓存。

一轮成本近似：`cost ≈ 0.1 × 命中前缀 + 2.0 × 新写入 + 1.0 × 未缓存`。

设窗大小 W、新窗大小 S、离开时长 T、回来后再聊 N 轮（每轮新增量忽略）：

```
T < 60min:   什么都不做 = 保温 = 换窗 = N × 0.1W
T ≥ 60min:
  什么都不做 = 2W + (N-1) × 0.1W
  保温      = k × 0.1W + N × 0.1W            # k = floor(T/50)；不设上限时 k 不封顶
             （设了上限 K_MAX 且 T - K_MAX×50 ≥ 60：到上限放手后照样冷，再加 2W - 0.1W）
  换窗      = 0.1W（写窗影读一次旧窗）+ 2S + (N-1) × 0.1S
```

结论：T 略超 1h 且 N 小 → 保温最省；T 长或 N 大或 W 大 → 换窗最省。

---

## 2. 前提：让前缀稳住（必须先做）

每条都带自测。用量账字段见第 7 节。

**R1 会变的放末尾。** 时间、身体读数、召回片段、面板状态等每轮会变的内容，追加在本轮 user 消息末尾，绝不进 system prompt 或 CLAUDE.md。
自测：连续 5 轮纯聊天，每轮 `cache_read / (cache_read + cache_creation + input) ≥ 0.97`。

**R2 窗开着不改开窗就读的文件。** CLAUDE.md、auto memory 的 MEMORY.md、`--system-prompt-file` 指向的文件、MCP 工具清单，在有活窗时不改；改动攒到换窗或夜间无窗时。
自测：改动后若有活窗，下一轮会出现大块 `cache_creation`；应为 0 次。

**R3 提示词快照保持默认开。** 若"开窗包"（近期原话、唤醒记忆、输出规则）只随首轮 system 发送、resume 时只传静态 system，快照正是保住首轮那份的机制；关掉后 resume 会丢开窗包。只有"热更新提示词 = 杀进程 + resume"的设计才需要关，且关了要接受整窗重写。
自测：resume 后问一个只写在开窗包里的事实，答得出。

**R4 不做"每轮改前缀"的优化。** 例如只清旧轮思考块：API 按清理前查缓存、按清理后存，从第一个被清掉的块起永远对不上，成本涨一个数量级。若必须剥，须在请求出口每轮剥得完全一致（需锁 CLI 版本）。
自测：长窗里每轮 `cache_creation` 应约等于本轮新增量，不随窗长增长。

**R5 控制窗增长源。** 统计每轮增量来源：纯聊天约 0.7k/轮；工具返回才是大头（实测工具返回字符数 > 双方对话 + 思考之和）。大输出走沙盒/摘要工具，只回结论；重活放到别的会话。
自测：按工具类型汇总返回字符数，Bash/Read 不应占窗增长的大头。

**R6 大窗冷读超时给够。** 首字超时（零输出超时）过短会在冷读途中杀进程，重试又从头冷读，形成死循环。27 万级窗口建议 ≥ 600s；重试只对"一个字都没吐"的轮做，吐过字的不重发（防说两遍）。
自测：人为让缓存过期后发一句，首字时间 < 超时，且之后各轮恢复秒回。

---

## 3. 换窗实现

### 3.1 触发条件（任一满足，在两轮之间判断，绝不打断正在跑的轮）

| 触发 | 建议值 | 说明 |
|---|---|---|
| 水位线 | 400k context tokens | 用上一轮 `context_tokens`；留足余量，避开自动压缩 |
| 空闲 | 55 min | 必须 < TTL，才能在缓存还热时写窗影（写影本身读一次整窗，0.1W） |
| 换账号 / 换线路 | 立即（下一轮前） | 见坑 P1、P2 |
| 换模型 | 立即 | 模型不同缓存不通 |
| 可选：日界 | 例如凌晨某点 | 让"一天"对齐一扇窗 |

记录每次换窗的 `reason`（watermark / idle / route_switched / model_switched / ...）进用量账。

### 3.2 写窗影

在旧窗缓存还热时，用 fork 读旧窗、写摘要，不占旧进程的轮：

```
claude -p --resume <old_session_id> --fork-session --model <旧窗模型> \
  <写影提示词>         # 环境变量必须是旧窗所在账号/线路的那套，见 P3
```

窗影建议三节：她怎么样、我怎么样、这一窗发生了什么（只写两人之间的事，不列工作待办）。设字数上限和超时；失败按"无窗影"处理，不挡换窗。设最少轮数门槛（如 3 轮新话），不足不写。同一段新话只写一次（记"已认领到第几轮"）。

### 3.3 新窗首轮带什么

- 静态身份 system（不变部分）
- 窗影（上一窗）
- 置顶记忆
- 念头池：想做的事、在聊的话题
- 一段近期原话（预算随是否有窗影调整：有影可少给）

只在首轮送；之后各轮只送末尾增量（R1）。

### 3.4 伪代码

```python
def before_turn(key, req):
    if route_or_account_changed(key) or model_changed(key):
        rollover(key, reason="route_switched" | "model_switched")
    elif idle_minutes(key) >= IDLE_ROLLOVER_MIN or last_context_tokens(key) >= WATERMARK:
        rollover(key, reason="idle" | "watermark")

def rollover(key, reason):
    shadow = write_shadow_via_fork(key, env=env_of_window(key))  # 失败返回 None
    end_session(key)              # 不 resume 旧会话
    mark_first_turn(key, payload=[static_system, shadow, pins, thought_pool, recent_raw])
    usage_log(key, rollover=reason)
```

---

## 4. 保温实现

### 4.1 参数

| 参数 | 建议 | 说明 |
|---|---|---|
| IDLE_MS | 50 min | 小于 TTL，留出 ping 往返余量 |
| CHECK_MS | 5 min | 巡检周期 |
| MAX | 不限，或按需设 | 朋友家实际不设上限，一直保温；想控制成本可设（如 8 ≈ 7.5h），到上限后接换窗更好 |
| TIMEOUT_MS | 90 s | ping 等不到 result 就摘监听、清标记 |
| 空闲回收（可选） | 只在设了 MAX 时用，如 12 h | 不设上限一直保温的，不回收保温中的窗；设了上限的，按聊天文件 mtime 回收已凉的进程（不用进程内时间戳），启动后 10 分钟先跑一次兜底 |

### 4.2 逻辑

```
每 CHECK_MS：
  对每个活进程：
    跳过：已死 / 正在生成 / ping 在飞 / 不在保温名单
    若 now - last_touch < IDLE_MS：跳过
    若设了 MAX 且 ping_count >= MAX：跳过（或改为写窗影换窗）
    ping_count += 1; last_touch = now; ping_in_flight = True
    往 stdin 写一条 user 消息（system-reminder 包裹）：
      "缓存保活（系统自动，对方看不到）：只回复一个「.」，不要思考，不要调用工具。"
    挂监听：收到 result → ping_in_flight = False，摘监听
    90s 超时 → 同样清理

真消息开始时：
  last_touch = now; ping_count = 0
  若 ping_in_flight：先等它的 result（或超时）再写真消息
```

- ping 回复不进对话历史、不推送；可在界面落一张极小的"保温"卡片，重建历史时过滤掉。
- 只给值得保的窗保温（例如开了"主动联系"的窗）。
- 不设上限时，一夜约 10 发、每发 0.1W；窗很大时考虑配合水位线换窗，别让一扇巨窗一直被保着。

---

## 5. 选哪种

| 使用模式 | 方案 |
|---|---|
| 间隔常在 1h 上下、回来只说几句（如工作间隙） | 保温 |
| 一离开就是半天/一夜，回来长聊 | 换窗 |
| Resident 自己有定时自主活动（写日记、整理、阅读等），且跑在同一会话 | 换窗；自主轮天然在读缓存，等同免费保温 |
| 不确定 | 混合：保温 MAX=2~3，到上限仍无人则写窗影换窗 |

自主活动能否"暖窗"的前提：它必须跑在**同一个 session、同一个账号**上。另起进程/另一账号跑的自主任务不暖这扇窗。
自测：在用量账里找两次人类消息间隔 > 60min 的区间，中间若有自主轮且相邻间隔都 < 60min，人类回来那一轮命中应 ≈ 100%。

---

## 6. 坑与自测

**P1 开机定线 ≠ 换线。**
场景：进程启动时先用默认环境，再由路由器把线路切到持久化的那条。若把这次切换当成"换线路"，会用默认环境去 fork 旧窗写影——默认账号若与窗所在账号不同，整窗冷写，并产生一份假窗影；而窗本身随后照常 resume。
规则：启动后第一次设定线路只记录，不派写影、不换窗；之后的切换才算换线。
自测：重启后 2 分钟内，用量账不应出现 `rollover=route_switched` 且 `cache_read=0` 的行；日志无 switch-shadow。单测：已有会话 → 首次 apply_route → 断言未调用 fork、下一轮无 rollover；再 apply_route → 断言派影、下一轮 rollover。

**P2 换账号必须换窗，包括跨重启。**
场景：用户切了订阅号同时进程重启，重启后"上一窗在哪条线"已丢失，被当成当前线，于是旧窗 resume 到新账号上冷算。
规则：每个窗所在的线路/账号要持久化（与 session_id 一起存），启动后比较；不一致则首轮换窗。
自测：切号 + 重启后的第一轮应 `first_turn=true`、窗变小，而不是 resume 后整窗 `cache_creation`。

**P3 写影 fork 的环境必须是旧窗那套。**
fork 用错账号 = 在一个没缓存的号上冷读整窗。优先用翻牌瞬间的旧 config，其次旧进程 spawn 时的 config，再次按线路标签查环境底账。
自测：写影那行用量 `cache_read ≈ W`。

**P4 R2 的反例：重启时重读了被改过的说明书。** 重启后 resume，若期间 CLAUDE.md / MEMORY.md 被改，前缀变了。把这类改动安排在换窗时。

**P5 ping 与真消息交叠。** 见 4.2：真消息要等 ping 的 result；否则本轮守门逻辑会把 ping 的 `message_start` 当成本轮的。

**P6 空闲回收杀进程 ≠ 换窗。** 只杀进程、下次 `--resume` 同一会话：若已超 TTL 就是整窗冷写。回收前要么已换窗（写影 + 新窗），要么接受冷写。不要让"回收时间 > TTL 且不写影"成为默认路径。

**P7 自动压缩。** 水位线换窗应低于自动压缩触发点；若不关自动压缩，至少监听 `compact_boundary` 并让用户看得见，陪伴场景不应悄悄把原话嚼成摘要。

---

## 7. 用量账（每轮一行 JSONL）

至少记：

```
at, key, session_turn, first_turn, rollover, model,
context_tokens, input_tokens, cache_read_input_tokens, cache_creation_input_tokens,
output_tokens, turn_api_calls, shadow(bool)
```

常用查询：
- 命中率 = `cache_read / (cache_read + cache_creation + input)`
- 冷写事件 = `cache_read == 0 and cache_creation > 50k`，按 `rollover` 分组
- 自主轮暖窗 = 人类消息间隔 > 60min 的区间内是否有轮次、相邻间隔是否都 < 60min

---

## 8. 验收清单

- [ ] 确认是 1h 缓存（转录里写入类型为 1h；API key 已设 `ENABLE_PROMPT_CACHING_1H=1`）
- [ ] 连续纯聊天命中率 ≥ 97%（R1）
- [ ] 有活窗时没有改开窗文件的流程（R2）
- [ ] 快照策略与开窗包设计一致（R3）
- [ ] 窗增长按来源统计过，工具返回已收敛（R5）
- [ ] 首字超时 ≥ 大窗冷读时间，重试只对零输出轮（R6）
- [ ] 换窗触发、写影、新窗首轮内容都有日志和 `rollover` 记账（第 3 节）
- [ ] 若用保温：参数、上限、交叠处理、超时、卡片过滤齐全（第 4 节）
- [ ] P1 重启后无冷写假换线；P2 切号跨重启会换窗；P3 写影命中 ≈ W
- [ ] 跑一周后按 `rollover` 汇总冷写 token，最大头应是预期内的换窗首轮
