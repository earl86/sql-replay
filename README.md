# 功能介绍
## parse 部分
读取 MySQL 慢查询日志，去掉 MySQL 中自动生成的 set timestamp=xx/# Administor/-- 等无效 SQL，生成一个可以格式化的 json 文件，用于回放
## replay 部分
1. 读取格式化后的 json 文件，支持指定上游指定用户、上游 SQL 类型（all、select）来进行回放
2. 回放前将日志根据 connection id 并行，相同 connection id 的 SQL 串行
3. 将回放结果输出成 json 文件（按照 connection id 区分）
## load 部分
1. 解析 replay 生成的 json 文件，使用 TiDB Parse 模块对 SQL 进行格式化，并生成指纹（sql digest）
2. 将解析出来的信息写入数据库的 replay_info 表中
## gen_report.py
对回放结果进行分析，生成回放报告（含响应时间对比、返回行数不一致、错误信息）

# 操作示例 
## 下载并解压 
```
mkdir replay && cd replay && wget https://github.com/Bowen-Tang/sql-replay/releases/download/v0.2/v0.2.zip
unzip v0.2.zip
```
 
## 1. 解析慢查询日志
```
./sql-replay -mode parse -slow-in /opt/slow.log -slow-out /opt/slow.format
```
说明：/opt/slow.log 为慢查询日志路径，slow.format 则为输出的格式化文件

## 2. 连接目标库回放

```
mkdir out # 用户存储回放结果
# 回放所有用户、所有 SQL
./sql-replay -mode replay -db "user:password@tcp(ip:port)/db" -slow-out /opt/slow.format -replay-out ./out/sb1_all -username all -sqltype all
# 回放所有用户、select 语句
./sql-replay -mode replay -db "user:password@tcp(ip:port)/db" -slow-out /opt/slow.format -replay-out ./out/sb1_select -username all -sqltype select
```
说明：out 为回放结果存储目录（可更换为其他目录，需手动创建），sb1_all/sb1_select 为回放任务名称

## 3. 导入回放结果到数据库
**连接目标库，创建表结构**
```
CREATE TABLE `test`.`replay_info` (
`sql_text` longtext DEFAULT NULL,
`sql_type` varchar(16) DEFAULT NULL,
`sql_digest` varchar(64) DEFAULT NULL,
`query_time` bigint(20) DEFAULT NULL,
`rows_sent` bigint(20) DEFAULT NULL,
`execution_time` bigint(20) DEFAULT NULL,
`rows_returned` bigint(20) DEFAULT NULL,
`error_info` text DEFAULT NULL,
`file_name` varchar(64) DEFAULT NULL
);
```
**导入数据**
```
# 导入回放任务 sb1_all 的回放数据 
./sql-replay -mode load -db "user:password@tcp(ip:port)/db" -out-dir ./out -replay-name sb1_all -table replay_info 
# 导入回放任务 sb1_select 的回放数据 
./sql-replay -mode load -db "user:password@tcp(ip:port)/db" -out-dir ./out -replay-name sb1_select -table replay_info 
```
说明：-out-dir 为回放结果存储目录，-replay-name 回放任务名称，table 为写入结果表

## 4. 生成报告

```
yum install -y python3
pip3 install pandas
pip3 install mysql-connector-python
# 生成 sb1_all 回放任务的报告
python3 ./gen_report.py --user username --password password --host ip --port port --database test --replay sb1_all
```
说明：执行完成会输出 sb1_all.html，下载到本地查看

# 报告示例 
![image](https://github.com/Bowen-Tang/sql-replay/assets/52245161/6259a65d-90d5-420e-8f54-8c35ccfe0b69)
![image](https://github.com/Bowen-Tang/sql-replay/assets/52245161/9ddb8433-e83e-48d4-8dd6-65f434d1f5ef)

# 编译安装方法

1. 安装 golang 1.20 及以上
2. Python 3 环境： mysql-connector-python (8.0.29+)、pandas (1.1.5)
3. 下载项目

```
git clone https://github.com/Bowen-Tang/sql-replay
```

4. 编译 sql-replay

```
    cd sql-replay
    go mod init sql-replay
    go mod tidy
    go build
```
