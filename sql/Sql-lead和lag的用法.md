# Sql-lead和lag的用法

现有表如下结构：

| user_id          | is_acct | fee  | flux | dura | part_month |
| :--------------- | :------ | ---- | ---- | ---- | ---------- |
| 8317102097526844 | 1       | 35   | 100  | 152  | 201812     |
| 8317102097526844 | 1       | 24   | 120  | 140  | 201901     |
| 8317102097526844 | 0       | 36   | 300  | 169  | 201902     |

| 用户ID  | 是否出账 | 出账金额 | 使用流量 | 使用语音 | 月份       |
| ------- | -------- | -------- | -------- | -------- | ---------- |
| user_id | is_acct  | fee      | flux     | dura     | part_month |

现有需求如下：

| 时间   | 流失用户数 | 流失用户拍照收入 | 流失用户拍照月使用流量 | 流失用户拍照月使用语音 | 流失用户前一月牌照收入 | 流失用户前一月使用流量 | 流失用户前一月使用语音 |
| ------ | ---------- | ---------------- | ---------------------- | ---------------------- | ---------------------- | ---------------------- | ---------------------- |
| 201901 |            |                  |                        |                        |                        |                        |                        |
| 201902 |            |                  |                        |                        |                        |                        |                        |
| 201903 |            |                  |                        |                        |                        |                        |                        |

- [ ] [流失用户]()：上月出账本月未出账用户为流失用户



此时则可以使用[Lead()]()[和Lag()]()函数：

LEAD(col,n,DEFAULT) 用于统计窗口内往下第n行值

> 第一个参数为列名，
> 第二个参数为往下第n行（可选，默认为1），
> 第三个参数为默认值（当往下第n行为NULL时候，取默认值，如不指定，则为NULL）

LAG(col,n,DEFAULT) 用于统计窗口内往上第n行值

> 第一个参数为列名，
> 第二个参数为往上第n行（可选，默认为1），
> 第三个参数为默认值（当往上第n行为NULL时候，取默认值，如不指定，则为NULL）



以下为实际开发代码：

```sql
create table temp_all_innet_users
as
with temp_1 as (
select user_id,month_id,max(total_dura) total_dura from report.dwa_v_m_cus_cb_sing_voice_u_tm group by user_id,month_id
),
temp_2 as (
select user_no,max(profession_line)  profession_line from report.dw_m_user_grid_list_all_tm where acct_month='201908' group by user_no
),
temp_3 as (
select user_id from depecc1.wwc_4g_user_acct_info s 
inner join depecc1.dim_product_2i dim on s.product_id = dim.product_id
where s.part_month='201812' 
  and s.is_acct=1
  and s.is_innet=1
)
select 
	t.user_id,
	t.is_acct,
	t.mon_total_fee fee,
	t.flux,
	t.sms,
	t1.mon_total_fee pz_fee,
	t1.flux	pz_flux,
	t1.dura	pz_dura,
	round(nvl(t2.total_dura,0)/60,2) dura,
	case when t3.profession_line='09' then 1 else 0 end online_flag,
	t.part_month
from depecc1.wwc_4g_user_acct_info t
inner join temp_1 t1 on t.user_id = s.user_id
left join tmep_2 t2 on t.user_id = t2.user_id and t.part_month= t2.month_id
left join tmep_3 t3 on t.user_id = t3.user_no
where t.part_month >='201812';

select count(case when is_acct=1 and is_acct1=0 then 1 else null end),  
       sum(case when is_acct=1 and is_acct1=0 then pz_fee else 0 end),  
       sum(case when is_acct=1 and is_acct1=0 then fee1 else 0 end),  
       sum(case when is_acct=1 and is_acct1=0 then flux else 0 end),  
       sum(case when is_acct=1 and is_acct1=0 then dura else 0 end),  
       sum(case when is_acct=1 and is_acct1=0 then pz_fee else 0 end),  
       sum(case when is_acct=1 and is_acct1=0 then pz_flux else 0 end),  
       sum(case when is_acct=1 and is_acct1=0 then pz_dura else 0 end),  
       count(case when part_month2='201908' and is_acct=1 and is_acct2=0 then 1 else null end),  
       sum(case when part_month2='201908' and is_acct=1 and is_acct2=0 then pz_fee else 0 end),  
       sum(case when part_month2='201908' and is_acct=1 and is_acct2=0 then pz_flux else 0 end),  
       sum(case when part_month2='201908' and is_acct=1 and is_acct2=0 then pz_dura else 0 end),  
       part_month,
       online_flag
from (
select t.user_id,
       nvl(is_acct,0) is_acct,
       nvl(fee,0) fee,
       nvl(flux,0) flux,
       nvl(dura,0) dura,
       pz_fee,
       pz_flux,
       pz_dura,
       online_flag,
       part_month,
       lead(nvl(is_acct,0),1,0) over(partition by user_id order by part_month) is_acct1,
       lead(nvl(fee,0),1,0) over(partition by user_id order by part_month) fee1,
       lead(nvl(flux,0),1,0) over(partition by user_id order by part_month) flux1,
       lead(nvl(dura,0),1,0) over(partition by user_id order by part_month) dura1,
       lead(nvl(is_acct,0),8,0) over(partition by user_id order by part_month) is_acct2,
       lead(nvl(fee,0),8,0) over(partition by user_id order by part_month) fee2,
       lead(nvl(flux,0),8,0) over(partition by user_id order by part_month) flux2,
       lead(nvl(dura,0),8,0) over(partition by user_id order by part_month) dura2,
       lead(nvl(part_month,0),8,0) over(partition by user_id order by part_month) part_month2
from temp_all_innet_users t
) t1
group by part_month,online_flag

```



