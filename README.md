# 溟月的工坊 (MoonWorkshop)

> 给 AstrBot 里的 AI 娘一双只在自己地盘上动手的手。

AstrBot 插件。为 AI 提供受限的文件读写、写前备份、语法与 JSON 校验、失败回滚、Docker 沙箱测试，以及「暂存 → 沙箱 → 主人批准 → 上线」的发布流水线。

## 工具

| 工具 | 说明 |
| --- | --- |
| `ws_list_files` | 列出允许目录下的文件 |
| `ws_read_file` | 读取允许目录下的文件 |
| `ws_write_file` | 写入文件（写前自动备份；开启暂存模式时先写到 staging） |
| `ws_rollback` | 回滚到最近一次备份（`reason` 含「演练」时记为演练） |
| `ws_check_syntax` | 对 `.py` / `.json` 做语法检查 |
| `ws_sandbox_test` | 把插件目录打成 tar 交给沙箱容器跑编译 + JSON 校验 + pytest |
| `ws_stage_status` | 查看暂存区各插件草稿与批准状态 |
| `ws_promote` | 沙箱 → 批准 → 时效校验通过后，把草稿上线到插件目录 |
| `ws_reload_plugin` | 重载 AstrBot 插件 |

## 安全边界

- 可写：`/AstrBot/data/plugins/`、工坊暂存区 `staging/`、工位 `workbench/`
- 可读：以上 + 工坊数据目录（`notes.md`、`backups/`）
- 仅允许 `.py` / `.json` / `.md` / `.txt`，单文件默认上限 2 MB
- 写前备份到 `backups/<插件桶>/<时间戳>/`，并写 `manifest.jsonl`（sha256、size）；每桶保留 20 份
- 语法检查失败自动还原，且不占生产目录
- 每 600 秒窗口最多写 3 个文件
- 禁止改工坊自身、`astrbot_plugin_moon_gate`、`cmd_config.json`、`data_v4.db`、`.env`
- 每次写 / 回滚 / 重载都追加到 `notes.md`，并记 `events.jsonl` 账本供考核插件计分
- 仅管理员可用

## 暂存审批流水线

1. AI 在 staging 里写草稿（生产目录不动）
2. `ws_sandbox_test` 跑沙箱（无网络、只读根、512 MB 内存、1 CPU、降权、用完即毁）
3. 主人 `/工坊 批准 <插件>`
4. `ws_promote` 校验「沙箱晚于最后改动、批准晚于最后改动、批准未过 30 分钟」后上线，逐文件备份

命令：`/工坊 状态`、`/工坊 批准 <插件>`、`/工坊 拒绝 <插件>`。

## 安装

1. 将本目录放入 `/AstrBot/data/plugins/astrbot_plugin_moon_workshop/`。
2. 重启 AstrBot。
3. 在插件配置里设置管理员；需要暂存流水线时把 `staging_required` 设为 `true`。

## 沙箱 runner（可选）

`moon_sandbox/runner.py` 监听 `172.17.0.1:8791`，把条目打包进 `moon-sandbox:1` 镜像执行（`--network=none --read-only --memory 512m --pids-limit 128 --cpus 1 --cap-drop ALL`）。不部署 runner 时，`ws_sandbox_test` 会直接报错，其余功能不受影响。

## 相关仓库

- [溟汐的考核](https://github.com/seek9527-open/astrbot_plugin_moon_gate)：消费本插件的事件账本做积分与提权。

## 许可

MIT，作者「溟月和主人」。
