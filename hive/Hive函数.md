## Hive函数

```sql
conact_ws(',', collect_list(user_id))

case order_dow when '0' then 1 else 0 end
Select case 100 when 50 then 'tom' when 100 then 'mary'else 'tim' end from lxw_dual;

row_number() over (partition by user_id order by prod_cnt desc) rank

regexp_replace(sentence, ' ', '')
select regexp_extract('foothebar', 'foo(.*?)(bar)', 1) fromlxw_dual;

-- url解析
-- 语法：parse_url(string urlString, string partToExtract [, stringkeyToExtract])
-- 返回URL中指定的部分。partToExtract的有效值为：HOST, PATH, QUERY, REF, PROTOCOL, AUTHORITY, FILE, and USERINFO.
select parse_url('http://facebook.com/path1/p.php?k1=v1&k2=v2#Ref1', 'HOST') from article;
select parse_url('http://facebook.com/path1/p.php?k1=v1&k2=v2#Ref1', 'QUERY','k1') from article;
```



参考：

[博客1](https://blog.csdn.net/wisgood/article/details/17376393)

