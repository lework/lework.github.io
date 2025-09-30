---
layout: post
title: '清理 Dify 应用日志'
date: '2025-05-22 20:00'
category: ai
tags: ai dify
author: lework
---
* content
{:toc}

{% raw %}

[Dify](https://dify.ai/) 是一款强大的开源大型语言模型（LLM）应用开发与运营平台。在运行过程中，Dify 会产生各类日志数据，默认情况下，Dify 系统本身不提供自动清理这些日志的功能。随着时间的推移，累积的日志可能会占用大量的磁盘存储空间，影响系统性能。因此，定期手动清理日志是一个推荐的做法。

本文档将指导您如何连接到 Dify 使用的 PostgreSQL 数据库，并执行 SQL 命令来安全地清理指定时间之前的日志数据。





## 清理步骤

### 1. 重要前提与注意事项

在执行任何数据库操作之前，请务必理解并遵循以下几点：

- **数据备份**：强烈建议在执行任何删除操作前，**完整备份您的 Dify 数据库**。这是防止意外数据丢失的关键步骤。
- **数据库访问权限**：您需要拥有直接访问 Dify 后端 PostgreSQL 数据库的权限和连接信息（如主机名、端口、数据库名、用户名和密码）。
- **了解 SQL 命令**：以下提供的 SQL 命令会永久删除数据。请确保您理解每一条命令的作用。
- **自定义清理周期**：示例中的 SQL 命令默认清理 4 周前的日志。您可以根据实际需求调整时间间隔，例如将 `'4 week'` 修改为 `'2 month'` (2 个月) 或 `'30 day'` (30 天) 等。

### 2. 连接到 PostgreSQL 数据库

您可以使用任何标准的 PostgreSQL 客户端工具（如 `psql` 命令行工具、pgAdmin、DBeaver 等）连接到 Dify 的数据库。

例如，使用 `psql` 的连接命令格式如下：

```bash
psql -h <主机名> -p <端口号> -U <用户名> -d <数据库名>
```

连接成功后，您将进入 SQL 命令交互提示符。

### 3. 执行 SQL 清理命令

以下 SQL 语句用于删除不同表中的过期日志数据。建议逐条执行，并观察执行结果。

```sql
-- 删除4周前创建的工作流运行记录
DELETE FROM public.workflow_runs
WHERE created_at < CURRENT_TIMESTAMP - INTERVAL '4 week';

-- 删除4周前创建的工作流节点执行记录
DELETE FROM public.workflow_node_executions
WHERE created_at < CURRENT_TIMESTAMP - INTERVAL '4 week';

-- 删除4周前创建的消息记录
DELETE FROM public.messages
WHERE created_at < CURRENT_TIMESTAMP - INTERVAL '4 week';

-- 删除4周前创建的工作流应用日志
-- 如果确认此表存在且需要清理，则取消注释下一行
DELETE FROM public.workflow_app_logs
WHERE created_at < CURRENT_TIMESTAMP - INTERVAL '4 week';

-- 删除4周前创建的终端用户记录 (请谨慎操作此表，确认这些用户数据是否可以安全删除)
DELETE FROM public.end_users
WHERE created_at < CURRENT_TIMESTAMP - INTERVAL '4 week';

-- 删除4周前创建的会话记录
DELETE FROM public.conversations
WHERE created_at < CURRENT_TIMESTAMP - INTERVAL '4 week';
```

**关于 `end_users` 表的说明：**

- `end_users`: 清理 `end_users` 表需要特别小心。这个表通常存储的是与您的 Dify 应用交互的最终用户信息。删除这些记录意味着您将丢失这些用户的历史信息。请务必确认业务上不再需要这些早于特定时间的用户数据，或者这些数据是匿名/不重要的测试数据。

### 4. 验证清理结果 (可选)

您可以执行一些 `SELECT COUNT(*)` 查询来验证数据是否已被删除，例如：

```sql
SELECT COUNT(*) FROM public.messages WHERE created_at < CURRENT_TIMESTAMP - INTERVAL '4 week';
```

如果结果为 0，说明指定时间之前的记录已被成功删除。

## 结论

通过执行上述步骤，您可以有效地管理 Dify 应用的日志数据，释放宝贵的存储空间，并可能提升系统性能。

为了实现更自动化的日志管理，您可以考虑将这些 SQL 清理脚本包装成一个脚本文件，并使用操作系统的计划任务功能（如 Linux 的 `cron` 或 Windows 的任务计划程序）来定期自动执行。但在配置自动化任务之前，请确保脚本的健壮性和安全性，并充分测试。

{% endraw %}
