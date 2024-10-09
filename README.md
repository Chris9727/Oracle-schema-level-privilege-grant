适用范围

Oracle数据库19c及以下版本
操作系统平台版本不限

脚本概述
由于Oracle 19c及之前版本不支持schema级别的授权操作（23ai新特性支持），故使用脚本方式满足对于一次全量授权后，后续新增表的追加授权，当前脚本支持单个、多个用户的追加权限授权使用，权限方面目前默认授权为查询权限。

脚本用例
1.创建权限管控用户（使用该用户授权需DBA权限）

create user privctl identified by Privctl!23;
grant resource,connect,dba to privctl;
创建参数表及日志表
--创建参数表
create table priv_user_tab (
 id number primary key,
 username varchar2(30),
 user_role varchar2(20) check (upper(user_role)='SOURCE' OR upper(user_role)='TARGET'),
 flag number);

--创建日志表
create table priv_grant_log(
 exec_time date,
 exec_text varchar2(1000),
 sql_code varchar2(1000),
 sql_err varchar2(1000));
创建权限追加存储过程
create or replace procedure privilege_grant authid current_user is
g_sql varchar2(1000);
sql_cd varchar2(1000);
sql_err varchar2(1000);
begin
for t_user in (select username,flag from priv_user_tab where upper(user_role)='TARGET') loop
  for s_user in (select username from priv_user_tab where upper(user_role)='SOURCE' and flag=t_user.flag) loop
   for g_info in (select owner,table_name from dba_tables where owner=upper(s_user.username) minus select owner,table_name from dba_tab_privs where grantee=upper(t_user.username) ) loop
    g_sql := 'grant select on "'||g_info.owner||'"."'||g_info.table_name||'" to '||t_user.username;
     begin
	  execute immediate g_sql;
      exception
      when others then
      	sql_cd:=sqlcode;
      	sql_err:=sqlerrm;
        insert into priv_grant_log values (sysdate,g_sql,sql_cd,sql_err);
        commit;
     end; 
    end loop;
  end loop;
end loop;
end;
/
插入授权参数
例：授权user2,user3的所有表查询权限与user1;

insert into priv_user_tab values (1,'USER1','TARGET',1);
insert into priv_user_tab values (2,'USER2','SOURCE',1);
insert into priv_user_tab values (3,'USER3','SOURCE',1);
参数表字段解释：

id:数值类型序号
username:用户名称（可小写）
user_role:用户角色，限定source/target(不区分大小写)，source为持有表用户，target为被授权用户
flag:数值类型分组序号，授权用户与被授权用户对应关系，相同代表同一组，根据flag进行分组授权。
设置定时任务&测试执行
连接privctl用户，创建任务定时执行：

declare
  job number;
  v_date varchar2(20) := to_char(sysdate,'yyyymmdd hh:mi:ss');
begin
  dbms_scheduler.create_job(
    job_name => 'priv_job',-- 定时器名字
    job_type => 'STORED_PROCEDURE',-- 类型：存储过程
    job_action => 'privilege_grant', -- 存储过程名称
    repeat_interval => 'FREQ=MINUTELY;interval=60', -- 定时规则，每60分钟执行一次
    enabled =>false -- 创建后不激活
  );
  dbms_scheduler.enable('priv_job'); -- 激活
  dbms_scheduler.run_job('priv_job'); -- 运行job
  end;
关闭任务：

dbms_scheduler.enable('priv_job');
脚本清理
删除定时任务
dbms_scheduler.drop('priv_job');
删除存储过程
drop procedure privilege_grant;
删除参数、日志表
drop table priv_user_tab;
drop table priv_grant_log;
