#### 牛客-SQL篇-SQL大厂笔试真题



###### 1. 每个月Top3的周杰伦歌曲

从听歌流水中找到18-25岁用户在2022年每个月播放次数top 3的周杰伦的歌曲

输入：

```
drop table if exists play_log;
create table `play_log` (
    `fdate` date,
    `user_id` int,
    `song_id` int
);
insert into play_log(fdate, user_id, song_id)
values 
('2022-01-08', 10000, 0),
('2022-01-16', 10000, 0),
('2022-01-20', 10000, 0),
('2022-01-25', 10000, 0),
('2022-01-02', 10000, 1),
('2022-01-12', 10000, 1),
('2022-01-13', 10000, 1),
('2022-01-14', 10000, 1),
('2022-01-10', 10000, 2),
('2022-01-11', 10000, 3),
('2022-01-16', 10000, 3),
('2022-01-11', 10000, 4),
('2022-01-27', 10000, 4),
('2022-02-05', 10000, 0),
('2022-02-19', 10000, 0),
('2022-02-07', 10000, 1),
('2022-02-27', 10000, 2),
('2022-02-25', 10000, 3),
('2022-02-03', 10000, 4),
('2022-02-16', 10000, 4);

drop table if exists song_info;
create table `song_info` (
    `song_id` int,
    `song_name` varchar(255),
    `singer_name` varchar(255)
);
insert into song_info(song_id, song_name, singer_name) 
values
(0, '明明就', '周杰伦'),
(1, '说好的幸福呢', '周杰伦'),
(2, '江南', '林俊杰'),
(3, '大笨钟', '周杰伦'),
(4, '黑键', '林俊杰');

drop table if exists user_info;
create table `user_info` (
    `user_id`   int,
    `age`       int
);
insert into user_info(user_id, age) 
values
(10000, 18)
```

输出：

```
month|ranking|song_name|play_pv
1|1|明明就|4
1|2|说好的幸福呢|4
1|3|大笨钟|2
2|1|明明就|2
2|2|说好的幸福呢|1
2|3|大笨钟|1
```

说明：

```
1月被18-25岁用户播放次数最高的三首歌为“明明就”、“说好的幸福呢”、“大笨钟”，“明明就”和“说好的幸福呢”播放次数相同，排名先后由两者的song_id先后顺序决定。2月同理。
```

###### 代码

```mysql
with tmp as(
    select 
    month(log.fdate) month,
    row_number() over(partition by month(log.fdate) order by count(song.song_id) desc, song.song_id asc) ranking, 
    song.song_name song_name, 
    count(song.song_id) play_pv
    from play_log log
    inner join (
        select * from user_info where age between 18 and 25
    ) user on log.user_id = user.user_id
    inner join (
        select * from song_info where singer_name = "周杰伦"
    ) song on log.song_id = song.song_id
    where year(log.fdate) = 2022
    group by month(log.fdate), song.song_name, song.song_id
)
select * from tmp
where ranking <= 3

# 公用表表达式（CTE）:
# WITH tmp AS (...): 定义了一个名为tmp的临时表，这个临时表包含了查询的结果。
# 选择列和窗口函数:
# month(t1.fdate) as month: 提取t1.fdate字段的月份。
# row_number() over(partition by month(t1.fdate) order by count(t1.song_id) desc, t1.song_id asc) as ranking: 对每个月的歌曲播放次数进行排名，如果播放次数相同，则根据歌曲ID进行升序排列。
row_number()函数是一个窗口函数，它为每个分组内的行分配一个唯一的连续整数，从1开始。
窗口函数的计算范围通常由两部分组成：
PARTITION BY: 这部分用来指定如何将数据分成不同的组。每个组内的行会被视为一个单独的集合，窗口函数会在这些组内分别计算。如果没有指定PARTITION BY，则整个查询结果集被视为一个单一的组。
ORDER BY: 这部分用来指定组内行的排序方式。
```

###### 2 最长连续登录天数

你正在搭建一个用户活跃度的画像，其中一个与活跃度相关的特征是“最长连续登录天数”， 请用SQL实现“2023年1月1日-2023年1月31日用户最长的连续登录天数”

输入：

```
drop table if exists tb_dau;
create table `tb_dau` (
    `fdate` date,
    `user_id` int
);
insert into tb_dau(fdate, user_id)
values 
('2023-01-01', 10000),
('2023-01-02', 10000),
('2023-01-04', 10000);
```

输出：

```
user_id|max_consec_days
10000|2
```

说明：

```
id为10000的用户在1月1日及1月2日连续登录2日，1月4日登录1日，故最长连续登录天数为2日
```

```
示例：
如用户在1月3日-1月10日登录，且在1月20日-1月22日登录，则最长连续登录天数为8

MySQL中日期加减的函数
日期增加 DATE_ADD，例：date_add('2023-01-01', interval 1 day) 输出 '2023-01-02'
日期减少 DATE_SUB，例：date_add('2023-01-01', interval 1 day) 输出 '2022-12-31'
日期差 DATEDIFF，例：datediff('2023-02-01', '2023-01-01') 输出31
```

###### 代码

```mysql
select user_id, max(consec_days) max_consec_days
from (
    # 3.对连续时间进行count计数，取max即为答案
    select user_id, date, count(date) consec_days
    from (
        # 2.排序后时间 - 序列号num， 相同时间即为连续时间
        # '2023-01-01' - '1'  = '2022-12-31'
        # '2023-01-02' - '2'  = '2022-12-31'
        # '2023-01-04' - '3'  = '2023-01-01'
        select *, date_sub(fdate, interval num day) as date
        from (
            # 1.使用row_number()根据时间排序，分配序列号num
            select *, row_number() over(partition by user_id order by fdate) as num
            from tb_dau
            where fdate between '2023-01-01' and '2023-01-31'
        ) as t
        group by user_id, fdate
    ) t1
    group by user_id, date
) t2
group by user_id

```

