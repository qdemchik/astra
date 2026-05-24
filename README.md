# DomainSentry — Аналитическая платформа мониторинга доменов .RU/.РФ

## Архитектура

```
domain_platform/
├── backend/
│   ├── app.py              # Flask REST API (главный сервер)
│   ├── domain_feed.py      # Источники новых доменов (CT-лог, whoisds, tcinet)
│   ├── crawler.py          # HTTP/Playwright краулер + извлечение признаков
│   ├── clone_detector.py   # Обнаружение клонов (TF-IDF, DOM, visual hash)
│   ├── classifier.py       # Классификация сайтов по типу
│   ├── search_engine.py    # Поиск по URL-путям и тексту
│   └── worker_manager.py   # Управление SSH-воркерами
├── frontend/
│   └── index.html          # Web-интерфейс (SPA, без зависимостей)
├── scripts/
│   └── find_astra_clone.py # Скрипт поиска клона astra.ru (задание 3)
├── data/                   # Кэш доменов и результаты задач
└── requirements.txt
```

---

## Быстрый старт

### 1. Установка зависимостей

```bash
pip install -r requirements.txt
playwright install chromium
```

### 2. Запуск API-сервера

```bash
cd путь\до\domain_platform\backend
python app.py
# Сервер запустится на http://0.0.0.0:5001
```

### 3. Открыть веб-интерфейс
Откройте новую вкладку в браузере. В адресной строке введите:

```
file:///путь/до/domain_platform/frontend/index.html
```

---

## Задание 3: Поиск клона astra.ru

Поиск домена .RU, зарегистрированного с 21 апреля 2026, с содержимым идентичным astra.ru.

### Способ 1 — через CLI-скрипт (рекомендуется)

```bash
cd scripts
python find_astra_clone.py
```

Скрипт:
1. Загружает fingerprint сайта astra.ru (заголовок, текст, DOM-структура, CSS-классы)
2. Получает список новых .RU доменов из CT-лога (crt.sh), whoisds.com и вариаций имени
3. Для каждого домена вычисляет TF-IDF косинусное сходство + структурное + классовое
4. Выводит совпадения и извлекает флаг с главной страницы клона

### Способ 2 — через API

```bash
# Запустить поиск
curl -X POST http://localhost:5000/api/clone/detect \
  -H "Content-Type: application/json" \
  -d '{
    "target_url": "https://astra.ru",
    "methods": ["text", "structure"],
    "threshold": 0.75,
    "since_date": "2026-04-21"
  }'
# Ответ: {"job_id": "abc123", "status": "started"}

# Получить результаты (опрашивать до status=done)
curl http://localhost:5000/api/clone/job/abc123
```

### Способ 3 — через веб-интерфейс

1. Открыть frontend/index.html
2. Перейти в «Поиск клонов»
3. URL-эталон: `https://astra.ru`, дата: `2026-04-21`
4. Нажать «Найти клоны»

---

## API — краткий справочник

| Метод  | Путь                        | Описание |
|--------|-----------------------------|----------|
| GET    | /api/stats                  | Статистика в реальном времени |
| POST   | /api/stats/refresh          | Принудительное обновление |
| GET    | /api/domains/new            | Новые домены (page, per_page, zone, since) |
| GET    | /api/domains/dropped        | Освобождающиеся домены |
| GET    | /api/domains/{domain}/info  | WHOIS + DNS + Geo |
| POST   | /api/search/url-path        | Поиск URL-пути на доменах |
| POST   | /api/search/text            | Поиск текста на главных страницах |
| GET    | /api/search/job/{id}        | Статус задачи поиска |
| POST   | /api/clone/detect           | Поиск клонов/фишинга |
| GET    | /api/clone/job/{id}         | Статус задачи клонирования |
| POST   | /api/clone/compare          | Прямое сравнение двух URL |
| POST   | /api/classify               | Классификация одного сайта |
| POST   | /api/classify/batch         | Пакетная классификация |
| GET    | /api/workers                | Список SSH-воркеров |
| POST   | /api/workers/add            | Добавить воркер по SSH |
| DELETE | /api/workers/{id}/remove    | Удалить воркер |
| POST   | /api/workers/distribute     | Распределить задачу по воркерам |
| GET    | /api/health                 | Health check |

---

## Источники данных о новых доменах

| Источник | Описание | Задержка |
|----------|----------|----------|
| crt.sh (Certificate Transparency) | SSL-сертификаты → новые сайты | ~часы |
| whoisds.com | Ежедневные списки новых доменов | ~1 день |
| tcinet.ru FTP | Zone file .RU/.РФ (ftp://ftp.tcinet.ru/pub/zone/) | ~1 день |
| RU-CENTER dropped | Освобождающиеся домены | реальное время |

---

## Методы обнаружения клонов

### 1. Текстовое сходство (TF-IDF косинус)
Сравнение нормализованного текста страниц через векторное пространство токенов. Вес: 50%.

### 2. Структурное сходство (DOM)
Косинусное сходство по вектору количества HTML-тегов. Вес: 30%.

### 3. CSS-классы (Jaccard)
Пересечение множеств CSS-классов. Вес: 20%.

### 4. Визуальное сходство (Perceptual Hash)
Скриншоты через Playwright → grayscale 32×32 → pHash → расстояние Хэмминга.

### 5. Внешние ссылки (Jaccard)
Пересечение множеств внешних ссылок.

---

## Распределённая архитектура

Воркеры подключаются по SSH. Мастер:
1. Делит список доменов поровну между воркерами
2. Загружает скрипт `/tmp/worker.py` через SFTP
3. Выполняет задачу параллельно через `ssh.exec_command()`
4. Собирает JSON-результаты из stdout

```bash
# Добавить воркер
curl -X POST http://localhost:5000/api/workers/add \
  -H "Content-Type: application/json" \
  -d '{"host":"192.168.1.10","user":"ubuntu","key_path":"~/.ssh/id_rsa"}'

# Распределить задачу
curl -X POST http://localhost:5000/api/workers/distribute \
  -H "Content-Type: application/json" \
  -d '{"task_type":"crawl","domains":["site1.ru","site2.ru"],"params":{}}'
```

---

## Классификация сайтов

Категории: `shop`, `forum`, `landing`, `blog`, `news`, `corporate`,
`phishing`, `parked`, `adult`, `malware`, `government`, `education`, `other`

Алгоритм: keyword scoring по тексту + URL-паттерны + структурные признаки.

---

## Зависимости

- Python 3.9+
- Flask + flask-cors — REST API
- requests + BeautifulSoup4 + lxml — парсинг
- Playwright + Chromium — рендеринг JS-сайтов и скриншоты
- paramiko — SSH-подключение к воркерам
- Pillow + numpy — visual hashing
- scikit-learn — (опционально) ML-классификация

```bash
pip install flask flask-cors requests beautifulsoup4 lxml playwright paramiko Pillow numpy scikit-learn
playwright install chromium
```
