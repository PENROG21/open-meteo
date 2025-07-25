# ETL-пайплайн для данных Open-Meteo

Этот проект реализует ETL-пайплайн для извлечения, трансформации и загрузки метеоданных с API [Open-Meteo](https://open-meteo.com/). Данные преобразуются в заданный формат, сохраняются в CSV и загружаются в PostgreSQL. Поддерживается гибкая настройка через Docker и параметры командной строки.

---

## 🚀 Что делает приложение?

Приложение выполняет следующие шаги:

1. **Extract (Извлечение)**  
   Отправляет HTTP-запрос к API Open-Meteo с указанными датами (`start_date`, `end_date`) и получает почасовые и ежедневные данные.

2. **Transform (Трансформация)**  
   - Конвертирует единицы измерения:
     - Температура: °F → °C
     - Осадки: дюймы → мм
     - Скорость ветра: узлы → м/с
     - Видимость: футы → метры
   - Агрегирует данные за 24 часа и **световой день (daylight)**:
     - Средние значения температуры, влажности, скорости ветра
     - Суммарные осадки
   - Добавляет вычисляемые поля:
     - `daylight_hours` — продолжительность светового дня в часах
     - `sunrise_iso`, `sunset_iso` — время восхода и заката в формате ISO 8601

3. **Load (Загрузка)**  
   - Сохраняет результат в CSV-файл: `weather_data.csv`
   - Загружает данные в PostgreSQL с защитой от дубликатов

---


---

## ⚙️ Как запустить?

### 1. Установите зависимости
- Docker
- Docker Compose

### 2. Настройте переменные окружения
Создайте файл `.env` в корне проекта:
```env
DB_USER=users
DB_PASSWORD=password
DB_HOST=db
DB_PORT=5432
DB_NAME=database
START_DATE=2025-05-16
END_DATE=2025-05-30
```

### 3. Запустите пайплайн
```bash
docker-compose up --build
```
После запуска:

Данные будут сохранены в etl_code/weather_data.csv
Таблица daily_weather будет создана и заполнена в PostgreSQL

## 🐳 Docker и тестовое окружение

Проект полностью контейнеризирован:
- `Dockerfile`: Сборка образа с Python и зависимостями
- `docker-compose.yml`:
  - Сервис `db`: PostgreSQL 16 с автосозданием таблицы через `init.sql`
  - Сервис `etl`: Запуск ETL-скрипта
  - Добавлен `healthcheck` для ожидания готовности БД

Поддерживается гибкость:
- Можно менять даты в `.env`
- Можно расширить `main.py` для поддержки разных режимов (`--mode api_to_csv`, `--mode api_to_db` и т.д.)

---

## 🛠️ Общее качество кода

- **Чистота и читаемость**: Код разбит на модули по ответственности (ETL, БД, утилиты)
- **Типизация**: Используются аннотации типов (в `data_base.py`)
- **Логирование**: Используется `logging` для отслеживания подключения и ошибок
- **Обработка ошибок**: Все ключевые операции (запросы, БД) обернуты в `try-except`
- **PEP8**: Код соответствует стандартам Python

---

## 🧠 Пояснение решений

### 🔐 Борьба с дубликатами
При вставке в PostgreSQL используется конструкция:
```sql
ON CONFLICT (date) DO NOTHING
```
**Обоснование**:
- Поле `date` объявлено как `PRIMARY KEY` в таблице `daily_weather`
- Это гарантирует, что при попытке вставить данные за уже существующую дату — дубликат будет проигнорирован
- Подход **идемпотентен**, что критично для повторных запусков ETL
- Альтернативы (например, `UPDATE`) не требуются, так как данные за день не меняются

### 🌞 Обработка светового дня
Для расчёта агрегатов за световой день:
1. Извлекаются `sunrise` и `sunset` (в Unix time)
2. Для каждого часа в `hourly` проверяется, попадает ли он в интервал светового дня
3. Только эти часы участвуют в агрегации (`mean`, `sum`)

Это обеспечивает точность в соответствии с "Итоговой таблицей".

---
