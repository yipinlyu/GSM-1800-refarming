# GSM-1800-refarming
SQL语句：

exec sp_configure 'show advanced options',1

reconfigure

exec sp_configure 'Ad Hoc Distributed Queries',1

reconfigure

EXEC master.dbo.sp_MSset_oledb_prop N'Microsoft.ACE.OLEDB.12.0', N'AllowInProcess', 1

EXEC master.dbo.sp_MSset_oledb_prop N'Microsoft.ACE.OLEDB.12.0', N'DynamicParameters', 1

select * into 表 from OPENROWSET('Microsoft.Ace.OLEDB.12.0','Excel 12.0;HDR=YES;DATABASE=F:\美丽杭州退频区域MR数据\omc30\0306\a.xlsx',sheet1$)

BULK INSERT MR数据lyp.dbo.MR数据十月

FROM 'D:\共享文件夹\MR数据\omc231合并_2.csv' ----修改文件目录和名称

WITH

  (

     FIELDTERMINATOR =',',--字段分割符号

     ROWTERMINATOR ='\n'--换行符号

  )
----导入原始数据格式需为csv---

BULK INSERT MR数据.[dbo].[MR数据]

FROM 'G:\韦\合并\p.csv' ----修改文件目录和名称

WITH

(

FIELDTERMINATOR =',',--字段分割符号

ROWTERMINATOR ='\n'--换行符号

)

----删除多余表头---

delete MR数据lyp.dbo.美丽杭州MR数据

where 时间='时间'

---修改字段类型---

ALTER TABLE MR数据.dbo.MR数据

ALTER COLUMN S372 real

---增加字段---

ALTER TABLE MR数据lyp.dbo.美丽杭州MR数据

add [CI] [int] NULL

[BCCH] [int] NULL,

[CI] [int] NULL
---对增加字段进行赋值---

create function [dbo].[HexToDec](@A nvarchar(100))

returns int

as

begin

declare @b int

select @A=replace(@A,'H',''),@b=0

while len(@A)<>0

begin

select @b=@b*16+case left(@A,1) when 'A' then 10

when 'B' then 11

when 'C' then 12

when 'D' then 13

when 'E' then 14

when 'F' then 15

else left(@A,1)

end

set @A=right(@A,len(@A)-1)

end

return @b

end

update MR数据lyp.dbo.美丽杭州MR数据

set ci=dbo.HexToDec

where 时间>='2015-11-18 16:00' and 时间<'2015-11-19 00:00'

update MR数据.[dbo].[MR数据]

set bcch=SUBSTRING(对象名称,CHARINDEX('BCCH:',对象名称)+5,CHARINDEX('BCC.NCC',对象名称) - CHARINDEX('BCCH',对象名称)-6)

update MR数据.[dbo].[MR数据]

set bsci=RIGHT(对象名称,1)+substring(对象名称,charindex('BCC.NCC:',对象名称)+8,1)

----对原始数据求和处理---

insert into MR数据lyp.[dbo].MR数据十月步骤一

select 对象名称,BSCI,BCCH,CI

,sum(cast([S360] as real)) as S360,sum(cast([S361] as real)) as S361,sum(cast([S362] as real)) asS362,sum(cast([S363] as real)) as S363,sum(cast([S364] as real)) as S364,sum(cast([S365] as real)) asS365,sum(cast([S366] as real)) as S366

,sum(cast([S367] as real)) as S367,sum(cast([S368] as real)) as S368,sum(cast([S369] as real)) asS369,sum(cast([S370] as real)) as S370

from MR数据lyp.dbo.MR数据十月

group by 对象名称,BSCI,BCCH,CI

----对原始数据求和处理---

INSERT INTO [MR数据].[dbo].[MR数据步骤二]

([对象名称],[AS362],[S360],[S361],[S362],[S363],[S364],[S365],[S366],[S367],[S368],[S369],[S370],[S371],[BSCI],[BCCH],[CI],[经度],[纬度])

SELECT [对象名称],[AS362],[S360],[S361],[S362],[S363],[S364],[S365],[S366],[S367],[S368],[S369],[S370],[S371],a.[BSCI],a.[BCCH],a.[CI],经度,纬度

FROM [MR数据].[dbo].[MR数据步骤一] a inner join [MR数据].[dbo].[小区经纬度] b on a.ci=b.ci

----距离计算---

INSERT INTO [MR数据].[dbo].[MR数据步骤三]

([对象名称],[AS362],[S360],[S361],[S362],[S363],[S364],[S365],[S366],[S367],[S368],[S369],[S370],[S371],[BSCI],[BCCH]

,[主小区CI],[主小区经度],[主小区纬度],[邻小区CI],[邻小区经度],[邻小区纬度],[距离])

SELECT [对象名称],[AS362],[S360],[S361],[S362],[S363],[S364],[S365],[S366],[S367],[S368],[S369],[S370],[S371],[BSCI],a.[BCCH],[主小区CI]

  ,[主小区经度] ,[主小区纬度],

  b.ci as 邻小区CI,

  b.经度 as 邻小区经度,

  b.纬度 as 邻小区纬度,

  sqrt((主小区纬度-纬度)*(主小区纬度-纬度)+(主小区经度-经度)*(主小区经度-经度)*0.75)*400/360*100 as 距离
FROM [MR数据].[dbo].[MR数据步骤二] a inner join [MR数据].[dbo].[小区经纬度] b on a.bcch=b.bcch anda.BSCI=b.bsic

----最小距离计算---

INSERT INTO [MR数据].[dbo].[MR数据步骤四]

([对象名称],[AS362],[S360],[S361],[S362],[S363],[S364],[S365],[S366],[S367],[S368],[S369],[S370],[S371],[BSCI],[BCCH]

,[主小区CI],[主小区经度],[主小区纬度],[邻小区CI],[邻小区经度],[邻小区纬度],[距离])

select [对象名称],[AS362],[S360],[S361],[S362],[S363],[S364],[S365],[S366],[S367],[S368],[S369],[S370],[S371],b.[BSCI],b.[BCCH],b.[主小区CI],[主小区经度]

  ,[主小区纬度]

  ,[邻小区CI]

  ,[邻小区经度]

  ,[邻小区纬度]

  ,b.[距离]

  from
(

select 主小区CI,BCCH,BSCI,MIN(距离) as 距离

from [MR数据].dbo.MR数据步骤三

group by 主小区CI,BCCH,BSCI

)a inner join [MR数据].dbo.MR数据步骤三 b on a.主小区CI=b.主小区CI and a.BCCH=b.BCCH and a.BSCI=b.BSCIand a.距离=b.距离

Ø 将小区频点方案带入小区对SCI&NCI ，计算小区对SCI&NCI同频及邻频采样点分布情况。

SQL语句：

----分频后频点对应---

INSERT INTO [MR数据].[dbo].[MR数据步骤分频后]

([主小区CI]

,[邻小区CI]

,[主小区分频后频点]

,[邻小区分频后频点]

,[差值])

select c.主小区CI as 主小区CI,c.邻小区CI as 邻小区CI,c.主小区分频后频点 as 主小区分频后频点,d.邻小区分频后频点 as 邻小区分频后频点,

(c.主小区分频后频点-d.邻小区分频后频点) as 差值

from

(

select a.主小区CI as 主小区CI,a.邻小区CI as 邻小区CI,b.分频后频点 as 主小区分频后频点

from [MR数据].dbo.MR数据步骤四 a inner join [MR数据].dbo.分频后频点方案 b on a.主小区CI=b.CI

)c

inner join

(

select a.主小区CI as 主小区CI,a.邻小区CI as 邻小区CI,b.分频后频点 as 邻小区分频后频点

from [MR数据].dbo.MR数据步骤四 a inner join [MR数据].dbo.分频后频点方案 b on a.邻小区CI=b.CI

)d

on c.主小区CI=d.主小区CI and c.邻小区CI=d.邻小区CI

where (c.主小区分频后频点-d.邻小区分频后频点)=0 or abs(c.主小区分频后频点-d.邻小区分频后频点)=1

----分频前频点对应---

INSERT INTO [MR数据].[dbo].[MR数据步骤分频前]

([主小区CI]

,[邻小区CI]

,[主小区分频前频点]

,[邻小区分频前频点]

,[差值])

select c.主小区CI as 主小区CI,c.邻小区CI as 邻小区CI,c.主小区分频前频点 as 主小区分频前频点,d.邻小区分频前频点 as 邻小区分频前频点,

(c.主小区分频前频点-d.邻小区分频前频点) as 差值

from

(

select a.主小区CI as 主小区CI,a.邻小区CI as 邻小区CI,b.分频前频点 as 主小区分频前频点

from [MR数据].dbo.MR数据步骤四 a inner join [MR数据].dbo.分频前频点方案 b on a.主小区CI=b.CI

)c

inner join

(

select a.主小区CI as 主小区CI,a.邻小区CI as 邻小区CI,b.分频前频点 as 邻小区分频前频点

from [MR数据].dbo.MR数据步骤四 a inner join [MR数据].dbo.分频前频点方案 b on a.邻小区CI=b.CI

)d

on c.主小区CI=d.主小区CI and c.邻小区CI=d.邻小区CI

where (c.主小区分频前频点-d.邻小区分频前频点)=0 or abs(c.主小区分频前频点-d.邻小区分频前频点)=1

----退频后频点方案带入计算干扰情况---

SELECT [对象名称],[AS362],[S360],[S361],[S362],[S363],[S364],[S365],[S366],[S367],[S368],[S369],[S370],[S371],[BSCI],[BCCH]

  ,b.[主小区CI]

  ,[主小区经度]

  ,[主小区纬度]

  ,b.[邻小区CI]

  ,[邻小区经度]

  ,[邻小区纬度]

  ,[距离]

  ,a.同频数

  ,a.邻频数

  from
(

select 主小区CI,邻小区CI,

count(case 差值 when '0' then 邻小区CI end) as 同频数,

count(case abs(差值) when '1' then 邻小区CI end) as 邻频数

from [MR数据].dbo.MR数据步骤分频后

group by 主小区CI,邻小区CI

) a right join [MR数据].[dbo].[MR数据步骤四] b on b.主小区CI=a.主小区CI and b.邻小区CI=a.邻小区CI

----分频前频点方案计算干扰值---

SELECT [对象名称],[AS362],[S360],[S361],[S362],[S363],[S364],[S365],[S366],[S367],[S368],[S369],[S370],[S371]

  ,[BSCI]

  ,[BCCH]

  ,b.[主小区CI]

  ,[主小区经度]

  ,[主小区纬度]

  ,b.[邻小区CI]

  ,[邻小区经度]

  ,[邻小区纬度]

  ,[距离]

  ,a.同频数

  ,a.邻频数

  from
(

select 主小区CI,邻小区CI,

count(case 差值 when '0' then 邻小区CI end) as 同频数,

count(case abs(差值) when '1' then 邻小区CI end) as 邻频数

from [MR数据].dbo.MR数据步骤分频前

group by 主小区CI,邻小区CI

) a right join [MR数据].[dbo].[MR数据步骤四] b on b.主小区CI=a.主小区CI and b.邻小区CI=a.邻小区CI
