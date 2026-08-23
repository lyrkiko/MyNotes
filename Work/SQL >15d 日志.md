```sql
SELECT
dw_fltdb.unzip_sgp(get_json_object(stored, '$.Request')) AS Request
        , dw_fltdb.unzip_sgp(get_json_object(stored, '$.Response')) AS Response
  FROM dw_fltlogdata.fx_cat_log_flight_bff
 WHERE d in ('2026-06-03') 
      AND h BETWEEN '' AND ''
      AND servicetype = 'MobileService'
      and servicecode='11100803'
       and UID='_TIKRufjo8hhvuce'

查看建表语句

SHOW CREATE TABLE dw_fltlogdata.fx_cat_log_flight_bff;

```



1. 报文时间是用户的时间  [NO！是转换过后的时间]
2. 下载文件是csv
3. 如果未指定精确时间，如何正确提取对应的航班信息报文？
