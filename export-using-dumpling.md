---
title: 使用 Dumpling 进行数据导出
summary: 使用新的导出工具 Dumpling 导出或者备份数据。
category: how-to
---

# 使用 Dumpling 进行数据导出

本文档将会介绍如何使用新的导出工具 Dumpling。
它可以把 TiDB 中存储的数据导出为 SQL 或者 CSV 格式，可以用它完成逻辑上的全量备份或者导出。
如果需要直接备份 SST 文件（KV 对）或者对延迟不敏感的增量备份，请参阅 [BR](/br/backup-and-restore-tool.md)。
如果需要实时的增量备份，请参阅 [TiCDC](/ticdc/ticdc-overview.md)。

Dumpling 需要在已经启动的集群上执行导出命令。这篇文章接下来假设在 `127.0.0.1:4000` 有一个 TiDB 实例，并且那个 TiDB 中有无密码的 root 用户。 

## 从 TiDB 导出数据

使用如下命令导出数据：

{{< copyable "shell-regular" >}}

```shell
dumpling \
  -u root \
  -P 4000 \
  -H 127.0.0.1 \
  --filetype sql \
  --threads 32 \
  -o /tmp/test \
  -F $(( 1024 * 1024 * 256 ))
```

这个命令中，`-H`，`-P`，`-u` 是经典的“地址，端口，用户”三元组。如果需要密码验证，可以用 `-p $YOUR_SECRET_PASSWORD` 传给 Dumpling。

默认情况下，除了系统数据库中的表之外，Dumpling 会导出整个数据库的表。你可以使用 `--where <SQL where expression>` 来选定要导出的记录。假如导出数据的格式是 CSV（使用 `--filetype csv` 即可导出 CSV 文件），还可以使用 `--sql <SQL>` 导出指定 SQL 选择出来的记录；例如，导出所有 `id < 100` 的

> **注意：**
> 
> 截止到这篇文章写作时候，Dumpling 暂时还不支持仅导出用户指定的某几张表（即 `-T` 标志，见[这个 issue](https://github.com/pingcap/dumpling/issues/76)）。
> 如果你确实需要这些功能，可以先使用 [MyDumper](/backup-and-restore-using-mydumper-lightning.md)。

默认情况下，导出的文件会存储到 `./export-<current local time>` 目录下。你可以使用 `-o` 来选择存储导出文件的目录。`-F` 选项能够指定单个文件的最大大小（和 MyDumper 不同，这里的单位是字节），与之相似的是 `-r` 选项，它指定单个文件的最大记录数（或者说，数据库中的行数）。利用这些参数可以让 Dumpling 的并行度更高。

比如说，使用以下指令，可以导出所有 `id < 100` 的记录：

{{< copyable "shell-regular" >}}

```shell
./dumpling \
  -u root \
  -P 4000 \
  -H 127.0.0.1 \
  -o /tmp/test \
  --where "id < 100"
```

> **提示：**
> 
> 这里需要在要导出的所有表上执行 `select * from <table-name> where <your --where flag>` 语句。如果部分表没有指定的字段，那么导出会失败。

还有一个尚未在上面展示出来的标志是 `--consistency <consistency level>`，这个标志控制导出数据“一致性保证”的方式。对于 TiDB 来说，默认情况下我们会通过获取某个时间戳的快照来保证一致性（即 `--consistency snapshot`）。在使用 snapshot 来保证一致性的时候，我们可以使用 `--snapshot` 参数指定要备份的时间戳。我们还可以使用以下的一致性级别：

- `flush`: 使用 [`FLUSH TABLES WITH READ LOCK`](https://dev.mysql.com/doc/refman/8.0/en/flush.html#flush-tables-with-read-lock) 来保证一致性。
- `snapshot`: 获取指定时间戳的一致性快照并导出。
- `lock`: 为待导出的所有表上读锁。
- `none`: 不做任何一致性保证。
- `auto`: 对 MySQL 使用 `flush`，对 TiDB 使用 `snapshot`。

一切完成之后，你应该可以在 `/tmp/test` 看到导出的文件了：

``` shell
$ ls -lh /tmp/test | awk '{print $5 "\t" $9}'

140B  metadata
66B   test-schema-create.sql
300B  test.sbtest1-schema.sql
190K  test.sbtest1.0.sql
300B  test.sbtest2-schema.sql
190K  test.sbtest2.0.sql
300B  test.sbtest3-schema.sql
190K  test.sbtest3.0.sql
```

另外，假如数据量非常大，可以提前调长 GC 时间避免因为导出中 GC 导致导出失败：

{{< copyable "sql" >}}

```sql
update mysql.tidb set VARIABLE_VALUE = '720h' where VARIABLE_NAME = 'tikv_gc_life_time';
```

在操作结束之后，再将 GC 时间调回原样（默认是 `10m`）：

{{< copyable "sql" >}}

```sql
update mysql.tidb set VARIABLE_VALUE = '10m' where VARIABLE_NAME = 'tikv_gc_life_time';
```

最后，所有的这些导出数据都可以用 [Lightning](/tidb-lightning/tidb-lightning-tidb-backend.md) 导入回 TiDB。