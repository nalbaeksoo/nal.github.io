---
layout: post
title:  "null column은 data size 측정이 어떻게 되는가..."
date:   2019-12-18 12:00:00 +0900
categories: Oracle Adm
---

### null column은 data size 측정이 어떻게 되는가...

급 모델러 분이 column에서 null 값이 있을때 용량산정을 물어보심..
익히 알고 있는바는 있었나 검증되지 않아 이번에 검증하도록 함미당. +_+)
다음과 같은 table을 생성한다.

```
SQL> desc null_table  

 Name                    Null?    Type        
 ----------------------- -------- ------------
COL1                             VARCHAR2(1) 
COL2                             NUMBER(5)   
COL3                             CHAR(2)     
COL4                             VARCHAR2(10)
COL5                             VARCHAR2(10)
COL6                             VARCHAR2(10)
```

해당 object를 확인해보쟈-

```
SQL> select header_file, header_block
from dba_segments where segment_name='NULL_TABLE';
no rows selected
       
SQL> -- 11g 부터는 데이터 입력값이 있어야 segment 생성

데이터를 넣어보겠슴미당.
insert into null_table values(1,null,null,null,null,null);   
insert into null_table values(1,1,null,null,null,null);      
insert into null_table values(1,null,null,null,null,1);      
insert into null_table values(1,null,null,1,null,1);         
insert into null_table values(1,null,null,null,null,1);      
insert into null_table values(null,null,null,null,null,null);
commit;
```

다시한번 슥삭- 해보면

```
SQL> select header_file, header_block 
from dba_segments where segment_name='NULL_TABLE';
                                                         
HEADER_FILE      HEADER_BLOCK        
-----------      ------------                  
4                50682                                           
```

쨔잔~ 하고 나옴-!
기타 등등을 알아보자-

```
SQL> select rowid, dbms_rowid.rowid_relative_fno(rowid) fno 
2  dbms_rowid.rowid_block_number(rowid) blkno,    
3  dbms_rowid.rowid_row_number(rowid) rowno      
4  from null_table;

ROWID                  FNO      BLKNO      ROWNO        
------------------ ---------- ---------- ----------        
AAAS5YAAEAAAMX9AAA          4      50685          0    
AAAS5YAAEAAAMX9AAB          4      50685          1        
AAAS5YAAEAAAMX9AAC          4      50685          2        
AAAS5YAAEAAAMX9AAD          4      50685          3        
AAAS5YAAEAAAMX9AAE          4      50685          4        
AAAS5YAAEAAAMX9AAF          4      50685          5    

SQL> select block_size from v$datafile where file#=4;

BLOCK_SIZE
----------
8192

```

음... block size는 8k인걸 확인할수 있다.
check point를 따시~ 먹이고 datafile dump를 생성하도록 하쟈-

```
SQL> alter system checkpoint;

System altered.

SQL> alter system dump datafile 4 block 50685;

System altered.



userdump_dest 항목을 참조..
*** 2012-05-22 11:55:42.437

Block header dump:  0x0100c5fd
 Object id on Block? Y
 seg/obj: 0x12e58  csc: 0x00.1a4dcf  itc: 2  flg: E  typ: 1 - DATA
     brn: 0  bdba: 0x100c5f8 ver: 0x01 opc: 0
     inc: 0  exflg: 0


 Itl           Xid                  Uba         Flag  Lck        Scn/Fsc
0x01   0x0004.012.000005b5  0x00c02a5c.011a.0c  --U-    6  fsc 0x0000.001a4dd0
0x02   0x0000.000.00000000  0x00000000.0000.00  ----    0  fsc 0x0000.00000000
bdba: 0x0100c5fd

data_block_dump,data header at 0x7ffc5adab064
===============
tsiz: 0x1f98
hsiz: 0x1e
pbl: 0x7ffc5adab064
     76543210
flag=--------
ntab=1
nrow=6
frre=-1
fsbo=0x1e
fseo=0x1f66
avsp=0x1f3d
tosp=0x1f3d
0xe:pti[0]      nrow=6  offs=0
0x12:pri[0]     offs=0x1f93
0x14:pri[1]     offs=0x1f8b
0x16:pri[2]     offs=0x1f80
0x18:pri[3]     offs=0x1f74
0x1a:pri[4]     offs=0x1f69
0x1c:pri[5]     offs=0x1f66
block_row_dump:

tab 0, row 0, @0x1f93
tl: 5 fb: --H-FL-- lb: 0x1  cc: 1
col  0: [ 1]  31
tab 0, row 1, @0x1f8b
tl: 8 fb: --H-FL-- lb: 0x1  cc: 2
col  0: [ 1]  31
col  1: [ 2]  c1 02
tab 0, row 2, @0x1f80
tl: 11 fb: --H-FL-- lb: 0x1  cc: 6
col  0: [ 1]  31
col  1: *NULL*
col  2: *NULL*
col  3: *NULL*
col  4: *NULL*
col  5: [ 1]  31
tab 0, row 3, @0x1f74
tl: 12 fb: --H-FL-- lb: 0x1  cc: 6
col  0: [ 1]  31
col  1: *NULL*
col  2: *NULL*
tl: 5 fb: --H-FL-- lb: 0x1  cc: 1
col  0: [ 1]  31
tab 0, row 1, @0x1f8b
tl: 8 fb: --H-FL-- lb: 0x1  cc: 2
col  0: [ 1]  31
col  1: [ 2]  c1 02
tab 0, row 2, @0x1f80
tl: 11 fb: --H-FL-- lb: 0x1  cc: 6
col  0: [ 1]  31
col  1: *NULL*
col  2: *NULL*
col  3: *NULL*
col  4: *NULL*
col  5: [ 1]  31
tab 0, row 3, @0x1f74
tl: 12 fb: --H-FL-- lb: 0x1  cc: 6
col  0: [ 1]  31
col  1: *NULL*
col  2: *NULL*
col  3: [ 1]  31
col  4: *NULL*
col  5: [ 1]  31
tab 0, row 4, @0x1f69
tl: 11 fb: --H-FL-- lb: 0x1  cc: 6
col  0: [ 1]  31
col  1: *NULL*
col  2: *NULL*
col  3: *NULL*
col  4: *NULL*
col  5: [ 1]  31
tab 0, row 5, @0x1f66
tl: 3 fb: --H-FL-- lb: 0x1  cc: 0

end_of_block_dump
```

음.. 다음과 같이 나왔다..
하나 하나 살펴보도록 하여욤..!

```
SQL> select * from null_table;

C       COL2 CO COL4       COL5       COL6
- ---------- -- ---------- ---------- ----------
1

tab 0, row 0, @0x1f93
tl: 5 fb: --H-FL-- lb: 0x1  cc: 1
col  0: [ 1]  31
```

보다시피 첫번째 행에는 1 값밖에 보이지 않는다.
col 0 첫번째컬럼에 [ 1] 1바이트를 차지하고 있으며 31 16진수로 표현된 ascii code값임을 확인
NULL 값은 header정보에 포함이 되어있지 않음을 확인할수 있다.
더불어 cc 값에 주목하도록 하자-!

```
C       COL2 CO COL4       COL5       COL6
- ---------- -- ---------- ---------- ----------
1          1

tab 0, row 1, @0x1f8b
tl: 8 fb: --H-FL-- lb: 0x1  cc: 2
col  0: [ 1]  31
col  1: [ 2]  c1 02
```

마찬가지이다..
다만 두번째행은 column type이 number 이므로 31이 아닌 c1으로 표시되었다.

```
C       COL2 CO COL4       COL5       COL6
- ---------- -- ---------- ---------- ----------
1                                     1

tab 0, row 2, @0x1f80
tl: 11 fb: --H-FL-- lb: 0x1  cc: 6
col  0: [ 1]  31col  1: *NULL*
col  2: *NULL*
col  3: *NULL*
col  4: *NULL*
col  5: [ 1]  31
```

여기서부터가 좀 흥미로운데 중간에 NULL에 대한 표시는 있으나
해당 byte에 대한 점유는하지 않은것으로 확인된다.
여전히 col6에 대한 정보는 표시 되지 않는다.

```
C       COL2 CO COL4       COL5       COL6
- ---------- -- ---------- ---------- ----------
1               1                     1

tab 0, row 3, @0x1f74
tl: 12 fb: --H-FL-- lb: 0x1  cc: 6
col  0: [ 1]  31
col  1: *NULL*
col  2: *NULL*
col  3: [ 1]  31
col  4: *NULL*
col  5: [ 1]  31
```

같은 패턴을 보이고 있는것이 확인된다.

```
C       COL2 CO COL4       COL5       COL6
- ---------- -- ---------- ---------- ----------
1                                     1

tab 0, row 4, @0x1f69
tl: 11 fb: --H-FL-- lb: 0x1  cc: 6
col  0: [ 1]  31
col  1: *NULL*
col  2: *NULL*
col  3: *NULL*
col  4: *NULL*
col  5: [ 1]  31
```

위와 같다.

```
C       COL2 CO COL4       COL5       COL6
- ---------- -- ---------- ---------- ----------

tab 0, row 5, @0x1f66
tl: 3 fb: --H-FL-- lb: 0x1  cc: 0
```
전부다 NULL 경우에는 아무것도 표시되지 않음을 알수있다..

그러므로 modeling 단계부터 data에 대한 분포도값을 정확히 측정하여
null column 이 많은경우 해당 column을 최대한 뒤로 배치하여 앞에서부터읽을수 있도록 조치하면되겠당.. 중간에 null 이 아닌 다른값이 들어와 있으면해당 부분까지 scan을 하는것도 비용처리가 될꺼라 생각이 된다..

하다보니 index에 대한 궁금증이 생겨서 테스트를 더 진행해 보았다



index를 생성 6번째 컬럼을 기준으로 index를 생성한다

```
SQL> create index COL6_NULL on NULL_TABLE(COL6);

SQL> select object_name ,object_id from dba_objects where object_name ='COL6_NULL';

OBJECT_NAME  OBJECT_ID
----------- ----------
COL6_NULL     77401
```


tree level dump를 뜹니당-

```
SQL> alter session set events 'immediate trace name treedump level 77401';



Session altered.
```

가뿐하게 trace file 생성후 열어보도록 합니다


```
[oracle@jhbae trace]$ vi DB11G_ora_31214_COL6_NULL.trc

Trace file /u01/app/oracle/diag/rdbms/db11g/DB11G/trace/DB11G_ora_31214_COL6_NULL.trc

Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options
ORACLE_HOME = /u01/app/oracle/product/11.2.0/dbhome_1
System name:    Linux
Node name:      jhbae
Release:        2.6.32-300.3.1.el6uek.x86_64
Version:        #1 SMP Fri Dec 9 18:57:35 EST 2011
Machine:        x86_64
Instance name: DB11G
Redo thread mounted by this instance: 1
Oracle process number: 27
Unix process pid: 31214, image: oracle@jhbae (TNS V1-V3)

*** 2012-05-22 14:06:46.957
*** SESSION ID:(35.215) 2012-05-22 14:06:46.957
*** CLIENT ID:() 2012-05-22 14:06:46.957
*** SERVICE NAME:(SYS$USERS) 2012-05-22 14:06:46.957
*** MODULE NAME:(SQL*Plus) 2012-05-22 14:06:46.957
*** ACTION NAME:() 2012-05-22 14:06:46.957

----- begin tree dump

leaf: 0x100c603 16827907 (0: nrow: 3 rrow: 3)

----- end tree dump
```


도출된 값을 가지고 dump file을 생성합니다.


```
[oracle@jhbae trace]$ sqlplus / as sysdba

SQL*Plus: Release 11.2.0.3.0 Production on Tue May 22 14:08:00 2012

Copyright (c) 1982, 2011, Oracle.  All rights reserved.



Connected to:

Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production

With the Partitioning, OLAP, Data Mining and Real Application Testing options



SQL> select dbms_utility.data_block_address_file(16827907) as file_no,
  2  dbms_utility.data_block_address_block(16827907) as block_no from dual;

   FILE_NO   BLOCK_NO
---------- ----------
         4      50691

Elapsed: 00:00:00.05



SQL> alter system dump datafile 4 block 50691;

System altered.

Elapsed: 00:00:00.03



[oracle@jhbae trace]$ vi DB11G_ora_32179.trc
Trace file /u01/app/oracle/diag/rdbms/db11g/DB11G/trace/DB11G_ora_32179.trc
Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options
ORACLE_HOME = /u01/app/oracle/product/11.2.0/dbhome_1
System name:    Linux
Node name:      jhbae
Release:        2.6.32-300.3.1.el6uek.x86_64
Version:        #1 SMP Fri Dec 9 18:57:35 EST 2011
Machine:        x86_64
Instance name: DB11G
Redo thread mounted by this instance: 1
Oracle process number: 27
Unix process pid: 32179, image: oracle@jhbae (TNS V1-V3)

*** 2012-05-22 14:25:19.217
*** SESSION ID:(35.221) 2012-05-22 14:25:19.217
*** CLIENT ID:() 2012-05-22 14:25:19.217
*** SERVICE NAME:(SYS$USERS) 2012-05-22 14:25:19.217
*** MODULE NAME:(sqlplus@jhbae (TNS V1-V3)) 2012-05-22 14:25:19.217
*** ACTION NAME:() 2012-05-22 14:25:19.217

Start dump data blocks tsn: 4 file#:4 minblk 50691 maxblk 50691
Block dump from cache:
Dump of buffer cache at level 4 for tsn=4 rdba=16827907
BH (0x71bdbca8) file#: 4 rdba: 0x0100c603 (4/50691) class: 1 ba: 0x71868000
  set: 3 pool: 3 bsz: 8192 bsi: 0 sflg: 1 pwc: 397,28
  dbwrid: 0 obj: 77401 objn: 77401 tsn: 4 afn: 4 hint: f
  hash: [0x922132e0,0x922132e0] lru: [0x71bdbec0,0x71bf4ef0]
  ckptq: [NULL] fileq: [NULL] objq: [0x643fc118,0x88732b38] objaq: [0x6a7db578,0x88732b28]
  st: XCURRENT md: NULL fpin: 'ktspbwh2: ktspfmdb' tch: 2
  flags: block_written_once redo_since_read
  LRBA: [0x0.0.0] LSCN: [0x0.0] HSCN: [0xffff.ffffffff] HSUB: [1]
Block dump from disk:
buffer tsn: 4 rdba: 0x0100c603 (4/50691)
scn: 0x0000.001a66e5 seq: 0x01 flg: 0x04 tail: 0x66e50601
frmt: 0x02 chkval: 0xb9e8 type: 0x06=trans data
Hex dump of block: st=0, typ_found=1

 seg/obj: 0x12e59  csc: 0x00.1a66e3  itc: 2  flg: E  typ: 2 - INDEX
     brn: 0  bdba: 0x100c600 ver: 0x01 opc: 0
     inc: 0  exflg: 0
 Itl           Xid                  Uba         Flag  Lck        Scn/Fsc
0x01   0x0000.000.00000000  0x00000000.0000.00  ----    0  fsc 0x0000.00000000
0x02   0xffff.000.00000000  0x00000000.0000.00  C---    0  scn 0x0000.001a66e3

Leaf block dump
===============
header address 139870805363300=0x7f3635ab0a64
kdxcolev 0
KDXCOLEV Flags = - - -
kdxcolok 0
kdxcoopc 0x80: opcode=0: iot flags=--- is converted=Y
kdxconco 2
kdxcosdc 0
kdxconro 3
kdxcofbo 42=0x2a
kdxcofeo 7999=0x1f3f
kdxcoavs 7957
kdxlespl 0
kdxlende 0
kdxlenxt 0=0x0
kdxleprv 0=0x0
kdxledsz 0
kdxlebksz 8032
row#0[8021] flag: ------, lock: 0, len=11
col 0; len 1; (1):  31
col 1; len 6; (6):  01 00 c5 fd 00 02
row#1[8010] flag: ------, lock: 0, len=11
col 0; len 1; (1):  31
col 1; len 6; (6):  01 00 c5 fd 00 03
row#2[7999] flag: ------, lock: 0, len=11
col 0; len 1; (1):  31
col 1; len 6; (6):  01 00 c5 fd 00 04
----- end of leaf block dump -----
```


위에서 도출된값과 같이 len (1) 16진수로 31을 가리키고 있는것을 확인할수 있음.
그렇다면 두번째 커럼인 col 1의 정체는 무엇인가... 그것은 바로..!
row id 값... 그것을 확인해 봅니당-


```
SQL*Plus: Release 11.2.0.3.0 Production on Tue May 22 14:54:11 2012
Copyright (c) 1982, 2011, Oracle.  All rights reserved.



Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options



SQL> select rowid extended_format,
dbms_rowid.rowid_to_restricted(rowid,0),col1,col2,col3,col4,col5,col6
 2   from null_table;



EXTENDED_FORMAT    DBMS_ROWID.ROWID_T C       COL2 CO COL4       COL5       COL6
------------------ ------------------ - ---------- -- ---------- ---------- ----------
AAAS5YAAEAAAMX9AAA 0000C5FD.0000.0004 1
AAAS5YAAEAAAMX9AAB 0000C5FD.0001.0004 1          1
AAAS5YAAEAAAMX9AAC 0000C5FD.0002.0004 1                                     1
AAAS5YAAEAAAMX9AAD 0000C5FD.0003.0004 1               1                     1
AAAS5YAAEAAAMX9AAE 0000C5FD.0004.0004 1                                     1
AAAS5YAAEAAAMX9AAF 0000C5FD.0005.0004

6 rows selected.



Elapsed: 00:00:00.01
```

index는 null값을 저장하지 않는것으로 확인하였습니다-
그리고 rowid는 위와같이 변환해서 보면 rowid_to_restricted 부분과 
index dump file의 col1 이 일치되는것을 확인할수 있음!

끗!
