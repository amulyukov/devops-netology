# Домашнее задание к занятию "6.6. Troubleshooting"

## Задача 1

Перед выполнением задания ознакомьтесь с документацией по [администрированию MongoDB](https://docs.mongodb.com/manual/administration/).

Пользователь (разработчик) написал в канал поддержки, что у него уже 3 минуты происходит CRUD операция в MongoDB и её 
нужно прервать. 

Вы как инженер поддержки решили произвести данную операцию:
- напишите список операций, которые вы будете производить для остановки запроса пользователя
- предложите вариант решения проблемы с долгими (зависающими) запросами в MongoDB

## Решение
```
$currentOp (aggregation) - для поиска активных или неактивных подключений
db.killOp(opid) - для остановки операции(запроса) CRUD
$maxTimeMS -  задает совокупное ограничение по времени в миллисекундах для операций обработки курсора.
```
## Задача 2

Перед выполнением задания познакомьтесь с документацией по [Redis latency troobleshooting](https://redis.io/topics/latency).

Вы запустили инстанс Redis для использования совместно с сервисом, который использует механизм TTL. 
Причем отношение количества записанных key-value значений к количеству истёкших значений есть величина постоянная и
увеличивается пропорционально количеству реплик сервиса. 

При масштабировании сервиса до N реплик вы увидели, что:
- сначала рост отношения записанных значений к истекшим
- Redis блокирует операции записи

Как вы думаете, в чем может быть проблема?

## Решение

Redis удаляет ключи с истекшим сроком действия двумя способами:
Ленивый способ(lazy way) - истекает срок действия ключа, когда он запрашивается командой, но оказывается, что срок его действия уже истек.
Активного способ(active way) - истекает срок действия нескольких ключей каждые 100 миллисекунд.

Однако активный алгоритм является адаптивным и будет зацикливаться, если он обнаружит, что более 25% ключей уже истекли в наборе выбранных ключей. 
Но учитывая, что мы запускаем алгоритм десять раз в секунду, это означает,
что неудачное событие для более чем 25% ключей в нашей случайной выборке истекает по крайней мере в ту же секунду.

По сути при активном способе, если в базе данных много-много ключей, срок действия которых истекает в одну и ту же секунду, 
и они составляют не менее 25% от текущей совокупности ключей с установленным сроком действия, 
Redis может заблокировать, чтобы получить процент ключей, срок действия которых уже истек, ниже 25%.

В нашем случае при увиличении количества реплик соответственно увеличивается количество записей и соответственно количество просроченных ключей(записей)
 
## Задача 3

Вы подняли базу данных MySQL для использования в гис-системе. При росте количества записей, в таблицах базы,
пользователи начали жаловаться на ошибки вида:
```python
InterfaceError: (InterfaceError) 2013: Lost connection to MySQL server during query u'SELECT..... '
```

Как вы думаете, почему это начало происходить и как локализовать проблему?

Какие пути решения данной проблемы вы можете предложить?

## Решение

Обычно данная ошибка указывает на проблемы с подключением к сети, если эта ошибка возникает часто, 
следует проверить состояние сети. Если сообщение об ошибке содержит “во время запроса(during query)”, 
вероятно, это тот случай, с которым вы столкнулись. 

Иногда форма “во время запроса(during query)” возникает, когда миллионы строк отправляются как часть одного или 
нескольких запросов. Если известно, что это происходит, следует попробовать увеличить 
net_read_timeout с 30 секунд по умолчанию до 60 секунд или дольше, что достаточно для завершения передачи данных.

Реже данная ошибка может произойти, когда клиент пытается установить первоначальное соединение с сервером. 
В этом случае, если для нашего значения connect_timeout установлено значение всего несколько секунд, 
вы можете решить проблему, увеличив его до десяти секунд, возможно и больше, если у нас очень большое
расстояние или медленное соединение. Вы можете определить, сталкиваетесь ли вы с этой более необычной 
причиной, используя SHOW GLOBAL STATUS LIKE 'Aborted_connects'. Он увеличивается на единицу 
при каждой первоначальной попытке подключения, которую прерывает сервер. Если видим 
“reading authorization packet”  как часть сообщения об ошибке, то это также говорит 
о том, что это именно то решение, которое вам нужно.

Если причина не является ни одной из только что описанных, возможно, у нас возникла проблема 
со значениями больших двоичных объектов(BLOB values), которые больше, чем max_allowed_packet, что может 
вызвать эту ошибку у некоторых клиентов. Иногда вы можете увидеть ошибку ER_NET_PACKET_TOO_LARGE, 
это так же подтверждает, что вам нужно увеличить max_allowed_packet.

## Задача 4


Вы решили перевести гис-систему из задачи 3 на PostgreSQL, так как прочитали в документации, что эта СУБД работает с 
большим объемом данных лучше, чем MySQL.

После запуска пользователи начали жаловаться, что СУБД время от времени становится недоступной. В dmesg вы видите, что:

`postmaster invoked oom-killer`

Как вы думаете, что происходит?

Как бы вы решили данную проблему?

## Решение

Это указывает на то, что процесс postgres был завершен из-за нехватки памяти. 
Поведение виртуальной памяти по умолчанию в Linux не является оптимальным для PostgreSQL. 
Из-за того, как ядро реализует перегрузку памяти, ядро может завершить PostgreSQL postmaster (процесс сервера супервизора), 
если требования к памяти PostgreSQL или другого процесса приводят к тому, что в системе заканчивается виртуальная память.

Один из способов избежать этой проблемы - запустить PostgreSQL на компьютере, 
где другие процессы не будут запускать машину из памяти. Если памяти мало, 
увеличение пространства подкачки операционной системы может помочь избежать 
проблемы, поскольку средство устранения нехватки памяти (OOM) вызывается 
только тогда, когда физическая память и пространство подкачки исчерпаны.

Если сам PostgreSQL является причиной нехватки памяти в системе, вы можете 
избежать этой проблемы, изменив свою конфигурацию. В некоторых случаях это
может помочь снизить связанные с памятью параметры конфигурации, в частности 
shared_buffers, work_mem и hash_mem_multiplier. В других случаях проблема 
может быть вызвана слишком большим количеством подключений к самому серверу базы данных. 
Во многих случаях может быть лучше уменьшить max_connections и вместо этого 
использовать внешнее программное обеспечение для объединения подключений.

---

### Как cдавать задание

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

---
