# Case-study оптимизации

## Актуальная проблема
В нашем проекте возникла серьёзная проблема.

Необходимо было обработать файл с данными, чуть больше ста мегабайт.

У нас уже была программа на `ruby`, которая умела делать нужную обработку.

Она успешно работала на файлах размером пару мегабайт, но для большого файла она работала слишком долго, и не было понятно, закончит ли она вообще работу за какое-то разумное время.

Я решил исправить эту проблему, оптимизировав эту программу.

## Формирование метрики
Для того, чтобы понимать, дают ли мои изменения положительный эффект на быстродействие программы я придумал использовать такую метрику: оперативная память

## Гарантия корректности работы оптимизированной программы
Программа поставлялась с тестом. Выполнение этого теста в фидбек-лупе позволяет не допустить изменения логики программы при оптимизации.

## Feedback-Loop
Для того, чтобы иметь возможность быстро проверять гипотезы я выстроил эффективный `feedback-loop`, который позволил мне получать обратную связь по эффективности сделанных изменений за 15 секунд

Вот как я построил `feedback_loop`: 
- замер потребляемой памяти профилировщиком
- выяснение точки роста
- поиск возможности оптимизации точки роста
- при возможности, оптимизация точки роста

## Вникаем в детали системы, чтобы найти главные точки роста
Для того, чтобы найти "точки роста" для оптимизации я воспользовался 
- gem 'memory_profiler'
- gem 'ruby-prof'
- gem 'stackprof'

Вот какие проблемы удалось найти и решить

### Ваша находка №1
- memory_profiler показал точку роста в загрузке всего файла
- переделать программу на построчное считывание
- изначально при чтении файла из 10000 строк потреблялось 441.91 MB, после изменения программы на построчное считывание стало 16.47 MB
- изменилась главна точка роста

### Ваша находка №2
- stackprof показал точку роста в split(',')
- убрал лишние split(','), оставил только в одном месте
- кол-во аллокаций уменьшилось с 93104 до 67712
- кол-во потребляемой памяти уменьшилось с 16.47 MB до 13.09 MB
- пробовал заменить (',') на регулярное выражение, но память увеличиласть на 1 мб
- точка роста осталась там же

### Ваша находка №3
- memory_profiler показал что создается много одинаковых срок
- добавил frozen_string_literal: true
- было 179224 в классе String, стало 146149
- потребление памяти уменьшилось с 13.09 MB до 11.77 MB

### Ваша находка №4
- memory_profiler показал что создается много одинаковых срок
- добавил frozen_string_literal: true
- было 179224 в классе String, стало 146149
- потребление памяти уменьшилось с 13.09 MB до 11.77 MB

### Ваша находка №5
- memory_profiler показал что создается много объектов в строке присвоения user_key
- заменил на интерполяцию
- объектов было 6144, стало 1536
- потребление памяти уменьшилось незначительно с 11.77 MB до 11.52 MB

### Ваша находка №6
- memory-profiler показал что создается 4438 объектов при сортировке браузеров
- заменил на sort!
- объектов было 4438, стало 2902
- потребление памяти уменьшилось незначительно с 11.52 MB до 11.40 MB

### Ваша находка №7
- memory-profiler показал что uniq потребляет много памяти
- было решено заменить uniq на Set как в первом задании
- потребление памяти уменьшилось незначительно с 11.40 MB до 11.15 MB

### Ваша находка №8
- memory-profiler показал что все еще не укладываемся в метрику, много памяти уходит на запись в файл
- было решено перевести запись в файл на постепенную, по мере поступления результатов
- потребление памяти уменьшилось незначительно с 11.40 MB до 5.83 MB

## Результаты
В результате проделанной оптимизации наконец удалось обработать файл с данными.
Удалось улучшить метрику системы с 441.91 MB до 5.83 MB и уложиться в заданный бюджет. Так же удалось довести время выполнения до заданных 30 сек.
xquartz запустить не удалось, может на big sur не идет. Однако, был найден онлайн аналог http://boutglay.com/massifjs/ который почему то не запускается с русского ip, но мы и не таких видали, поэтому скриншоты прилагаю. В итоге, удалось снизить постребление памяти до ~38,6 MB.

## Защита от регрессии производительности
Для защиты от потери достигнутого прогресса при дальнейших изменениях программы тесты на rspec.
