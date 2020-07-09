У вас бывало такое на сервере Confluence закончилось место, а вы не знаете кто больше всего загружает вложений?

Чтобы узнать это необходим доступ к в бд confluence.

С помощью SQL запросом можно узнать полезную статистику по Confluence.

### Общий размер всех вложений в Confluence:

```sql
SELECT s.spaceid,
       s.spacename,
       sum(LONGVAL)
FROM contentproperties c
JOIN content co ON c.contentid = co.contentid
JOIN spaces s ON co.spaceid = s.spaceid
WHERE c.contentid IN
    (SELECT contentid
     FROM content
     WHERE contenttype = 'ATTACHMENT')
  AND c.propertyname = 'FILESIZE'
GROUP BY s.spaceid
ORDER BY SUM DESC
LIMIT 5;
```

Вывод (spacename заменил):

![](https://habrastorage.org/webt/pc/dp/m8/pcdpm8rk3wsr6sj5wqvn3lrtog8.jpeg)

### Cтраницы с большинством исторических версий в стечении:

```sql
SELECT title,
       MAX(VERSION)
FROM content
WHERE contenttype = 'PAGE'
GROUP BY title
ORDER BY 2 DESC
LIMIT 5;
```
Вывод:

![](https://habrastorage.org/webt/-n/zs/xd/-nzsxd2rxvb6ch8jme-z4kkc2lc.jpeg)

### Cамые большие файлы вложений в вашем экземпляре Confluence

```sql
SELECT DISTINCT c.contentid,
                c.title AS attachmentTitle,
                u.username AS uploadedBy,
                co.title AS pageTitle,
                cn.longval AS bytes
FROM CONTENT AS c
JOIN USER_MAPPING AS u ON u.user_key = c.creator
JOIN CONTENT AS co ON c.pageid = co.contentid
JOIN CONTENTPROPERTIES AS cn ON cn.contentid = c.contentid
WHERE c.contenttype = 'ATTACHMENT'
  AND cn.longval IS NOT NULL
ORDER BY cn.longval DESC
LIMIT 5;
```

Вывод:

![](https://habrastorage.org/webt/qh/3e/wp/qh3ewpuzbpzsfyowujsfi5smdks.jpeg)

### Количество страниц в корзине и общий размер страниц в корзине на пространство:

```sql

SELECT Count(content.contentid) AS number_of_trashed_pages,
       Pg_size_pretty(SUM(Pg_column_size(bodycontent.BODY))) AS trash_total_size,
       spaces.spacename AS space_name
FROM bodycontent
INNER JOIN content ON (content.contentid = bodycontent.contentid)
INNER JOIN spaces ON (content.spaceid = spaces.spaceid)
WHERE bodycontent.contentid IN
    (SELECT contentid
     FROM content
     WHERE content_status = 'deleted'
       AND contenttype = 'PAGE')
GROUP BY space_name
ORDER BY trash_total_size
LIMIT 5;
```

Вывод:

![](https://habrastorage.org/webt/mg/et/n2/mgetn2a3ofzh8pctwkei1naoj9s.jpeg)

### Общий размер вложений, загруженных каждым пользователем на всех страницах

```sql
SELECT u.lower_username,
       sum(cp.longval) AS "size"
FROM content c1
JOIN content c2 ON c1.contentid = c2.pageid
JOIN user_mapping u ON c1.creator=u.user_key
JOIN contentproperties cp ON c2.contentid = cp.contentid
WHERE c2.contenttype='ATTACHMENT'
GROUP BY u.lower_username
ORDER BY sum(cp.longval) DESC
LIMIT 5;
```

Вывод:

![](https://habrastorage.org/webt/ts/jz/hb/tsjzhbbmzk0vcf0a2khqpljlx90.jpeg)