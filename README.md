# lark-sheet2base

**傻瓜式**把飞书电子表格(Sheets)同步为多维表格(Base),并可按你给的时间定时自动刷新。

下载后,你只需要说三句话:

> 源数据表:`https://.../sheets/xxx`
> 目标多维表格:`https://.../base/yyy`(不填就自动新建)
> 更新时间:`每天上午 9 点`

其余全自动:探表结构 → 建表/字段 → 自动定主键 → 幂等写入 → 定时。

## 前提(一次性,约 5 分钟)

1. **装 lark-cli**(飞书/Lark 官方 CLI):
   ```bash
   # 见 https://github.com/larksuite/cli,或按你平台的安装方式
   ```
2. **准备一个自己的飞书/Lark 应用**,开通权限:
   - `sheets`(电子表格读写)
   - `bitable` / `base`(多维表格读写)
3. **登录一次**(之后不用再管):
   ```bash
   lark-cli auth login --domain feishu   # 国内飞书 feishu.cn
   lark-cli auth login --domain lark     # 国际版 larksuite.com
   ```

## 安装

```bash
# 方式一:HclawHub 一键装
hub skills install lark-sheet2base -o .claude/skills/

# 方式二:直接放目录
git clone <你的仓库> .claude/skills/lark-sheet2base/
```

## 使用

对 AI 助手说(三句话即可):
```
把这份电子表格同步成多维表格:
源数据表 https://xxx.feishu.cn/sheets/<token>
目标多维表格(不填则新建)
更新时间:每天 9 点和 18 点
```

回答它可能追问的唯一一个问题(如果列有歧义):「用哪一列做主键?」

## 目录结构

```
lark-sheet2base/
├── SKILL.md      # 核心:name + description + 全自动流程
├── README.md
└── .gitignore
```

## 发布

```bash
# GitHub(源码)
git init && git add . && git commit -m "init"
git remote add origin <仓库地址> && git push -u origin main

# 平台(HclawHub)
zip -r lark-sheet2base.zip . -x '*.git*'
hub skill publish lark-sheet2base.zip
```

## 兼容性说明

- **国内飞书 / 国际 Lark**:按 URL 域名自动判,登录时选对 `--domain` 即可。
- **定时**:在有 `hclaw cron` 的平台上用 `hclaw cron`;纯本地环境用系统 crontab 周期触发同一段同步指令。
- **身份**:全程 `--as user`,不会越权操作别人的数据。
