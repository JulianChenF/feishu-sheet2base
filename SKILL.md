---
name: lark-sheet2base
version: 1.0.0
description: "傻瓜式把飞书电子表格(Sheets)同步为多维表格(Base),可定时自动刷新。当用户只需要给出「源数据表 + 目标多维表格 + 更新时间」三样信息、其余全自动(自动探结构/建表/映射字段/幂等写入/定时)时使用。触发词:同步、转多维表格、电子表格转 Base、定时刷新、傻瓜同步。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# Sheet → Base 傻瓜式同步

用户只需说三句话,其余全部由本 skill 自动完成:
1. **源数据表**:电子表格 URL(或 spreadsheet_token + 子表名)。
2. **目标多维表格**:Base URL(或 base_token;不填则自动新建一个 Base)。
3. **更新时间**:每天 N 点 / 每 N 小时(`0 9,18 * * *` / `1h`)等。

## 一次性准备(仅在首次,之后不用再管)

**身份**:全程 `--as user`,以调用者本人身份操作,各人动各人的数据。

**登录自己的应用**(不要用公司共享 app,任何人生效):
```bash
lark-cli auth login --domain feishu   # 国内飞书(feishu.cn 链接)
lark-cli auth login --domain lark     # 国际版(larksuite.com 链接)
```
> 域名判别:URL 含 `larksuite.com` → lark;含 `feishu.cn` → feishu。权限需含 `sheets`(电子表格)与 `base`/`bitable`(多维表格)。

**验证登录是否可用**(读一条即可):
```bash
lark-cli sheets +workbook-info --spreadsheet-token <t> --as user
```

## 主流程(接到三条输入后执行)

### Step 0 — 解析三条输入
- 源:URL 里抽取 token(`/sheets/<token>`)或直接用 `--spreadsheet-token`;没给子表名先 `+workbook-info` 拿 `sheets[]`。
- 目标:URL 抽取 `/base/<token>` 或 `--base-token`;没给则 `+base-create` 新建。
- 时间:把自然语言转成 `--cron "分 时 日 月 周"`(国内加 `--tz Asia/Shanghai`)或 `--every 1h`。

### Step 1 — 探源表结构(自动识别表头与数据范围)
```bash
lark-cli sheets +workbook-info --spreadsheet-token <t> --as user \
  --jq '.data.sheets[] | {sheet_id,title}'
lark-cli sheets +csv-get --spreadsheet-token <t> --sheet-id <sid> --range "A1:last" --as user \
  --jq '.data.annotated_csv'
```
- 自动找**表头行**:第一行出现多个非空、非纯数字的文本即视为表头。
- 自动找**数据末行**:读到连续空行为止(读全再写,别只探前 N 行)。

### Step 2 — 确保目标表与字段存在(自动建)
```bash
lark-cli base +table-list --base-token <b> --as user
lark-cli base +field-create --base-token <b> --table-id <tid> \
  --json '{"type":"number","name":"数量","style":{"type":"plain","precision":0}}' --as user
```
目标表名默认取源子表名;没有对应表就 `+table-create` 新建,再按字段映射 `+field-create`。

### Step 3 — 自动映射字段(类型推断)
按列数据推断字段类型,写进字段 JSON:
| 数据特征 | 字段类型 |
|---|---|
| 纯整数/小数 | `number`(百分比列一样是 number,展示用 `style.percentage`) |
| 年月日/时间 | `datetime` |
| 其余(含编号 `001`、带前缀的标识) | `text` |

**主键自动挑**:优先第一个 `text` 列做主键;若多个候选拿不准,**只问用户一句**"用哪列做主键",不瞎猜。

### Step 4 — 幂等写入(重复跑不重复插行)
1. 读目标现有记录、建「主键值 → record_id」映射:
```bash
lark-cli base +record-list --base-token <b> --table-id <tid> --format json --as user
```
2. 逐条比对:
   - 主键无 → `+record-batch-create` 新增(单批 ≤ 200 条,超了分页写)
   - 主键有且值变 → `+record-batch-update`(`{"update_records":{"<record_id>":{字段:值}}}`)
   - 没变 → 跳过
> `+record-upsert` 不会按业务键查重,查重必须显式做。

### Step 5 — 回读校验
```bash
lark-cli base +record-list --base-token <b> --table-id <tid> --format json --as user
```
写返回 ok 只代表请求被接受;回读确认条数/首末行值后再交付。

### Step 6 — 注册定时更新
把「已解析好的全部细节(源 token/sheet/范围、目标 base/table、字段映射、主键)」写成一段**自包含同步指令**,再挂定时:
```bash
hclaw cron add --name "sheet2base同步" --prompt "<自包含指令>" \
  --cron "<用户给的更新时间>" --tz Asia/Shanghai
```
- 若环境没有 `hclaw cron`:改用系统 crontab,把同一条指令交给 agent 周期触发,或提供 `scripts/sync.sh` 手动跑。

## 关键约束(踩过的坑,必须遵守)
- 读 JSON 只解析 stdout,不要 `2>&1`(stderr 混入会解析失败)。
- `+record-list` 必须 `--format json`,否则默认返回 markdown 表。
- `formula` 字段只读,写记录时不要提交。
- `select` 字段只能写已有选项;写前 `+field-list` 确认选项名。
- 量值数据写数字,不要写成 `"40%"` 字符串(会落成文本、丢失计算)。
- 建 Base 仪表盘时,组件**串行**创建;`+dashboard-arrange` 会清掉 text 块,text 块要最后建。

## 权限
| 操作 | 所需能力 |
|------|-----------|
| 读电子表格 | `--as user`(sheets) |
| 建/写 Base、建仪表盘 | `--as user`(bitable/base) |