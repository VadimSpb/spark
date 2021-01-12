# GB: BigData. Фреймворк Apache Spark
> **Geek University Data Engineering**

`Apache Spark`  <br>
`Python:PySpark`

## Урок 1. Архитектура Spark. Принципы исполнения запросов. Сохранение и чтение данных
_Apache Spark позволяет разделять хранение данных от их обработки, в отличие от СУБД._

**Задание**<br>
Построить распределения клиентов по возрастам `spark.table("homework.bank")`:
1. Распределение по возрасту с динамическим численным параметром `max_age`
2. Распределение по возрасту с динамическим параметром `marital`

[Решение](https://github.com/bostspb/spark/blob/master/lesson01.md) <br>
[Файл JSON для импорта в Zeppelin](https://github.com/bostspb/spark/blob/master/lesson01.zeppelin.json)

## Урок 2. Операции с данными: агрегаты, джойны. Оптимизация SQL-запросов
### Задание 1
Для упражнений сгрененирован большой набор синтетических данных в таблице `hw2.events_full`. 
Из этого набора данных созданы маленькие (относительно исходного набора) 
таблицы разного размера `kotelnikov.sample_[small, big, very_big]`. 

Ответить на вопросы:
 * какова структура таблиц
 * сколько в них записей 
 * сколько места занимают данные
 
### Задание 2
Получить планы запросов для джойна большой таблицы `hw2.events_full` 
с каждой из таблиц `hw2.sample`, `hw2.sample_big`, `hw2.sample_very_big` по полю `event_id`. <br>
В каких случаях используется **BroadcastHashJoin**? <br>
**BroadcastHashJoin** автоматически выполняется для джойна с таблицами, 
размером меньше параметра `spark.sql.autoBroadcastJoinThreshold`. 
Узнать его значение можно командой `spark.conf.get("spark.sql.autoBroadcastJoinThreshold")`.

### Задание 3
Выполнить джойны с таблицами `hw2.sample`, `hw2.sample_big` в отдельных параграфах, 
чтобы узнать время выполнения запросов (например, вызвать `.count()` для результатов запросов). 
Время выполнения параграфа считается автоматически и указывается в нижней части по завершении

Зайти в **spark ui** (ссылку сгенерировать в следующем папраграфе). 
Сколько tasks создано на каждую операцию? 
Почему именно столько? 
Каков DAG вычислений?

***Насильный broadcast***<br>
Оптимизировать джойн с таблицами hw2.sample_big, hw2.sample_very_big с помощью broadcast(df). Выполнить запрос, посмотреть в UI, как поменялся план запроса, DAG, количество тасков. Второй запрос не выполнится 

***Отключение auto broadcast***<br>
Отключить автоматический броадкаст командой `spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "-1")`. 
Сделать джойн с семплом `hw2.sample`, сравнить время выполнения запроса.

***Вернуть настройку к исходной***

### Задание 4
В процессе обработки данных может возникнуть перекос объёма партиций по количеству данных (data skew). 
В таком случае время выполнения запроса может существенно увеличиться, так как данные распределятся по исполнителям неравномерно. В следующем параграфе происходит инициализация датафрейма, этот параграф нужно выполнить, изменять код нельзя. В задании нужно работать с инициализированным датафреймом.

Датафрейм разделен на 30 партиций по ключу `city`, который имеет сильно  неравномерное распределение.


#### 4.1. Наблюдение проблемы
Посчитать количество event_count различных событий event_id , содержащихся в skew_df с группировкой по городам. Результат упорядочить по event_count.

В spark ui в разделе jobs выбрать последнюю, в ней зайти в stage, состоящую из 30 тасков (из такого количества партиций состоит skew_df). На странице стейджа нажать кнопку Event Timeline и увидеть время выполнения тасков по экзекьюторам. Одному из них выпала партиция с существенно большим количеством данных. Остальные экзекьюторы в это время бездействуют – это и является проблемой, которую предлагается решить далее.

#### 4.2. repartition
Один из способов решения проблемы агрегации по неравномерно распределенному ключу является 
предварительное перемешивание данных. Его можно сделать с помощью метода `repartition(p_num)`, 
где `p_num` -- количество партиций, на которые будет перемешан исходный датафрейм

#### 4.3. Key Salting
Другой способ исправить неравномерность по ключу -- создание синтетического ключа с равномерным распределением. 
В нашем случае неравномерность исходит от единственного значения city='BIG_CITY', 
которое часто повторяется в данных и при группировке попадает к одному экзекьютору. 
В таком случае лучше провести группировку в два этапа по синтетическому ключу CITY_SALT, 
который принимает значение BIG_CITY_rand (rand -- случайное целое число) для популярного значения 
BIG_CITY и CITY для остальных значений. На втором этапе восстанавливаем значения CITY и проводим повторную агрегацию, 
которая не занимает времени, потому что проводится по существенно меньшего размера данным. 

Такая же техника применима и к джойнам по неравномерному ключу, 
см, например https://itnext.io/handling-data-skew-in-apache-spark-9f56343e58e8

Что нужно реализовать:
* добавить синтетический ключ
* группировка по синтетическому ключу
* восстановление исходного значения
* группировка по исходной колонке

[Решение](https://github.com/bostspb/spark/blob/master/lesson02.md) <br>
[Файл JSON для импорта в Zeppelin](https://github.com/bostspb/spark/blob/master/lesson02.zeppelin.json)

## Урок 3. Типы данных в Spark. Коллекции как объекты DataFrame. User-Defined Functions
* По данным `habr_data` получить таблицу с названиями топ-3 статей (по `rating`) для каждого автора
* По данным `habr_data` получить топ (по встречаемости) английских слов из заголовков.<br> 

_Возможное решение:_ <br>
1) выделение слов с помощью регулярных выражений, 
2) разделение на массивы слов 
3) explode массивовов 
4) группировка с подсчетом встречаемости

[Решение](https://github.com/bostspb/spark/blob/master/lesson03.md) <br>
[Файл JSON для импорта в Zeppelin](https://github.com/bostspb/spark/blob/master/lesson03.zeppelin.json)

## Урок 4. Машинное обучение на pySpark на примере линейной регрессии
* построить распределение статей в датасете по `rating` с `bin_size = 10`
* написать функцию `ratingToClass(rating: Int): String`, которая определяет категорию статьи (A, B, C, D) на основе рейтинга. Границы для классов подобрать самостоятельно.
* добавить к датасету категориальную фичу `rating_class`. При добавлении колонки использовать `udf` из функции в предыдущем пункте
* построить модель логистической регрессии `one vs all` для классификации статей по рассчитанным классам.
* получить `F1 score` для получившейся модели

[Решение](https://github.com/bostspb/spark/blob/master/lesson04.md) <br>
[Файл JSON для импорта в Zeppelin](https://github.com/bostspb/spark/blob/master/lesson04.zeppelin.json)
