# 精投看板

看板是追踪表：每个岗位一条记录，页面实时订阅，Claude 写进去马上显示；用户在页面上改状态、写不投原因、加备注、导入自动投递记录，Claude 也能读到。

- 链接：https://claude.ai/artifact/CEkH5tUewLKw4r4urY8CpV（私有，只有主人能打开）
- 读写：`ArtifactData` 工具，`url` 填上面的链接。写之前先读，写的时候带上 `if_version`。
- 找不到链接时：`Artifact` 工具 `action: "list"`，找标题「精投看板」。

## `jobs/<id>`：每个岗位一条

| 字段 | 说明 |
|---|---|
| `company` `title` `location` `link` | 基本信息 |
| `source` | 来源：检索渠道 / 自己添加 / 自动投递 tracker.csv |
| `origin` | `自动投递` 表示来自自动投递系统，卡片上会显示 |
| `jobId` `grade` | 自动投递系统的岗位编号和 A / B / C 等级 |
| `found` | 发现日期 `YYYY-MM-DD` |
| `status` | `candidate` `applied` `interview` `offer` `closed` `skipped` |
| `stage` | interview：轮次；offer：说明；closed：被拒 / 我放弃 / 岗位关闭 / 无回复 |
| `verdict` | 只用于 candidate：`必投` `可投` `边缘` |
| `fit` `outlook` | 内容匹配度、前景，1–5 |
| `why` `outlookNote` `concerns` | 打分理由、前景依据、顾虑和缺口 |
| `salary` `wfh` `jdStatus` | 薪资、居家办公、JD 是否读全 |
| `applied` | 投递日期 |
| `next` `nextDate` | 下一步和日期。看板顶部「接下来」按 `nextDate` 排序 |
| `skipReason` | 不投的原因 |
| `notes` | 用户备注 |
| `timeline` | `[{date, text}]`，只追加 |
| `sources` | `[{title, url}]` |
| `updated` | ISO 时间，每次写都更新 |

**id 规则**

- Claude 检索到的：`YYYY-MM-DD-公司-职位`，英文短名，小写，用连字符。
- 自动投递系统的：`auto-` + 岗位编号（小写，非字母数字换成 `-`）。看板的「导入投递记录」用同一个规则，所以同一个岗位写几次都只有一条。

**状态只往前推**：candidate → applied → interview → offer → closed。看板上已经推进的状态不要改回去；`skipped` 不自动改。

## `jobs/<id>/docs/<name>`：岗位材料

`{title, markdown, updated}`。`name` 用 `jd`、`cover-letter`、`salary`、`emails`、`interview-prep-1`、`debrief-1`……看板详情的「材料」按这个顺序显示。岗位下线后，JD 在这里还能看。

## `meta/profile`：看板顶部「我的标准」摘要

`{positioning, hard[], directions[], exclude[], updated}`。`criteria.md` 改了之后同步更新。

## 读回用户在看板上的操作

每次开始和求职有关的工作时，先 `list` 一遍 `jobs`：

- 新出现的 `skipped` 且带 `skipReason` 的：用 `job-criteria` 把原因写进 `criteria.md` 的「从反馈中学到的」，然后在这条的 `timeline` 追加「已写入标准」，避免重复处理。
- `source` 是「自己添加」、还没有 `fit` 的：读 JD、打分，补上 `verdict` `fit` `outlook` `why` `outlookNote` `concerns`。
- `origin` 是「自动投递」的：它们已经投过，检索时要去重。

## 和自动投递系统结合

两种方式，可以同时用：

1. **手动导入**：在看板上点「导入投递记录」，选自动投递系统的 `tracker.csv`（或者粘贴内容）。已投、跳过、面试、被拒的记录会合并进看板。
2. **自动同步**：在自动投递技能里加下面这一步，每提交或跳过一个岗位就写进看板。

```markdown
## 同步到精投看板
每次更新 tracker.csv 之后，用 ArtifactData 把这个岗位写进看板
https://claude.ai/artifact/CEkH5tUewLKw4r4urY8CpV 的 `jobs` 集合：
- doc_id：`auto-` + 岗位编号（小写，非字母数字换成 `-`）。
- 先 get。不存在就 set：{company, title, location, link, jobId, grade,
  origin: "自动投递", source: "自动投递 tracker.csv", status, stage: "",
  applied, found: applied, next: "还没消息就跟进", nextDate: follow_up_on,
  timeline: [{date, text: "自动投递：已提交"}], updated: 现在的 ISO 时间}。
  跳过的岗位 status 写 "skipped"，skipReason 写原因。
- 已存在就 update（带 if_version）：只把 status 往前推
  （candidate → applied → interview → offer → closed），timeline 追加一条，
  不要把看板上已经推进的状态改回去。
- 没有 ArtifactData 工具时跳过这一步，提醒在看板上点「导入投递记录」。
```
