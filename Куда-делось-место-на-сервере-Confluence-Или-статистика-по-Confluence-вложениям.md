У вас бывало такое на сервере Confluence закончилось место, а вы не знаете кто больше всего загружает вложений?

Чтобы узнать это необходим доступ к в бд confluence.

С помощью SQL запросом можно узнать полезную статистику по Confluence.

### Общий размер всех вложений в Confluence:

``
select s.spaceid, s.spacename, sum(LONGVAL)from contentproperties c join content co on c.contentid = co.contentid join spaces s on co.spaceid = s.spaceid where c.contentid in (select contentid from content where contenttype = 'ATTACHMENT') and c.propertyname = 'FILESIZE' group by s.spaceid ORDER BY sum DESC limit 5;
``
Вывод (spacename заменил):

```
  spaceid  |                 spacename                  |     sum     
-----------+--------------------------------------------+-------------
 111869963 | spacename1                                 | 31800028643
  75104259 | spacename2                                 | 25357092405
 106594322 | spacename3                                 | 16488575453
  12615683 | spacename4                                 | 12859292546
 106594323 | spacename5                                 | 10700185628
```

### Cтраницы с большинством исторических версий в стечении:

```
select title,MAX(version) from content where contenttype = 'PAGE' group by title order by 2 desc limit 5;
```
Вывод:
```
            title            |  max  
-----------------------------+-------
 title1                      | 10530
 title2                      |  6754
 title3                      |  5372
 title4                      |  3860
 title5                      |  3624
```

### Cамые большие файлы вложений в вашем экземпляре Confluence

``
SELECT DISTINCT c.contentid, c.title AS attachmentTitle, u.username AS uploadedBy, co.title AS pageTitle, cn.longval as bytes FROM CONTENT AS c JOIN USER_MAPPING AS u ON u.user_key = c.creator JOIN CONTENT AS co ON c.pageid = co.contentid JOIN CONTENTPROPERTIES AS cn ON cn.contentid = c.contentid WHERE c.contenttype = 'ATTACHMENT' AND cn.longval IS NOT NULL  ORDER BY cn.longval DESC LIMIT 5;
``
Вывод:

```
 contentid |                       attachmenttitle                       |     uploadedby     |       pagetitle       |   bytes   
-----------+-------------------------------------------------------------+--------------------+-----------------------+-----------
 112234492 | attachmenttitle1 | uploadedby1   | pagetitle1  | 390969791
 112240558 | attachmenttitle2 | uploadedby2   | pagetitle2  | 385857558
 112254963 | attachmenttitle3 | uploadedby3   | pagetitle3  | 383218649
 103467629 | attachmenttitle4 | uploadedby4   | pagetitle4  | 370464296
 103467641 | attachmenttitle5 | uploadedby5   | pagetitle5  | 370175883
```

### Количество страниц в корзине и общий размер страниц в корзине на пространство:

``
SELECT Count(content.contentid) AS number_of_trashed_pages, Pg_size_pretty(SUM(Pg_column_size(bodycontent.BODY))) AS trash_total_size, spaces.spacename as space_name FROM bodycontent inner join content ON ( content.contentid = bodycontent.contentid ) inner join spaces ON ( content.spaceid = spaces.spaceid ) WHERE bodycontent.contentid IN (SELECT contentid FROM content WHERE content_status = 'deleted' AND contenttype = 'PAGE') GROUP BY space_name ORDER BY trash_total_size LIMIT 5;
``
Вывод:

```
 number_of_trashed_pages | trash_total_size |           space_name            
-------------------------+------------------+---------------------------------
                       5 | 1014 bytes       | space_name1
                      38 | 101 kB           | space_name2
                       5 | 1056 bytes       | space_name3
                       2 | 1093 bytes       | space_name4
                       9 | 10 kB            | space_name5
```

### Общий размер вложений, загруженных каждым пользователем на всех страницах

``
select u.lower_username, sum(cp.longval) as "size" from content c1 join content c2 on c1.contentid = c2.pageid join user_mapping u on c1.creator=u.user_key join contentproperties cp on c2.contentid = cp.contentid where c2.contenttype='ATTACHMENT' group by u.lower_username order by sum(cp.longval) desc LIMIT 5;
``
Вывод:

```
   lower_username   |    size     
--------------------+-------------
 username1          | 19282081239
 username2          | 16448214138
 username3          |  8561688622
 username4          |  6548428078
 username5          |  6058144588
```
