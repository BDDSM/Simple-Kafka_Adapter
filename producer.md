# Публикация сообщений

Общий принцип:

1. Устанавливаем необходимые параметры (УстановитьПараметр) (необязательно) https://github.com/confluentinc/librdkafka/blob/master/CONFIGURATION.md
2. Инициализируем подключение к брокеру (ИнициализироватьПродюсера)
3. Производим публикацию сообщений (ОтправитьСообщение, ОтправитьСообщениеСОжиданиемРезультата)
4. Останавливаем продюссера (ОстановитьПродюсера)

## Асинхронная и синхронная отправка сообщений

Библиотека, на которой основана компонента, по своей природе имеет асинхронную архитектуру взаимодействия с брокерами.

### Асинхронная отправка

Для **асинхронной** отправки сообщения необходимо использовать метод **ОтправитьСообщение**. Метод возвращает булево значение "Истина" практически во всех случаях, кроме следующих кейсов:
+ Размер сообщения превышает установленный максимальный размер: message.max.bytes
+ Запрошенный раздел (партиция) неизвестен в кластере Kafka
+ Указанная тема не найдена в кластере Kafka (только если отключено автосоздание топиков на брокере)

В случае асинхронной отправки - результаты доставки можно получить с помощью следующих архитектурных подходов:
+ При создании экземпляра продюссера, создаем так же экземпляр консьюмера, подписываемся на тему, куда производится отправка. После отправки каждого сообщения - получаем с помощью экземпляра консьюмера сообщение, вычисляем хэш отправленного и полученного сообщения (или же используем ключи сообщений). Сравниваем хэши или ключи - в случае совпадения - считаем сообщение доставленным.
+ Настраиваем считывание из файла лога, который был установлен методом **УстановитьФайлЛогирования**. Если при отправке был указан key сообщения, в лог будет выведен key и результат доставки. Если key не был указан - в лог будет выведен хэш (md5) сообщения.
    
Порядок полей в логе: Дата время: Хэш или ключ сообщения, Статус доставки:Причина, Размер сообщения в байтах, Топик, Смещение (-1001 если все плохо), Партиция (-1 если все плохо), Брокер (-1 если все плохо).

Примеры строк в логе доставки
    
    2023-09-01 13:46:03.145: Hash:83781bf6c9b31ee4445a0c257d7feba0, NotPersisted:Local: Message timed out, Size:21, Topic:testTopic, Offset:-1001, Partition:-1, BrokerID:-1
    2023-09-01 13:57:58.160: Hash:83781bf6c9b31ee4445a0c257d7feba0, Persisted:Success, Size:21, Topic:testTopic, Offset:20520, Partition:0, BrokerID:0

### Синхронная отправка

Для **синхронной** отправки необходимо использовать метод **ОтправитьСообщениеСОжиданиемРезультата**, состав параметров аналогичен методу асинхронной отправки. Метод возвращает значение типа булево - "Истина", если при доставке все брокеры вернули подтверждение о приемке сообщения, "Ложь", если хотя бы один брокер не подтвердил получение или не было получено подтверждение от брокеров в течении 20 секунд. Крайне рекомендуется не изменять значение параметра **acks**, который по умолчанию имеет значение "all". 


### Пример асинхронной отправки

```1c

	Если ТипЗнч(Данные) <> Тип("Массив") Тогда
		Сообщения = ОбщегоНазначенияКлиентСервер.ЗначениеВМассиве(Данные);
	Иначе
		Сообщения = Данные;
	КонецЕсли;
	
	Если ТипЗнч(Параметры) = Тип("Соответствие") ИЛИ (ТипЗнч(Параметры) = Тип("Структура")) Тогда
		Для каждого Параметр Из Параметры Цикл
			Компонента.УстановитьПараметр(Строка(Параметр.Ключ), Строка(Параметр.Значение));		
		КонецЦикла;
	КонецЕсли; 	
	
    // инициализируем подключение к брокеру
    // достаточно указать одного брокера из кластера, либо есть возможность перечислить брокеров через ,
	РезультатИнициализации = Компонента.ИнициализироватьПродюсера(Брокеры);
	
	Если РезультатИнициализации Тогда
		
		Для каждого СообщениеВКафку Из Сообщения Цикл			        			
			// Parametr1 - Тело сообщения (строка)
            // Parametr2 - Топик (строка)
			// Parametr3 - Номер партиции, по умолчанию = -1 (число)
			// Parametr4 - Произвольный ключ, идентифицирующий сообщение, например GUID (строка)
			// Parametr5 - Заголовки (строка), "ключ1,значение1;ключ2,значение2"
			Компонента.ОтправитьСообщение(СообщениеВКафку, Топик,, Ключ, Заголовки);  
		КонецЦикла;
		
		Компонента.ОстановитьПродюсера();
		
	КонецЕсли;
		
	Компонента = Неопределено;

```