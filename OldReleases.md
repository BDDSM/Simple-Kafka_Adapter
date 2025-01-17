## Использование компоненты, предыдущих редакций < 1.1.0 

Можно подключать как изолировано (в child процессе), так и в рамках основного процесса 

`пример подключения на клиенте`
```1c
	Подключено = Ждать ПодключитьВнешнююКомпонентуАсинх("ОбщийМакет.Компонента", "Integration",, ТипПодключенияВнешнейКомпоненты.Изолированно);
	
	Если Не Подключено Тогда
	    Ждать УстановитьВнешнююКомпонентуАсинх(
	        "ОбщийМакет.Компонента");
	    Подключено = Ждать ПодключитьВнешнююКомпонентуАсинх(
	        "ОбщийМакет.Компонента", "Integration");		
		ПоказатьПредупреждение(Неопределено, СтрШаблон("Результат подключения: %1", ?(Подключено, "подключено", "ошибка подключения!")));
	КонецЕсли;
	
	Попытка
		Компонента = Новый("AddIn.Integration.simpleKafka1C");     
	Исключение
		ПоказатьПредупреждение(Неопределено, "Компонента <Simple Kafka 1C> не подключена!");
		Возврат;
	КонецПопытки;
```

`пример подключения на сервере`
```1c
	ПодключитьВнешнююКомпоненту("ОбщийМакет.Компонента", "Integration", ТипВнешнейКомпоненты.Native, ТипПодключенияВнешнейКомпоненты.Изолированно);		

	Попытка
		Компонента = Новый("AddIn.Integration.simpleKafka1C");     
	Исключение
		ВызватьИсключение "Компонента <Simple Kafka 1C> не подключена!";
	КонецПопытки;       
```

### Публикация сообщений 

`пример публикации на клиенте`

```1c
	// инициализируем подключение к брокеру и топику  
	// Parametr1 - Брокеры, к которым производится подключение (строка) 
	// Parametr2 - Имя топика, в который производится публикация (строка) 
	Результат = Ждать Компонента.ИнициализироватьПродюсераАсинх(Брокеры, Топик);

	Если Результат.Значение Тогда
		Для т = 1 по КоличествоСообщений Цикл    
			// Parametr1 - Тело сообщения (строка)
			// Parametr2 - Номер партиции, по умолчанию = -1 (число)
			// Parametr3 - Произвольный ключ, идентифицирующий сообщение, например GUID
			// Parametr4 - Заголовки (строка), "ключ1,значение1;ключ2,значение2"
			РезультатПубликации = Ждать Компонента.ОтправитьСообщениеАсинх(СообщениеВКафку,, Ключ, Заголовки);  
			
			Если НЕ РезультатПубликации.Значение Тогда 
	        	Прервать; // здесь обработать ошибку бы надо...
			КонецЕсли;
			
		КонецЦикла;
		
		// освобождаем память
		// если не вызвать, то очень вероятна потеря части сообщений и расход памяти
		Ждать Компонента.ОстановитьПродюсераАсинх();  

	КонецЕсли;           
	
	Компонента = Неопределено;   
```

### Чтение сообщений

`пример чтения на клиенте`
```1c
    // указываем параметр - идентификатор группы, в рамках которой мы подписываемся на топик
	Ждать КомпонентаСлушателя.УстановитьПараметрАсинх("group.id", "odin_c");
	Результат = Ждать КомпонентаСлушателя.ИнициализироватьКонсьюмераАсинх(Брокеры, Топик);    
	ПодключитьОбработчикОжидания("СлушатьСообщения", 1);   
```

```1c
&НаКлиенте
Асинх Процедура СлушатьСообщения()
	Партиция  = Неопределено;     
	Смещение  = Неопределено;
	Сообщение = Неопределено;
	Ключ      = Неопределено;
	Топик     = Неопределено;
	ОтметкаВремени = Неопределено;
	
	Сообщение = Ждать КомпонентаСлушателя.СлушатьАсинх();     
	
	Если ЗначениеЗаполнено(Сообщение.Значение) Тогда       

    		ОбъектЧтение = Новый ЧтениеJSON;
		ОбъектЧтение.УстановитьСтроку(Сообщение.Значение);
		СтруктураJSON = ПрочитатьJSON(ОбъектЧтение);
		ОбъектЧтение.Закрыть();
			
		СтруктураJSON.Свойство("topic",  Топик);
		СтруктураJSON.Свойство("offset",  Смещение);
		СтруктураJSON.Свойство("partition",  Партиция);
		СтруктураJSON.Свойство("message", Сообщение);	
		СтруктураJSON.Свойство("key",     Ключ);
		СтруктураJSON.Свойство("timestamp",     ОтметкаВремени);
		
	КонецЕсли;

КонецПроцедуры
```

`пример чтения на сервере`

Один из подходов - используем регламентное задание, которое стартует, к примеру, каждую минуту и запускает фоновое задание, в котором будет простушиваться топик. Предварительно проверяем - запущено ли уже такое фоновое задание, если запущено - то ничего не делаем.

```
// Процедура - Слушать сообщения
//
// Параметры:
//  Брокер						 - Строка	 -  Имена брокеров, разделенных ","
//  Топик						 - Строка	 -  Топик, на который подписывается слушатель
//  Параметры					 - Соответствие	 - Ключ/Значение параметров, см https://github.com/confluentinc/librdkafka/blob/master/CONFIGURATION.md
//  ВремяРаботы					 - Число	 -  Максимальная продолжительность работы компоненты. Определяет время жизни фонового задания.
//												Необязательный, если не задан, то значение = 3600 sec.
//  Таймаут						 - Число	 -  Время ожидания получения сообщения из топика, если сообщение есть. Необязательный, если не задан, то значение = 100 ms.
//
Процедура СлушатьСообщения(Брокер, Топик, Параметры, ВремяРаботы = 3600, Таймаут = 100) Экспорт
						   
	РазрешеноСлушать = ПолучитьЗначениеНастройки("РазрешенаПрослушкаТопика"); // получение из константы или настроечного регистра
	
	Если НЕ РазрешеноСлушать Тогда
		Возврат;
	КонецЕсли;
						   
	ВремяНачала = ТекущаяДатаСеанса();	
	ЗаписьЖурналаРегистрации("Интеграция Kafka. Consumer", УровеньЖурналаРегистрации.Информация,,, СтрШаблон("Старт подписки на топик %1", Топик));

    Попытка
		Компонента = Новый(СтрШаблон("AddIn.%1.simpleKafka1C", "Integration"));   
	Исключение
		Подключено = ПодключитьВнешнююКомпоненту("ОбщийМакет.КомпонентаSimpleKafka", "Integration", 
													ТипВнешнейКомпоненты.Native, ТипПодключенияВнешнейКомпоненты.Изолированно);
		Если Подключено Тогда
			Компонента = Новый(СтрШаблон("AddIn.%1.simpleKafka1C", "Integration"));  	
		КонецЕсли;
	КонецПопытки;
	
	Если Компонента = Неопределено Тогда
		Возврат;	
	КонецЕсли;
	
	Для каждого Параметр_ Из Параметры Цикл
		Компонента.УстановитьПараметр(Строка(Параметр_.Ключ), Строка(Параметр_.Значение));	
	КонецЦикла;   	
	
	КаталогЛогов = ПолучитьЗначениеНастройки("КаталогЛоговKafka");
	Если ЗначениеЗаполнено(КаталогЛогов) Тогда
		Компонента.УстановитьФайлЛогирования(СтрШаблон("%1%2%3.log", КаталогЛогов , ПолучитьРазделительПути(), Новый УникальныйИдентификатор));
	КонецЕсли;
   
	Результат = Компонента.ИнициализироватьКонсьюмера(Брокер, Топик);      
	
	// установка таймаута для ожидания сообщений
	Компонента.УстановитьТаймаутОжидания(Таймаут);	

	Если Не Результат Тогда  
		ТекстОшибки = СтрШаблон("Не удалось инициализировать консьюмера для топика %1", Топик);  
		ЗаписьЖурналаРегистрации("Интеграция Kafka. Consumer", УровеньЖурналаРегистрации.Ошибка,,, ТекстОшибки);
		Возврат;
	КонецЕсли;                 
		
	Пока РазрешеноСлушать Цикл
		
		РазрешеноСлушать = ПолучитьЗначениеНастройки("РазрешенаПрослушкаТопика");
		
		Если Не РазрешеноСлушать Тогда
			Прервать;	
		КонецЕсли;			      	
		
		Попытка
			Сообщение = Компонента.Слушать(); 
		Исключение                	
			Компонента = Неопределено;                                                         
			ВызватьИсключение ОписаниеОшибки();
		КонецПопытки;	
		
		Если Не ЗначениеЗаполнено(Сообщение) Тогда  
			Если ТекущаяДатаСеанса() > ВремяНачала + ВремяРаботы Тогда 
				РазрешеноСлушать = Ложь;
			КонецЕсли;
			Продолжить;	    	
		КонецЕсли;
		
		// проверяем что сообщение с указанным offset и timestamp нами не было ранее получено
		
		// здесь дальнейшие действия с сообщением:
		// - можно сложить в отложенный регистр, из которого будет идти фоновая обработка полученных сообщений
		// - можно обработать сразу, но лучше так не делать, т.к. в случае длительной обработки сообщения - клиент может быть отключен   
		
	КонецЦикла;  
	
	Компонента.ОстановитьКонсьюмера();	
	Компонента = Неопределено;
	
	ЗаписьЖурналаРегистрации("Интеграция Kafka. Consumer", УровеньЖурналаРегистрации.Информация,,, СтрШаблон("Остановка подписки на топик %1", Топик));
	
КонецПроцедуры
```

### Получение offset и ручная фиксация offset

Актуальный вариант:
```
Сообщение = Компонента.Слушать();

ОбъектЧтение = Новый ЧтениеJSON;
ОбъектЧтение.УстановитьСтроку(Сообщение);
СтруктураJSON = ПрочитатьJSON(ОбъектЧтение);
ОбъектЧтение.Закрыть();
			
Смещение = СтруктураJSON.offset;
Партиция = СтруктураJSONpartition;

Компонента.ЗафиксироватьСмещение(Смещение, Партиция);
```

Legacy вариант:
```
ТекущееСмещение = Компонента.ТекущееСмещение();
Компонента.ЗафиксироватьСмещение(ТекущееСмещение);
```

### Полный перечень методов компоненты

|Eng   	|Рус   	|Описание|
|---	|---	|---	|
|SetLogsReportFileName   |УстановитьФайлЛогирования   	|Указывается полный путь до файла лога   	|
|SetParameter   	|УстановитьПараметр   	|Все параметры указаны в https://github.com/confluentinc/librdkafka/blob/master/CONFIGURATION.md	|
|InitializeProducer   	|ИнициализироватьПродюсера |Указываются брокеры первым параметром, вторым - топик	|
|Produce   	| ОтправитьСообщение   	|Непосредственная отправка, Сообщение, Номер партиции (по умолчанию 0), ключ, заголовки	|
|StopProducer |ОстановитьПродюсера | |
|InitializeConsumer|ИнициализироватьКонсьюмера|Указываются брокеры первым параметром, вторым - топик (на данный момент - единственный)|
|setWaitingTimeout|УстановитьТаймаутОжидания|Таймаут ожидания сообщений в ms|
|Consume|Слушать|Ожидает сообщение, если в течении указанного таймаута ожидания - сообщений нет - вернется пустая строка|
|CurrentOffset|ТекущееСмещение|Legacy: Получить текущую отметку смещения прочитанного сообщения (рекомендуется использовать метод Слушать)|
|CommitOffset|ЗафиксироватьСмещение|Ручная фиксация отметки смещения прочитанного сообщения. Рекомендуется использовать при подходе - когда мы считали сообщение, что то долго делаем с ним и затем фиксируем смещение. Параметр enable.auto.commit при этом должен быть установлен в false|
|StopConsumer|ОстановитьКонсьюмера|Остановка консьюмера. Обязательно вызывается при завершении итераций чтений сообщений|
