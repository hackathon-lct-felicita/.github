#### Хакатон "Лидеры цифровой трансформации" 2025
---
# Команда "Феличита"
## Бизнес | Задача 10. Сервис выделения сущностей из поискового запроса клиента в мобильном приложении торговой сети “Пятерочка” | X5

## 🔹 Описание
Высоконагруженный веб-сервис, который извлекает сущности из поисковых запросов клиентов торговой сети. Ядро решения — мультиязычная BERT-модель, обогащенная данными OpenFoodFacts и актуального каталога товаров с сайта «Пятёрочки». Уникальность решения — сочетание NER (Named Entity Recognition) с семантическим поиском и привязкой сущностей к реальным товарам из ассортимента сети.

Система представляет собой многоуровневый веб-сервис, поддерживающий три режима работы:  
1. **Тестовый режим** (облегченный локальный запуск) — пользователь взаимодействует напрямую с моделью.
2. **RPC-режим** (надстрйока над п.1 с возможностью гибкого горизонтального масштабирования и observability) — пользователь обращается к модели через `nginx`, `bot-test-gateway` и `RabbitMQ`.
3. **Полный режим** (надстройка над п.2 с демонстрацией реальной работы) — пользователь работает через `frontend`, `backend`, очередь сообщений и поисковый движок `OpenSearch`.  

## 🔹 Демонстрация Полного режима
https://github.com/user-attachments/assets/63058a90-6f74-4c68-bbd8-2ffd61186cf6


## 🔹 Технические требования
Для корректного запуска необходимо:  
- **ОС**: Linux / macOS / Windows (WSL2 рекомендуется)  
- **Python**: >= 3.11  
- **Docker** и **Docker Compose**: >= 20.10  
- **uv** (менеджер зависимостей Python): [uv installation](https://github.com/astral-sh/uv)  
- **Git**: >= 2.40


## 1. Тестовый режим
### model

```bash
git clone git@github.com:hackathon-lct-felicita/model.git
cd model
uv run python run.py
```

Подробнее: https://github.com/hackathon-lct-felicita/model/blob/main/README.md


## 2. RPC-режим

```mermaid
sequenceDiagram
    participant Bot as @x5_ner_hack_bot
    participant Nginx as nginx
    participant Gateway as bot-test-gateway
    participant RabbitMQ as RabbitMQ
    participant Model as Model Service

    Bot->>+Nginx: POST /api/predict<br/>{input: "текст для анализа"}
    
    Nginx->>+Gateway: Пересылка HTTP запроса
    
    Gateway->>Gateway: Обработка middleware<br/>(заголовки, метрики)
    
    Gateway->>+RabbitMQ: RPC запрос к модели
    
    RabbitMQ->>+Model: Пересылка запроса
    
    Model->>Model: Обработка NER предсказания<br/>(анализ входного текста)
    
    Model-->>-RabbitMQ: Результаты предсказания
    
    RabbitMQ-->>-Gateway: RPC ответ
    
    Gateway->>Gateway: Запись метрик
    
    Gateway-->>-Nginx: HTTP 200<br/>Результаты NER
    
    Nginx-->>-Bot: Пересылка ответа
```


### nginx
```bash
git clone git@github.com:hackathon-lct-felicita/nginx.git
cd nginx
docker compose up --build
```

### bot-test-gateway
```bash
git clone git@github.com:hackathon-lct-felicita/bot-test-gateway.git
cd bot-test-gateway
uv run python run.py
# или: docker compose up --build
```

Подробнее: https://github.com/hackathon-lct-felicita/bot-test-gateway/blob/main/README.md


### rabbitmq

```bash
git clone git@github.com:hackathon-lct-felicita/rabbitmq.git
cd rabbitmq
docker compose up
```

### model
```bash
git clone git@github.com:hackathon-lct-felicita/model.git
cd model
uv run python run_rpc.py
```

Подробнее: https://github.com/hackathon-lct-felicita/model/blob/main/README.md

## 3. Полный режим
```mermaid
sequenceDiagram
    participant U as User
    participant N as Nginx
    participant F as Frontend
    participant B as Backend<br/>(FastAPI)
    participant OS as OpenSearch
    participant RMQ as RabbitMQ
    participant ML as ML Model<br/>(NER Service)

    Note over U, ML: 1. Поиск продуктов с NER

    U->>N: GET /search?q="молоко Простоквашино 1л"
    Note right of U: Пользователь вводит запрос

    N->>F: Проксирует запрос
    Note right of N: Nginx перенаправляет на frontend

    F->>B: GET /search?q="молоко Простоквашино 1л"
    Note right of F: Frontend отправляет запрос в backend

    activate B

    B->>B: SearchProductsWithNERUseCase.execute()
    Note right of B: Запуск use case для поиска с NER

    Note over U, ML: 2. Извлечение сущностей через NER

    B->>RMQ: Подключение к RabbitMQ
    Note right of B: Установка соединения с RabbitMQ

    B->>RMQ: RPC Call: predict("молоко Простоквашино 1л")
    Note right of B: Отправка текста на анализ

    RMQ->>ML: Передача запроса в очередь model_rpc_queue
    Note right of RMQ: Запрос попадает в очередь ML сервиса

    activate ML
    ML->>ML: NER анализ текста
    Note right of ML: ML модель извлекает сущности:<br/>- "молоко" -> B-TYPE<br/>- "Простоквашино" -> B-BRAND<br/>- "1л" -> B-VOLUME

    ML->>RMQ: Возврат сущностей
    Note right of ML: Возврат списка сущностей с индексами

    RMQ->>B: RPC Response: [ApiPredictResponse]
    Note right of RMQ: Получение результата NER анализа
    deactivate ML

    Note over U, ML: 3. Обработка сущностей

    B->>B: EntityService.filter_entities()
    Note right of B: Фильтрация сущностей:<br/>- Исключение неизвестных типов<br/>- Включение O сущностей

    B->>B: EntityService.group_entities_by_type()
    Note right of B: Группировка по типам:<br/>- TYPE: ["молоко"]<br/>- BRAND: ["Простоквашино"]<br/>- VOLUME: ["1л"]

    B->>B: EntityService.get_search_terms_from_entities()
    Note right of B: Извлечение поисковых терминов:<br/>- Приоритет TYPE сущностям<br/>- Добавление BRAND, VOLUME, PERCENT<br/>- Включение O сущностей

    B->>B: Комбинирование запроса
    Note right of B: Создание расширенного запроса:<br/>"молоко Простоквашино 1л молоко Простоквашино 1л"

    Note over U, ML: 4. Поиск в OpenSearch

    B->>OS: Поиск продуктов по расширенному запросу
    Note right of B: Поиск с учетом извлеченных сущностей

    activate OS
    OS->>OS: Полнотекстовый поиск
    Note right of OS: Поиск по полям name, description<br/>с нечетким соответствием

    OS->>B: Результаты поиска: [Product]
    Note right of OS: Возврат найденных продуктов
    deactivate OS

    Note over U, ML: 5. Возврат результата

    B->>F: HTTP 200: [Product]
    Note right of B: Возврат результатов поиска

    F->>N: HTTP Response
    Note right of F: Frontend возвращает ответ

    N->>U: Отображение результатов
    Note right of N: Пользователь видит найденные продукты

    deactivate B

    Note over U, ML: Обработка ошибок

    Note over B, ML: Если NER сервис недоступен:
    B->>B: Fallback на обычный поиск
    Note right of B: Использование оригинального запроса<br/>без извлечения сущностей

    B->>OS: Поиск по оригинальному запросу
    OS->>B: Результаты поиска
    Note right of OS: Система продолжает работать<br/>даже при недоступности NER
```


### frontend-ngnix
```bash
git clone git@github.com:hackathon-lct-felicita/frontend.git
cd frontend
docker compose up --build
```

Подробнее: https://github.com/hackathon-lct-felicita/frontend/blob/main/README.md


### backend
```bash
git clone git@github.com:hackathon-lct-felicita/backend.git
cd backend
uv run python run.py
# или: docker compose up --build
```

### rabbitmq
```bash
git clone git@github.com:hackathon-lct-felicita/rabbitmq.git
cd rabbitmq
docker compose up
```

### model
```bash
git clone git@github.com:hackathon-lct-felicita/model.git
cd model
uv run python run_rpc.py
```

Подробнее: https://github.com/hackathon-lct-felicita/model/blob/main/README.md


### opensearch
```bash
git clone git@github.com:hackathon-lct-felicita/opensearch.git
cd opensearch
docker compose up
```




