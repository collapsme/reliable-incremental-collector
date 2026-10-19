# Reliable Incremental Collector

Language-agnostic Codex skill for designing and implementing collectors that repeatedly read growing sources and must deduplicate records, resume interrupted runs, and publish validated outputs safely.

## What it covers

- Stable record identity, cursor and overlap semantics, pagination guarantees, and source-retention gaps
- Durable batches, pending-run recovery, idempotent replay, and success-cursor boundaries
- Bounded retries and concurrency, explicit skip conditions, and optional enrichment failures
- Safe replacement of files and preservation of manually edited spreadsheet fields
- Privacy-aware handling of authenticated browser sessions and external model calls

It is intended for new collectors or substantial extensions, not one-off exports or generic scraping without incremental state. It gives implementation guidance rather than a source-specific adapter or reusable collection framework.

## Install

Copy or clone this repository into your Codex skills directory. Run these commands from the directory containing the cloned repository. For example, on Windows PowerShell:

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.codex\skills" | Out-Null
Copy-Item -Recurse .\reliable-incremental-collector "$env:USERPROFILE\.codex\skills\"
```

On macOS or Linux:

```sh
mkdir -p ~/.codex/skills
cp -R reliable-incremental-collector ~/.codex/skills/
```

Restart Codex if the skill does not appear. It can be invoked as `$reliable-incremental-collector` or discovered automatically for matching tasks.

## Example requests

- “Add a daily order collector that resumes after failure and does not duplicate late-arriving records.”
- “Harden this browser-session collector so a failed workbook write is recovered before the next scan.”
- “Refresh source fields in this workbook while preserving manually maintained notes by record ID.”

## Validation

The skill contains instructions and a conditional reference; it has no runtime dependencies. Validate its metadata and structure with the Codex skill creator's `quick_validate.py`, then review behavior against at least one source API, authenticated-session, and editable-workbook scenario. The acceptance cases in `SKILL.md` and `references/recovery-patterns.md` describe expected outcomes.

## License

MIT. See [LICENSE](LICENSE).

---

# Reliable Incremental Collector（中文）

这是一个语言无关的 Codex skill，用于设计和实现持续读取新增数据的采集器，重点处理稳定去重、运行中断恢复以及安全发布经过校验的输出。

## 能力范围

- 稳定记录身份、游标与重叠窗口、分页排序保证及来源保留期
- 持久化分批、待完成批次恢复、幂等重放与成功游标边界
- 有界重试与并发、明确跳过条件及可选数据加工失败
- 安全替换文件，以及保留电子表格中的人工编辑字段
- 以隐私为先处理已登录浏览器会话和外部模型调用

适用于新建采集器或较大扩展，不适用于一次性导出或没有增量状态的通用网页抓取。该 skill 提供实现指导，不包含特定网站适配器或通用采集框架。

## 安装

将本仓库复制到 Codex skills 目录。Windows PowerShell 示例：

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.codex\skills" | Out-Null
Copy-Item -Recurse .\reliable-incremental-collector "$env:USERPROFILE\.codex\skills\"
```

macOS 或 Linux 示例：

```sh
mkdir -p ~/.codex/skills
cp -R reliable-incremental-collector ~/.codex/skills/
```

如果 skill 未显示，请重启 Codex。可通过 `$reliable-incremental-collector` 显式调用，也可由 Codex 自动发现匹配任务。

## 示例请求

- “每天增量采集订单，处理失败后能恢复，也不能漏掉迟到的数据。”
- “加固这个浏览器登录态采集器，工作簿写入失败后，下次运行要先恢复。”
- “刷新工作簿里的来源字段，同时按记录 ID 保留人工维护的备注。”

## 验证

该 skill 由指令和按需读取的参考文档组成，没有运行时依赖。使用 Codex skill creator 的 `quick_validate.py` 检查元数据和目录结构，并分别检查 API 采集、登录态采集、人工编辑工作簿三个场景。验收行为见 `SKILL.md` 与 `references/recovery-patterns.md`。

## 许可证

MIT，详见 [LICENSE](LICENSE)。
