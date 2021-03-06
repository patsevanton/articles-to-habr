### Связь Many to many для базы данных во Flask

https://programmer.help/blogs/many-to-many-relationship-of-database-in-flask.html

Большинство других типов связей могут быть производными от одного до многих типов. С точки зрения «многих», отношение «многие к одному» - это отношение «один ко многим». Отношения «один к одному» - это упрощенная версия отношения «один ко многим». Единственный тип, который нельзя развить от отношений "один ко многим", - это отношения "многие ко многим".

- **Связь многие ко многим**

Связи «один ко многим», «многие к одному» и «один к одному» имеют по крайней мере одну сторону, которая является единым целым. Связь между таблицами реализуется с помощью внешних ключей, которые указывают на эту сущность. Чтобы решить проблему отношения «многие ко многим», нам нужно ввести третью таблицу, называемую таблицей ассоциаций, которую можно разложить на две взаимосвязи «один ко многим» между исходной таблицей и таблицей ассоциаций. Например, когда студенты выбирают курсы, один студент может выбрать несколько курсов, а один курс может быть выбран несколькими студентами. Это типичные отношения "многие ко многим".

![](https://programmer.help/images/blog/47fda377baa78674da54df48888276b7.jpg)

Процесс запроса связи "многие ко многим": например, чтобы узнать, какие курсы выбрал студент, сначала получите студента-студента из отношения "один ко многим" между студентом и регистрацией_ Все классы, соответствующие ID_ ID, а затем просмотрите Связь один ко многим между курсом и регистрацией в направлении «многие к одному», чтобы найти все курсы, выбранные студентом.
1. #### Связь "многие ко многим" между двумя организациями.

##### Flask Sqlalchemy реализует отношения многие ко многим

```
 1 registrations = db.Table('registrations', 
 2     db.Column('student_id', db.Integer, db.ForeignKey('students.id')),
 3     db.Column('class_id', db.Integer, db.ForeignKey('classes.id')))
 4 
 5 class Student(db.Model):
 6     __tablename__ = 'students'
 7     id = db.Column(db.Integer, primary_key=True)
 8     name = db.Column(db.String)
 9     classes = db.relationship('Class', secondary=registrations,
10                               backref = db.backref('students', lazy='dynamic'),
11                               lazy='dynamic')
12 
13 class Class(db.Model):
14     __tablename__ = 'classes'
15     id = db.Column(db.Integer, primary_key=True)
16     name = db.Column(db.String)
```



Все еще используется в определении метода db.relationship () во многих отношениях, но во многих отношениях вторичный параметр должен быть установлен как связанная таблица. Отношения "многие ко многим" можно определить в любом классе, а параметр backref обрабатывает другую сторону связи. Связанная таблица - это простая таблица, а не модель. Flask Sqlalchemy автоматически возьмет на себя управление таблицей.

Курс регистрации студентов:

```
1 stu.classes.append(c)
2 db.session.add(stu)
```

Перечислите все курсы, на которых обучаются студенты:

```
1 stu.classes.all()
```

Все студенты, зарегистрированные на Курс c:

```
1 c.students.all()
```

- **Cамореферентное отношение**

В функции «Пользовательский фокус» во взаимосвязи «многие ко многим» нет двух сущностей, только модель сущности «Пользователь». Если обе стороны отношения находятся в одной таблице, это отношение называется самореферентным отношением.

![](https://programmer.help/images/blog/7f129d6eb42da659f60539b3d8d0407f.jpg)

Связанная таблица следует, где каждая строка представляет одного пользователя, следующего за другим. Слева - последователь, которого можно понять как поклонник, справа - последователь, который можно понять как внимание к другим.
При использовании отношений "многие ко многим" необходимо сохранять дополнительную информацию между двумя объектами, например информацию о времени, на которую один пользователь обращает внимание на другого пользователя. Для обработки настроенных данных таблица ассоциаций может быть разработана как доступная модель.

```
 1 class Follow(db.Model):
 2     __tablename__ = 'follows'
 3     follower_id = db.Column(db.Integer, db.ForeignKey('users.id'), primary_key=True)
 4     followed_id = db.Column(db.Integer, db.ForeignKey('users.id'), primary_key=True)
 5     timestamp = db.Column(db.DateTime, default=datetime.utcnow)
 6     
 7 class User(UserMixin, db.Model):
 8     __tablename__ = 'users'
 9     ...
10     followers = db.relationship('Follow',
11                                foreign_keys=[Follow.followed_id],
12                                backref=db.backref('followed', lazy='joined'),
13                                lazy='dynamic',
14                                cascade='all, delete-orphan')
15     followed = db.relationship('Follow',
16                                foreign_keys=[Follow.follower_id],
17                                backref=db.backref('follower', lazy='joined'),
18                                lazy='dynamic',
19                                cascade='all, delete-orphan')
```



Введение параметра:
1. foreign_ Параметр keys представляет связанный внешний ключ
2. Параметр backref указывает на то, что подписанный / подписчик возвращается к модели отслеживания через Follow.follower Посетите поклонников через Follow.followed Посетите следующих людей
3. Если параметр отложенного вызова в обратном вызове указан как объединенный, связанные объекты могут быть загружены из запроса объединения немедленно.
4. Влияние операции настройки параметров каскада на родительский объект на связанные объекты

- **Вспомогательные методы отношения внимания**

```
 1 class User(UserMixin, db.Model):
 2     __tablename__ = 'users'
 3     ...
 4     
 5     # Follow a user
 6     def follow(self, user):
 7         # Judge if you have paid attention
 8         if not self.is_following(user):
 9             # Create a new association table object instance to record the relationship between fans and followers
10             f = Follow(follower=self, followed=user)
11             db.session.add(f)
12     
13     # Unfollow a user
14     def unfollow(self, user):
15         # Check if the user of the cancelled attention has been paid attention to
16         f = self.followed.filter_by(followed_id=user.id).first()
17         if f:
18             db.session.delete(f)
19     
20     # Judge whether you have paid attention
21     def is_following(self, user):
22         # Confirm whether the user has id，In case a user is created but not committed to the database
23         if user.id is None:
24             return False
25         return self.followed.filter_by(followed_id=user.id).first() is not None
26     
27     # Judge whether it is concerned
28     def is_followed_by(self, user):
29         if user.id is None:
30             return False
31         return self.followers.filter_by(follower_id=user.id).first() is not None
```

