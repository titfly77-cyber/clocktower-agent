# 事件协议 v0.1

目标：让 App 的人工操作、音频推断和总结记录有明确来源，并能处理重复、迟到与修正。本文是实现约定，未绑定具体后端或通信方式。

## 1. 状态编码

流程状态：`SETUP`、`NIGHT`、`DAWN`、`DISCUSSION`、`NOMINATION`、`VOTING`、`DAY_END`、`POSTGAME`。

氛围标签：`RELAXED`、`FOCUSED`、`CONTENTIOUS`、`TENSE`、`EXCITED`、`CONFUSED`、`DISENGAGED`、`FRICTION`。

氛围模式：`auto`、`manual`。强度与模型置信度分别使用 `low`、`medium`、`high`；缺少对应判断时为 `null`。

## 2. 事件公共字段

| 字段 | 含义 |
| --- | --- |
| schema_version | 固定为 `0.1`。 |
| event_id | 本局内唯一事件 ID；用于去重与引用。 |
| session_id | 本局 ID。 |
| sequence | 接收并提交事件时分配的单调递增序号。 |
| type | 事件类型。 |
| occurred_at | 发生时间，含时区的 ISO 8601 字符串。 |
| received_at | 接收时间，与发生时间分开。 |
| source | `app_dm`、`audio_model` 或 `system`。 |
| payload | 具体事件内容。 |

按接收顺序归档，按事件来源与版本规则更新状态。音频分析即使晚到，也可以挂回原音频时间范围，但不能因此覆盖之后的手动操作。重传事件维持原 `event_id`，不重复提交。

## 3. 核心事件

### session.started

来源：`app_dm`。创建新局，将阶段设为 SETUP，phase_revision=0，day=0，night=0，氛围模式设为 auto，当前氛围为空。

字段：`title`、可为空的 `script_name`。

### phase.suggested

来源：`audio_model`。不改变实际阶段。

字段：`proposed_phase`、`base_phase_revision`、`confidence`、`reason`、`evidence_start_ms`、`evidence_end_ms`。证据时间相对本局音频起点。

只有 `base_phase_revision` 等于当前修订号时，才可作为可采纳建议显示；否则只保留为历史分析。

### phase.changed

来源：`app_dm`。实际切换阶段，也用于撤销／修正。

字段：`from`、`to`、`day`、`night`、`base_phase_revision`、`phase_revision`、`reason`，可选 `accepted_suggestion_id` 和 `corrects_event_id`。

- 当前状态与 `from` 及 `base_phase_revision` 必须匹配；不匹配时让 App 重新取得状态，不盲目写入。
- 新修订号为旧修订号加一。
- `reason` 使用 `manual_selection`、`accept_suggestion` 或 `correction`。
- 撤销是一次新的修正事件，恢复目标阶段与轮次，修订号继续增加。
- 采纳建议仍是一次 DM 操作，引用建议 ID；不能由模型伪造此事件。
- 同阶段允许修正轮次。普通重复点击在状态已更新后不再次生效。

### atmosphere.estimated

来源：`audio_model`。

字段：`base_phase_revision`、`primary`、`secondary`、`intensity`、`confidence`、`status`、`reason`、`evidence_start_ms`、`evidence_end_ms`。

- `status` 为 `valid` 或 `insufficient_evidence`。
- `valid` 时主标签、强度及置信度必须存在；次标签可为空，且不得与主标签相同。
- `insufficient_evidence` 时标签、强度及置信度均为空；`reason` 说明不足原因。
- 手动模式下仅归档，不改变有效氛围。
- 与当前阶段修订号不匹配的估计仅归档。
- 自动模式下，以证据窗口的新旧决定更新次序；迟到的旧窗口不能替换较新的结果。

### atmosphere.overridden

来源：`app_dm`。

字段：`primary`、`intensity`、可选 `note`。切换至 manual 模式，有效氛围采用此事件。人工标签不带模型置信度，次标签清空。

手动状态持续至 atmosphere.auto_restored，不因阶段变化、模型估计或超时自动解除。

### atmosphere.auto_restored

来源：`app_dm`。

字段：`reason`。切换至 auto 模式，清空当前有效标签，等待新估计。

新估计必须针对当前阶段，且证据窗口起点不早于此次恢复操作在音频时间线上的位置。恢复之前的录音即使恢复后才分析完成，也不作为恢复后的当前判断。

## 4. 派生状态

至少维护以下字段：

- `phase`、`phase_revision`、`day`、`night`。
- `atmosphere_mode`。
- `effective_atmosphere`：主／次标签、强度、来源、对应事件 ID 与更新时间；可为空。
- `latest_model_estimate`：与有效人工标签分开保存。
- `pending_phase_suggestions`：只包含当前修订号下的可采纳建议。

进入新阶段且处于自动模式时，上一阶段的估计保留为历史，当前氛围等待新估计；处于手动模式时保持人工标签，并显示原标记时间。

## 5. 转写、记录与总结对象

以下是下一步接入音频和总结服务时的最低字段约定，不假设这些模块已经实现。

### 转写片段

`segment_id`、`start_ms`、`end_ms`、`text`、`speaker_ref`、`speaker_mapping_status`、`transcription_status`、`visibility`。

默认 `visibility=dm_only`。匿名说话人标识不自动映射玩家座位，识别到句子中出现的座位号也不改变映射。

### 结构化记录

`record_id`、`kind`、`content`、`evidence_ids`、`confirmation_status`、`visibility`、`confirmed_by`、`revision`。

- `kind` 区分 `player_claim`、`announced_event` 和 `postgame_revelation`。
- `confirmation_status` 区分 `unconfirmed` 和 `dm_confirmed`。
- `visibility` 区分 `dm_only` 和 `public_confirmed`。
- DM 确认一段玩家说法的记录准确且可公开，不会把其类别从 claim 变为真实角色事实。

### 总结

`summary_id`、`session_id`、`scope`、`audience`、`source_record_ids`、`source_revisions`、`version`、`status`、`generated_at`、`content`。

- `scope` 区分当前阶段、某一天和整局。
- `audience` 区分 DM 与玩家。
- 玩家版只能从已经核对并标记可公开的记录生成；生成输入就应过滤，而非生成后再尝试删去秘密。
- 依赖记录被修正后，`status` 标记需要更新，保留旧版本。
- 重复收到相同阶段事件，不重复创建相同输入版本的自动总结。

## 6. 示例范围

`examples/session-events.json` 全部是虚构演示，包含：

1. 从开局到第一次投票的手动阶段变化。
2. 一个阶段建议到达时，其依据的状态已经过期。
3. 手动标记氛围后，新的模型估计不覆盖人工选择。
4. 在投票结束后回到白天交流。
5. 恢复自动后等待新音频窗口，并更新有效氛围。

文件末尾的 `expected_projection` 是预期结果，不是实际模型推理或已运行程序的输出。
