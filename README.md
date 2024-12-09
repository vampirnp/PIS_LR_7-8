# PIS_LR_7-8

## 1. Описание предметной области
### Система разработана для управления электронной коммерцией. Основной функционал включает:

- Регистрацию и управление пользователями.
- Поиск и выбор товаров.
- Оформление заказов, включая выбор способов доставки и оплаты.
- Проведение платежей через интеграцию с внешними системами (например, SberPay).
- Управление заказами и уведомления пользователей.

## 2. Цель и назначение разработки
### Цель: Создать модульный сервис электронной коммерции с интеграцией платежей и удобным пользовательским интерфейсом.

### Назначение: 
- Масштабируемость системы для увеличения числа пользователей и товаров.
- Гибкость и простоту добавления новых функций.
- Надежность при сбоях отдельных компонентов.

## 3. Краткое описание функционала (ЛР 4-6)
### ЛР4: Разработка пользовательского интерфейса для оформления заказа:
- Ввод контактных данных.
- Выбор способов доставки и оплаты.

### ЛР5: Создание REST API для обработки заказов:
- Добавление товаров в корзину.
- Создание и удаление заказов.
- Интеграция с платежными системами.

### ЛР6: Проектирование микросервисной архитектуры:
- Разделение на независимые модули (Cart, Order, Payment).
- Оптимизация взаимодействия через REST API и асинхронные протоколы.

## 4. Схема архитектуры системы (ЛР6)

![image](https://github.com/user-attachments/assets/9af9515e-501c-42cd-a29e-50a8af04758c)

## 5. ТЗ на front-end
### Макеты страниц:

- Внесение данных для оформления заказа.
- Оплата.
## Примеры макетов:
#### Внесение данных
![image](https://github.com/user-attachments/assets/d38441fe-5dd1-4c76-8456-86fed7ba9ad8)

#### Оплата
![image](https://github.com/user-attachments/assets/20b59c1e-d5d9-42b6-8299-810857a66228)

### Основные элементы формы:

|Элемент|	Тип элемента|	JSON Поле	|Обязательность|
|---------|--------------|--------------|------------|
|Поле "Фамилия"|	Input	|clientInfo.lastName	|Обязательное|
|Поле "Имя"	|Input|	clientInfo.firstName|	Обязательное|
|Поле "Адрес электронной почты"|	Input|	contactInfo.email|	Обязательное|
|Поле "Контактный телефон"	|Input|	contactInfo.phone|	Обязательное|
|Переключатель "Способ оплаты"	|Toggle|	paymentMethod.method|	Обязательное|

### Валидация адреса доставки
**Маска ввода адреса**: `г. [Город], ул. [Улица], д. [Дом], кв. [Квартира]`  
Пример: `г. Москва, ул. Тверская, д. 1, кв. 2`.

**Валидационные правила**:
- Длина не менее 10 символов.
- Наличие ключевых слов: "г.", "ул.", "д.", "кв.".
- Запрещены специальные символы, кроме точек и запятых.

**Сообщение об ошибке**:  
*"Пожалуйста, введите адрес в формате: г. [Город], ул. [Улица], д. [Дом], кв. [Квартира]"*.

## 6. ТЗ на back-end
### Модель данных:
- User: Идентификация и контактные данные пользователя.
- Product: Информация о товарах.
- Order: Детали заказа, включая товары и способы доставки.
- Basket_Items: Содержимое корзины.

### Унификация статусов товаров и заказов
#### Товары:
- `ACTIVE` – доступен для покупки.
- `INACTIVE` – временно недоступен.
- `OUT_OF_STOCK` – закончился.

#### Заказы:
- `PENDING` – создан, ожидает оплаты.
- `PAID` – оплата успешно завершена.
- `CANCELED` – заказ отменён.
- `IN_PROGRESS` – выполняется (сборка/доставка).
- `COMPLETED` – выполнен и доставлен.

### Пример запроса для создания заказа:
- URL: `/api/orders`
- Метод: `POST`
- Авторизация: Базовая

#### Тело запроса:
```json
{
  "ProductInfo": {
    "product_name": "Проводные наушники HyperX Cloud II KHX-HSCP-RD красный",
    "article": "12h12322",
    "rating": "4.6",
    "price": 13272
  },

  "clientInfo": {
    "firstName": "Орлов",
    "lastName": "Александр",
    "middleName": "Кириллович"
  },

  "contactInfo": {
    "email": "example@mail.ru",
    "phone": "+7 777 777 02 03"
  },

  "deliveryOfGoods": {
    "inStore": {
      "included": true
    },
    "deliveryToAddress": {
      "included": false,
      "address": "г.Клин Ул.полевая д.45",
      "price": 2000
    }
  },

  "paymentMethod": {
    "SBP": {
      "included": false
    },
    "SBERPay": {
      "included": false
    },
    "Card": {
      "included": false
    },
    "CashOnDelivery": {
      "included": true
    }
  }
}

```


- ### Логика обработки успешной и неуспешной оплаты
#### При успешной оплате:
- Статус заказа обновляется на `PAID`.
- В Kafka отправляется событие об успешной оплате.
- Логируется информация о платеже.

#### При неуспешной оплате:
- Сообщение клиенту: *"Оплата не прошла. Попробуйте снова или выберите другой способ оплаты."*
- Статус заказа остаётся `PENDING`.

### Kafka: сообщения и события
1. **Уведомление об успешной оплате**:  
   - Тема: `payment.success`.  
   - Сообщение:  
     ```json
     {
       "orderId": "uuid",
       "userId": "uuid",
       "status": "PAID",
       "paymentDate": "2024-12-09T12:00:00Z"
     }
     ```

2. **Изменение статуса заказа**:  
   - Тема: `order.status.update`.  
   - Сообщение:  
     ```json
     {
       "orderId": "uuid",
       "status": "IN_PROGRESS",
       "updatedAt": "2024-12-09T12:00:00Z"
     }
     ```

3. **Уведомление о доставке**:  
   - Тема: `delivery.update`.  
   - Сообщение:  
     ```json
     {
       "orderId": "uuid",
       "deliveryStatus": "DELIVERED",
       "deliveryDate": "2024-12-10T15:30:00Z"
     }
     ```

### Интеграция с платёжным шлюзом
Пример ответа шлюза:
```json
{
  "transactionId": "12345abcde",
  "status": "SUCCESS",
  "paymentDate": "2024-12-09T12:00:00Z",
  "amount": 13272
}
```

#### Логика обработки:
-При статусе SUCCESS: Обновить статус заказа, отправить уведомление через Kafka.
- 200 OK: {"message": "Заказ успешно создан"}
-
-При статусе FAILED: Вернуть сообщение с описанием ошибки.
- 400 Bad Request: {"message": "Неверные данные"}

# PIS_LR_3
### вытаскивал из сообщений на почте поэтому отредачить не могу
## 1. Концепт
![image](https://github.com/user-attachments/assets/8a63f56f-b481-479c-8552-9e962dc34086)
## 2. Структура Бд
![image](https://github.com/user-attachments/assets/9fa4b4cc-166f-4ec7-a074-05db4f4d2228)

## 3.Таблицы Бд
## User
![image](https://github.com/user-attachments/assets/54e03ad7-ee62-4f50-bae7-a977ed24e4c6)
## Basket
![image](https://github.com/user-attachments/assets/9b9bb3b8-10ad-4734-b66b-229fcfb7f803)
## Product
![image](https://github.com/user-attachments/assets/6054a4bc-d15e-4490-a9d4-454fbcb6664d)
## Order
![image](https://github.com/user-attachments/assets/2ef7a911-e250-4d2e-a3e0-38f549c3276a)
## KUper
![image](https://github.com/user-attachments/assets/4d81785b-ec2c-4423-b217-263e78a17227)
## Basekt_Items
![image](https://github.com/user-attachments/assets/1233e8c8-2ede-46d5-b323-8059c7ba7503)


# PIS_LR_4
### ТЗ на страницу добавление внесение данных для оплаты и оплата товара front-end

## 1. Макеты страницы/функции/фичи
![Внесение данных](https://github.com/user-attachments/assets/d38441fe-5dd1-4c76-8456-86fed7ba9ad8)
![Оплата](https://github.com/user-attachments/assets/20b59c1e-d5d9-42b6-8299-810857a66228)



## 2. SEO

URL страницы: /checkout
Хлебные крошки: Главная > Корзина > Внесение данных для оплаты > Оплата заказа

## 3. JSON при инициализации


```json
{
  "ProductInfo": {
    "product_name": "Проводные наушники HyperX Cloud II KHX-HSCP-RD красный",
    "article": "12h12322",
    "rating": "4.6",
    "price": 13272
  },

  "clientInfo": {
    "firstName": "Орлов",
    "lastName": "Александр",
    "middleName": "Кириллович"
  },

  "contactInfo": {
    "email": "example@mail.ru",
    "phone": "+7 777 777 02 03"
  },

  "deliveryOfGoods": {
    "inStore": {
      "included": true
    },
    "deliveryToAddress": {
      "included": false,
      "address": "г.Клин Ул.полевая д.45",
      "price": 2000
    }
  },

  "paymentMethod": {
    "SBP": {
      "included": false
    },
    "SBERPay": {
      "included": false
    },
    "Card": {
      "included": false
    },
    "CashOnDelivery": {
      "included": true
    }
  }
}

```

## 4. Маппинг данных

### 4.1 Форма внесения данных

Скриншот элемента: 
![Внесение данных](https://github.com/user-attachments/assets/d38441fe-5dd1-4c76-8456-86fed7ba9ad8)

Условие отображения: Всегда

Описание элемента:

| Элемент | Тип элемента | Поле из JSON | Примечание |
|---------|--------------|--------------|------------|
| Заголовок "Оплата заказа" | Текст | - | Статический текст |
| Заголовок "1 данные покупателя" | Текст | - | Статический текст |
| Поле "Фамилия" | Input, обязательное | clientInfo.lastName | - |
| Поле "Имя" | Input, обязательное | clientInfo.firstName | - |
| Поле "Отчество" | Input | clientInfo.middleName | Необязательное поле |
| Заголовок "2 Контактные данные" | Текст | - | Статический текст |
| Поле "Адрес электронной почты" | Input, обязательное | contactInfo.email | Валидация формата email |
| Поле "Контактный телефон" | Input, обязательное | contactInfo.phone | Маска ввода телефона |
| Заголовок "3 Способ получения" | Текст | - | Статический текст |
| Переключатель Способ получения | Toggle | delivery_of_goodsInfo.method | Выбор между в магазине и доставка по адрессу |
| Радио-кнопка "В магазине" | Radio button | - | Опция способа оплаты |
| Радио-кнопка "доставка по адресу" | Radio button | - | Опция способа оплаты|
| Стоимость доставки | Текст | delivery_of_goodsInfo.price | "2000 ₽" |
| Поле "Адрес" | Input, обязательное | contactInfo.addres | Валидация формата addres |
|Заголовок "4 Способ оплаты"	|Текст	|-|	Статический текстм|
|Переключатель Способ оплаты	|Toggle|	paymentMethod.method|	|Выбор оплаты между СБП СберПэй картой и наличкой в магазине|
|Радио-кнопка "Система быстрых платежей"	|Radio button|	paymentMethod.SBP.included	|Опция способа оплаты, по умолчанию выбрана если выбран способ доставка по адресу|
|Радио-кнопка "Сберпэй"	|Radio button	|paymentMethod.SBERPay.included	|Опция способа оплаты|
|Радио-кнопка "оплата Картой"	|Radio button	|paymentMethod.Card.included	|Опция способа оплаты|
|Радио-кнопка "оплата наличныйми"	|Radio button	|paymentMethod.CashOnDelivery.included	|Опция способа оплаты, по умолчанию выбрана если выбран способ получения в магазине|

P.S если пользователь выбрал сберпэй или сбп то его направляет на их сайт.

### 4.2 Форма оплаты 


![Оплата](https://github.com/user-attachments/assets/20b59c1e-d5d9-42b6-8299-810857a66228)

Условие отображения: После заполнения формы оформление заказа выбрав способ оплаты карта

Описание элементов:

| Элемент | Тип элемента | Поле из JSON | Примечание |
|---------|--------------|--------------|------------|
| Заголовок "Осталось только оплатить" | Текст | - | Статический текст |
| Поле "Номер карты" | Input | - | Появляется при выборе оплаты картой |
| Поле "Имя на карте" | Input | - | Появляется при выборе оплаты картой |
| Поле "Срок до" | Input | - | Появляется при выборе оплаты картой |
| Поле "CVC" | Input | - | Появляется при выборе оплаты картой |
| Сумма к оплате | Текст | price | Отображение итоговой стоимости |
| Кнопка "Купить" | Button | - | При нажатии происходит оплата |

## 5. Действия на странице/в функции

| Действие | Формат вызова функции | Эндпоинт | Примечание |
|----------|------------------------|----------|------------|
| Заполнение формы внесение данных для оплаты | Ввод данных в форму | /api/orders | Валидация полей при вводе |
| выбор способа получения товара | Клик по кнопке "в магазине" или "доставка по адресу" | POST /api/delivery-method | Обновление стоимости |
| Переход к оплате | Клик по кнопке "Далее" | POST /validate-booking | Отправка данных формы для валидации |


# PIS_LR_5

### Техническое задание на разработку. Бэкенд

## 1. Входные параметры

| Параметр       | Тип параметра | Обязательность | Описание                                |
|----------------|---------------|----------------|-----------------------------------------|
| userId         | uuid          | +              | Идентификатор сущности                  |
| productId      | uuid          | +              | Идентификатор продукта                  |

## 2. Авторизация

a. По бизнесу
Доступ к функции добавления товара в корзину
Просмотр и управление корзиной доступно только владельцу 
Доступ к информации о остатке товара предоставляется всем пользователям

b. Техническая реализация
-- Проверка прав доступа к корзине
SELECT id FROM tbl_User
WHERE user_id = {current_user} 
AND id = {orderId}

-- Проверка статуса
SELECT status FROM tbl_product 
WHERE id = {product_Id} 
AND status = 'ACTIVE'

## 3. Передача данных при инициализации

```json
{
  "productInfo": {
    "product_name": "Проводные наушники HyperX Cloud II KHX-HSCP-RD красный",
    "article": "12h12322",
    "rating": 4.6,
    "price": 13272
  },

  "clientInfo": {
    "firstName": "Орлов",
    "lastName": "Александр",
    "middleName": "Кириллович"
  },

  "contactInfo": {
    "email": "example@mail.ru",
    "phone": "+7 777 777 02 03"
  },

  "deliveryOfGoods": {
    "inStore": {
      "included": true
    },
    "deliveryToAddress": {
      "included": false,
      "address": "г.Клин Ул.полевая д.45",
      "price": 2000
    }
  },

  "paymentMethod": {
    "SBP": {
      "included": false
    },
    "SBERPay": {
      "included": false
    },
    "card": {
      "included": false
    },
    "cashOnDelivery": {
      "included": true
    }
  }
}
```

## 4. Таблица атрибутов

| Атрибут, уровень 1 | Уровень 2 | Тип     | Название атрибута       | Формирование на бэкенде                                                                | Обязательность |
|--------------------|------------|---------|-------------------------|---------------------------------------------------------------------------------------|----------------|
| productId          |            | object  | индетификатор продукта  | select tbl_pruduct_id from tbl_product where id =<product_Id>                         | +              |
| product_name       |            | string  | Имя продукта            | select tbl_product_name from tbl_product where product_name=<str>                     | +              |
| article            |            | integer | Артикул                 | SELECT article FROM tbl_product WHERE product_id = <product_id>                       | +              |
| price              |            | integer | Цена                    | SELECT price FROM tbl_product WHERE product_id = <product_id>	                        | +              |
| clientInfo         |            | object  | Информация клиента      |                                                                                       | +              |
| firstName          |            | string  | Имя                     | SELECT first_name FROM tbl_users WHERE user_id = <user_id>                            | +              |
| lastName           |            | string  | Фамилия                 | SELECT last_name FROM tbl_users WHERE user_id = <user_id>                             | +              |
| middleName         |            | string  | Отчество                | SELECT middle_name FROM tbl_users WHERE user_id = <user_id>                           | -              |
| contactInfo        |            | object  | Контактная информация   | -                                                                                     | +              |
| email              |            | string  | Электронная почта       | SELECT email FROM tbl_users WHERE user_id = <user_id>                                 | +              |
| phone              |            | string  | Телефон                 | SELECT phone FROM tbl_users WHERE user_id = <user_id>                                 | +              |
| deliveryOfGoods    |            | object  | Доставка товаров        |                                                                                       | +              |
| inStore            |            | boolean | В магазине              | SELECT in_store FROM tbl_delivery WHERE order_id = <order_id>                         | +              |
| deliveryToAddress  |            | object  | Доставка по адресу      | -                                                                                     | +              |
| address            |            | string  | Адрес                   | SELECT address FROM tbl_delivery WHERE order_id = <order_id>                          | +              |
| price              |            | integer | Цена                    | SELECT delivery_price FROM tbl_delivery WHERE order_id = <order_id>                   | +              |
| paymentMethod      |            | object  | Способ оплаты           | -                                                                                     | +              |
| SBP                |            | boolean | Система быстрых платежей| SELECT SBP FROM tbl_payment_methods WHERE order_id = <order_id>                       | +              |
| SBERPay            |            | boolean | СберПэй                 | SELECT SBERPay FROM tbl_payment_methods WHERE order_id = <order_id>                   | +              |
| card               |            | boolean | Оплата картой           | SELECT card FROM tbl_payment_methods WHERE order_id = <order_id>	                    | +              |
| cashOnDelivery     |            | boolean | Оплата наличными        | SELECT cash_on_delivery FROM tbl_payment_methods WHERE order_id = <order_id>          | +              |




## 6.  REST-запросы

| Название          | URL          | Тип метода | Проверка авторизации       | Действия на бэкенде                      | Query, path parameters              | Body          | Responses  |
|-------------------|--------------|------------|----------------------------|------------------------------------------|-------------------------------------|---------------|--------------------------------------------|
| Внесение данных   | /api/orders  | POST       | Базовая авторизация        | Создание нового заказа с данными из JSON | id: Uuid, обязательный, ИД сущности | JSON с данными| 200OK:{"message":"Заказ успешно создан"} <br> 400 Bad Request: {"message": "Неверные данные"}|
|                   |              |            |                            |                                          |                                     |               |                                            |
| Оплата            | /api/orders{id}/payment   |POST  | Базовая авторизация | Проверка данных и проведение оплаты      | id: Uuid, обязательный, ИД сущности |JSON с данными оплаты | 200OK:{"message":"Оплата успешно проведена"} <br> 404 Not Found: {"message":"Заказ не найден"}| 
|                   |              |            |                            |                                          |                                     |               |                                            |
|  Удаление заказа  |/api/orders/{id}| DELETE   | Базовая авторизация        | Удаление заказа по ИД                    | id: Uuid, обязательный, ИД сущности |      -        | 200OK:{"message": "Заказ успешно удален" <br> 404 Not Found: {"message":"Заказ не не найден"}|
|                   |              |            |                            |                                          |                                     |               |                                            |


## 7.  Пример JSON для создания заказа

```json
{
  "productInfo": {
    "product_name": "Проводные наушники HyperX Cloud II KHX-HSCP-RD красный",
    "article": "12h12322",
    "rating": 4.6,
    "price": 13272
  },

  "clientInfo": {
    "firstName": "Орлов",
    "lastName": "Александр",
    "middleName": "Кириллович"
  },

  "contactInfo": {
    "email": "example@mail.ru",
    "phone": "+7 777 777 02 03"
  },

  "deliveryOfGoods": {
    "inStore": {
      "included": true
    },
    "deliveryToAddress": {
      "included": false,
      "address": "г.Клин Ул.полевая д.45",
      "price": 2000
    }
  },

  "paymentMethod": {
    "SBP": {
      "included": false
    },
    "SBERPay": {
      "included": false
    },
    "card": {
      "included": false
    },
    "cashOnDelivery": {
      "included": true
    }
  }
}
```

## 8.  Пример JSON для оплаты заказа

```json
{
  "paymentMethod": {
    "SBP": {
      "included": false
    },
    "SBERPay": {
      "included": false
    },
    "card": {
      "included": true,
      "cardNumber": "1234567890123456",
      "cardHolderName": "Иван Иванов",
      "expiryDate": "12/23",
      "cvv": "123"
    },
    "cashOnDelivery": {
      "included": false
    }
  }
}
```

# PIS_LR_6

## 1. Выбор архитектуры
Для данной системы выбрал *микросервисную* архитектуру так как она наиболее подходящая. 
Этот выбор обоснован следующими причинами:

- Масштабируемость: *Микросервисная* архитектура позволяет легко масштабировать отдельные компоненты системы в зависимости от нагрузки. Например, сервис оплаты может быть масштабирован независимо от сервиса управления корзиной.
- Гибкость и адаптивность: Микросервисы могут быть разработаны, развернуты и обновлены независимо друг от друга, что упрощает внедрение новых функций и исправление ошибок.
- Интеграция со сторонними системами: Микросервисная архитектура упрощает интеграцию с внешними системами, такими как платежные шлюзы, системы управления складом.
- Отказоустойчивость: В случае сбоя одного из микросервисов, остальные части системы могут продолжать работать, что повышает общую надежность системы.


## 2. Основные компоненты
- Frontend: React 
- Backend Services: Spring Boot
- Database: PostgreSQL
- Внешний сервис: Payment Service (например, Sberpay или PayPal)

![image](https://github.com/user-attachments/assets/9af9515e-501c-42cd-a29e-50a8af04758c)

### 2.1. Компоненты системы
Система будет состоять из следующих микросервисов:

- User Service: Управление пользователями и их данными.
- Product Service: Управление товарами и их данными.
- Cart Service: Управление корзиной пользователя.
- Order Service: Управление заказами.
- Payment Service: Обработка платежей.
- Delivery Service: Управление доставкой.
- Notification Service: Уведомления пользователей.
- Интеграция со сторонней системой
- Для обработки платежей система будет интегрирована с платежным шлюзом, таким как Sberpay или PayPal. Взаимодействие с платежным шлюзом будет осуществляться через REST API.

![image](https://github.com/user-attachments/assets/0f3e97f6-bd3a-4048-aa8e-0bf312e5c8ae)


## 3. Взаимодействие между сервисами:
- Синхронное взаимодействие: Используютя для операций, требующих немедленного ответа, таких как добавление товара в корзину или создание заказа. Протокол: HTTP/REST.
- Асинхронное взаимодействие: Используютя для операций, которые могут выполняться в фоновом режиме, таких как уведомления пользователей или обработка платежей. Протокол: HTTP/REST/Kafka/TLS.
 
### Все Протоколы связи которые используются
- От сервиса к сервису: HTTP/REST
- Обработка платежей: HTTP/REST/Kafka/TLS.
- Работа с БД: JDBC
