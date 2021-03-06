# Домашнее задание к занятию "6.1. Типы и структура СУБД"

## Введение

Перед выполнением задания вы можете ознакомиться с 
[дополнительными материалами](https://github.com/netology-code/virt-homeworks/tree/master/additional/README.md).

## Задача 1

Архитектор ПО решил проконсультироваться у вас, какой тип БД 
лучше выбрать для хранения определенных данных.

Он вам предоставил следующие типы сущностей, которые нужно будет хранить в БД:

- Электронные чеки в json виде
- Склады и автомобильные дороги для логистической компании
- Генеалогические деревья
- Кэш идентификаторов клиентов с ограниченным временем жизни для движка аутенфикации
- Отношения клиент-покупка для интернет-магазина

Выберите подходящие типы СУБД для каждой сущности и объясните свой выбор.

## Решение

1. документо-ориентированная БД - чек - финансовый документ для хранения информации о совершенной сделке. MongoDB. У нас на работе храняться в реляционной базе postgres в json как отдельный атребут.
2. графовой БД - ребра между складами(узлами) и есть дороги, узел - склад, свойства - описание складов.
3. иерархическая БД - один отец от которого пошел род.
4. БД ключ-значение - нам важно быстрый доступ к данным, по конкретному указателю, ключ, все остальное не интересно. Подойдет redis, tarantool.
5. реляционные БД - в данном случае нам важно хранить данные о клиенте, о том, что покупал, какое кол-во, 
в какой период, в таком случае мы можем получать статистику и аналитику по людям. postgres, mysql.


## Задача 2

Вы создали распределенное высоконагруженное приложение и хотите классифицировать его согласно 
CAP-теореме. Какой классификации по CAP-теореме соответствует ваша система, если 
(каждый пункт - это отдельная реализация вашей системы и для каждого пункта надо привести классификацию):

- Данные записываются на все узлы с задержкой до часа (асинхронная запись)
- При сетевых сбоях, система может разделиться на 2 раздельных кластера
- Система может не прислать корректный ответ или сбросить соединение

А согласно PACELC-теореме, как бы вы классифицировали данные реализации?

## Решение

1. CA | PC/EL
2. PA | PA/EL
3. PC | PA/EC

## Задача 3

Могут ли в одной системе сочетаться принципы BASE и ACID? Почему?

## Решение

Принцыпы BASE и ACID сочетаться не могут. По ACID - данные согласованные, а по BASE - могут быть неверные, следовательно они противоречат друг другу.
ACID позволяет проектировать высоконадежные системы. BASE позволяет проектировать высокопроизводительные системы.

## Задача 4

Вам дали задачу написать системное решение, основой которого бы послужили:

- фиксация некоторых значений с временем жизни
- реакция на истечение таймаута

Вы слышали о key-value хранилище, которое имеет механизм [Pub/Sub](https://habr.com/ru/post/278237/). 
Что это за система? Какие минусы выбора данной системы?

## Решение

В Redis данным можно присваивать Time-To-Live. Данная функция приминяется при менее требовательном хранении данных. 
По истечению TTL данные безвозвратно удаляются.

Redis как раз является key-value хранилищем, обычно используется для хранения сессий пользователей, т.е. не сильно критичных данных.
Это своего рода система брокера сообщений, очередей. После того как сообщение было прочитано оно либо, может быть удаленно, либо отсчитывается TTL.
Как плюс большая скорость чтения и записи данных, минус более низкая сохранность данных и соответственно дороговизна, так как данные храняться в оперативной памяти.



