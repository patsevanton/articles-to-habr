# Лабораторная работа 7b

## Политики на основе тегов (интеграция с Atlas + Ranger)

Цель: в этой лабораторной работе мы рассмотрим, как интегрируются Atlas и Ranger для улучшения доступа к данным и авторизации с помощью тегов.

#### Подготовка Атласа

Чтобы создать политики на основе тегов, нам сначала нужно будет создать теги в Атласе и связать их.

- Перейдите по адресу https: // localhost: 21000 и войдите в пользовательский интерфейс Atlas, используя admin / admin в качестве имени пользователя и пароля
![](https://habrastorage.org/webt/3l/j6/bc/3lj6bcllbqdpqs9abilnc6ehmsm.png)
- Выберите вкладку «TAGS» и нажмите «Create Tag».
![](https://habrastorage.org/webt/-4/vc/oj/-4vcojmdhzjd8p91igbinorp1tu.png)
- Создайте новый тег, введя
	- Name: `Private`
	- Нажмите Create
![](https://habrastorage.org/webt/yi/pl/wo/yiplwofj9cphpr7ayxr_8lcpjjc.png)
- Повторите процесс создания тега выше и создайте дополнительный тег с именем "Restricted".
- Создайте третий тег с именем «Sensitive», однако во время создания нажмите «Add New Attributes» и введите:
  - Attribute Name: `level`
  - Type: `int`
  ![](https://habrastorage.org/webt/e0/28/cy/e028cybz7dl4eu3auqzhj5_qt8i.png)
- Создайте четвертый тег с именем «EXPIRES_ON» и во время создания нажмите «Add New Attributes» и введите:
	- Attribute Name: `expiry_date`
	- Type: `int`
![](https://habrastorage.org/webt/lk/5z/g2/lk5zg2vcgvzcnmjofya7z6k596s.png)
- На вкладке «Tags» на главном экране вы должны увидеть список вновь созданных тегов.
![](https://habrastorage.org/webt/dl/rd/xl/dlrdxltlexcrreb6_x7dwpotmuc.png)
- На вкладке поиска выполните поиск, используя следующие данные:
	- Search By Type: `hive_table`
	- Search By Text: `sample_08`
	- Нажмите Search
![](https://habrastorage.org/webt/gx/cl/gk/gxclgkuying6mosskhwz0m7gc8m.png)
- Чтобы связать тег с таблицей «sample_08», нажмите «+» в столбце «Tags» в результатах поиска для «sample_08».
![](https://habrastorage.org/webt/lc/ih/qc/lcihqcex7f1jghgrmpyfamlrkow.png)
- В раскрывающемся списке выберите `Private` и нажмите `Add`
![](https://habrastorage.org/webt/_o/_b/ic/_o_bic0f-myk6ti3rwgn1r85q7s.png)
- Вы должны увидеть, что тег "Private" был связан с таблицей sample_08.
![](https://habrastorage.org/webt/hh/h_/3l/hhh_3l_n-mpur7ymdqfet5mj0fo.png)
- Теперь таким же образом свяжите тег «EXPIRES_ON» с таблицей «sample_08».
	При появлении запроса выберите дату в прошлом для "expiry_date".![](https://habrastorage.org/webt/5s/sj/v-/5ssjv-yreo2svy_a1etybgjorn4.png)
- На панели результатов поиска нажмите ссылку «sample_08».
![](https://habrastorage.org/webt/jb/fc/ud/jbfcudgisyoe0lybbjpylowxw54.png)
- Прокрутите вниз и выберите вкладку «Schema».
![](https://habrastorage.org/webt/5k/2k/sz/5k2ksz6o681tn2bd4tncsl_vj7w.png)
- Нажмите кнопку «+» под столбцом «Tag» для «salary» и свяжите с ним тег «Restricted».
- Нажмите кнопку «+» под столбцом «Tag» для «total_emp» и свяжите с ним тег «Sensitive».
   - При появлении запроса введите «5» для «level».
- ![](https://habrastorage.org/webt/bw/nc/x6/bwncx6qcjdd4gzdmx8u_6vy_2da.png)
- На странице schema таблицы sample_08 вы должны увидеть столбцы таблицы со связанными тегами.
![](https://habrastorage.org/webt/pz/3c/bq/pz3cbqyfhj6g9vjb22mbgdw1dw4.png)

Мы завершили подготовительную работу в Атласе и создали следующие теги и ассоциации:
- Тег "Private", связанный с таблицей sample_08
- Тег «Sensitive» с «уровнем» 5, связанный со столбцом «sample_08.total_emp»
- Тег «Restricted», связанный со столбцом «sample_08.salary»

#### Подготовка Ranger 

Чтобы включить Ranger для политик на основе тегов, выполните следующие действия:

- Выберите «Access Manager», а затем «Tag Based Policies» в верхнем левом углу главной страницы пользовательского интерфейса Ranger.

![](https://habrastorage.org/webt/vu/p3/my/vup3mynbagb1ns6p7pvjtmdwvls.png)

- Нажмите "+", чтобы создать новую службу тегов.
![](https://habrastorage.org/webt/3x/h0/0d/3xh00dllyqikk0wh8iirwn7qldy.png)
- На странице сервиса тегов введите:
	- Service Name: `tags`
	- Наажмите Save
![](https://habrastorage.org/webt/vg/1d/y6/vg1dy6pe-huvoxxsheqfsv41tti.png)
- На главной странице пользовательского интерфейса Ranger нажмите кнопку редактирования рядом с вашей службой (имя кластера) _hive.
![](https://habrastorage.org/webt/s9/ag/zf/s9agzfue3gduffsugg8muz2s-kc.png)
- На странице редактирования службы добавьте нижеприведенное и нажмите "Сохранить".
	- Select Tag Service: `tags`
![](https://habrastorage.org/webt/jb/v5/4c/jbv54cwb-zqfaprusgr8t_yskw0.png)

- Теперь у нас должна быть возможность создавать политики на основе тегов для Hive.

  #### Контроль доступа на основе тегов

  Цель: создать политику продаж на основе тегов для доступа ко всем объектам, помеченным как "Private".

  - Выберите «Access Manager», а затем «Tag Based Policies» в верхнем левом углу главной страницы пользовательского интерфейса Ranger.

  ![](https://habrastorage.org/webt/wt/lr/zw/wtlrzwcf273qkpbyulpqz7snqby.png)

- Выберите службу «tags», которую вы создали ранее.

- На странице «tags Policies» нажмите «Add New Policy».![](https://habrastorage.org/webt/1h/8q/w1/1h8qw14sh_pnmjawqkg_n8yxb0m.png)

- Введите следующее, чтобы создать новую политику
	- Policy Name: `Private Data Access`
	- TAG: `Private`
	- В разделе "Allow Conditions"
		- Select Group: `sales`
		- Component Permissions: (выберите `Hive` и включите все действия)
	- Нажмите Add
![](https://habrastorage.org/webt/u8/lb/ux/u8lbuxkgra9wri1qirwaumiqflg.png)
![Image](/screenshots/Ranger-Tags-component-permissions.png)

- Выполните эти шаги с узла, на котором установлен Hive (или клиент).

- С узла, на котором установлен Hive (или клиент), войдите в систему как sales1 и подключитесь к beeline:
```
su - sales1
klist
## Default principal: sales1@LAB.HORTONWORKS.NET
```
```
beeline -u "jdbc:hive2://localhost:10000/default;principal=hive/$(hostname -f)@LAB.HORTONWORKS.NET"
```
- Теперь попробуйте получить доступ к таблице sample_08 и обратите внимание, что у вас есть доступ ко всему содержимому таблицы.

  beeline> select * from sample_08;
#### Атрибутный контроль доступа
Цель: запретить всем доступ к данным, помеченным как "Sensitive" и имеющим атрибут "уровень" 5 или выше.

- Вернитесь на страницу «tags Policies» Ranger и «Add New Policy» со следующими параметрами.
	- Policy Name: `Sensitive Data Access`
	- TAG: `Sensitive`
	- В разделе "Deny Conditions" 
		- Select Group: `public`
		- Policy Conditions/Enter boolean expression: `level>=5`
			- Примечание: логические выражения написаны на Javascript.
		- Component Permissions: (выберите `Hive` и включите все действия)
	- Нажмите Add
![](https://habrastorage.org/webt/0m/bn/or/0mbnor5kyqk9bldbbvmkq9g78da.png)
![](https://habrastorage.org/webt/r2/yj/mr/r2yjmrhjweluyoc8k7w_unyeu-a.png)

- Подождите 30 секунд, прежде чем пытаться получить доступ к столбцу «total_emp» в таблице «sample_08» и обратите внимание, что вам отказано в доступе.
```
beeline> select total_emp from sample_08;
```

- Теперь попробуйте получить доступ к другим столбцам и обратите внимание, что вам разрешен доступ к ним.beeline> select code, description, salary from sample_08;
#### Маскирование на основе тегов
Цель: замаскировать данные с пометкой "Restricted"

- Вернитесь на страницу «tags Policies» Ranger, щелкните вкладку «Masking» в правом верхнем углу. 
![](https://habrastorage.org/webt/pj/ap/tc/pjaptcjk34tkmzex654blvde-le.png)

- Нажмите «Add New Policy» и введите следующие параметры.
	- Policy Name: `Restricted Data Access`
	- TAG: `Restricted`
	- Under "Mask Conditions" 
		- Select Group: `public`
		- Component Permissions: (выберите `Hive` и включите все действия)
		- Select Masking Option: `Redact`	
	- Нажмите Add

- Подождите 30 секунд и попробуйте выполнить запрос ниже. Обратите внимание, что  данные о зарплате стали замаскироваными
```
beeline> select code, description, salary from sample_08;
```

#### Контроль доступа на основе местоположения
Цель: ограничить доступ к данным в зависимости от физического местоположения пользователя в данный момент.

- Вернитесь на страницу «tags Policies» Ranger (вкладка «Access») и «Add New Policy» со следующими параметрами.
	- Policy Name: `Geo-Location Access`
	- TAG: `Restricted`
	- Во вкладке "Deny Conditions" 
		- Select Group: `public`
		- Policy Conditions/Enter boolean expression: `country_code=='USA'`
			- Если вы находитесь за пределами США, используйте вместо этого `country_code! = 'USA' '
		- Component Permissions: (выберите `Hive` и включите все действия)
	- Нажмите Add
![](https://habrastorage.org/webt/i3/78/br/i378brosnrapojxhldutbufgusm.png)
![](https://habrastorage.org/webt/wt/py/pa/wtpypau-tkgz0whnhmmhung3z0q.png)

- Подождите 30 секунд и попробуйте выполнить запрос ниже. Обратите внимание, что теперь вам запрещен доступ к столбцу "salary" из-за вашего местоположения.
```
beeline> select code, description, salary from sample_08;
```

#### Политики на основе времени
Цель: установить дату истечения срока действия политики доступа продаж к данным, помеченным как "Private", после чего доступ будет запрещен.

- Вернитесь на страницу «tags Policies» Ranger (вкладка «Access») и «Add New Policy» со следующими параметрами. Возможно, у вас уже есть политика по умолчанию с именем «EXPIRES_ON», в этом случае удалите ее, прежде чем нажимать «Add New Policy».
	- Policy Name: `EXPIRES_ON`
	- TAG: `EXPIRES_ON`
	- Under "Deny Conditions" 
		- Select Group: `public`
		- Policy Conditions/Accessed after expiry_date: `yes`
		- Component Permissions: (выберите `Hive` и включите все действия)
	- Нажмите Add
![](https://habrastorage.org/webt/h3/-4/wn/h3-4wnmuempv9io3it5l2zer_tu.png)
![](https://habrastorage.org/webt/ci/uh/jj/ciuhjj1_2j9zy0yfa7wu8sa31ca.png)

- Подождите 30 секунд и попробуйте выполнить запрос ниже. Обратите внимание, что теперь вам отказано в доступе ко всей таблице sample_08, потому что доступ к ней осуществляется после даты истечения срока, отмеченной в Атласе.
```
beeline> select code, description from sample_08;
```

- Выйдите из beeline
```
!q
```
- Разлогиньтесь как sales1
```
logout
```

## Оценка политики и приоритетности

Обратите внимание, как в приведенных выше политиках политики, запрещающие доступ, всегда имеют приоритет над политиками, разрешающими доступ. Например, несмотря на то, что отдел продаж имел доступ к «частным» данным в разделе «Контроль доступа на основе тегов», им постепенно запрещался доступ к следующим разделам, когда мы настраивали политику «Запретить». Это относится как к политикам, основанным на тегах, так и к политикам на основе ресурсов. Чтобы лучше понять последовательность оценки политики, взгляните на следующую блок-схему.
![](https://habrastorage.org/webt/3w/xv/aa/3wxvaas-ulfqmkv2tfvqnqnmhdu.png)

------------------

# Lab 8























