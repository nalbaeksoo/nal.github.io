filepath : ?/sqlplus/admin/glogin.sql

Syntax highlighted code block
set lines 120
set pages 900
define_editor=vi
 
set verify off termout off head off feed off
 
col login_prompt new_value welcome
 
SELECT upper(SYS_CONTEXT('USERENV','SERVER_HOST')
||' '
|| SYS_CONTEXT('USERENV','CURRENT_USER')
||'@'
|| SYS_CONTEXT('USERENV','DB_NAME')) login_prompt
FROM DUAL
;
 
 
set sqlprompt "_user'@'_connect_identifier> "
 
set verify on termout on head on feed on
prompt ************************************
prompt WELCOME TO &&welcome
prompt ************************************
prompt
set echo off serveroutput on size 100000 line 100 trims on
set time on
set timing on
