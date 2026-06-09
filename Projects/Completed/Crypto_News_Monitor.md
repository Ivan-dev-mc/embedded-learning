
project: Crypto & VPN News Monitor
status: ✅ Работает (Production)
stack: [Python, Requests, Feedparser, Telegram Bot API]
tags: [project, automation, monitoring, python, telegram]
created: 2026-05-22
links: [[VPN_personal]], [[Shopping_list]] 

# 📊 Crypto & VPN News Monitor

## 📋 Назначение
**Автоматический мониторинг криптовалют и новостей про VPN/блокировки**

Скрипт собирает данные из разных источников и отправляет сводку в Telegram.

## 🏗 Архитектура

### Источники данных
| Источник | API | Данные |
|----------|-----|--------|
| **CoinGecko** | REST API | Цены BTC/ETH в USD/RUB |
| **ЦБ РФ** | JSON API | Официальные курсы валют |
| **Habr, Interfax, Lenta, OpenNet** | RSS | Новости технологий |

### Фильтрация новостей
Ключевые слова:
- 🔴 **VPN/Блокировки**: `vpn`, `блокировк`, `tcpup`, `dpi`, `vless`, `reality`, `роскомнадзор`
- 🟡 **Законы**: `закон`, `запрет`, `статья`, `уголовн`, `штраф`
- 🟢 **Финансы**: `ипотек`, `налог`, `кредит`, `инфляц`

### Вывод
- Telegram-бот (автоматическая отправка)
- Консоль (для локальной отладки)

## 🔧 Технические детали

### Зависимости
```bash
pip install requests feedparser
```

### Структура
```python
#!/usr/bin/env python3
import requests            # HTTP-запросы
import feedparser          # RSS-парсинг
from datetime import datetime

# Telegram Bot
BOT_TOKEN = "..."
CHAT_ID = "..."

def send_tg(text):
    # Отправка в Telegram

def log(text):
    # Вывод в консоль + сохранение для ТГ

# 1. Криптовалюты (CoinGecko API)
# 2. Курсы валют (ЦБ РФ API)
# 3. RSS-парсинг новостей
# 4. Отправка в Telegram
```

## 📊 Результаты
- ✅ Работает стабильно
- ✅ Актуальная информация о блокировках
- ✅ Мониторинг крипты в одном сообщении

## 🔗 Связи
- [[VPN_personal]] → мониторинг угроз для моего сервера
- [[Shopping_list]] → не требует железа, чисто софт

## 🚀 Улучшения (бэклог)
- [ ] Добавить мониторинг других крипто-бирж
- [ ] Веб-интерфейс вместо Telegram
- [ ] База данных для истории цен
- [ ] Уведомления при резких изменениях курса

---

**Статус:** ✅ Production  
**Обновление:** По расписанию (cron/systemd timer)
## 💻 Исходный код: `crypto_monitor.py`

```python
#!/usr/bin/env python3
# ^ Shebang — говорит системе, что это Python 3 скрипт

import requests            # Библиотека для HTTP-запросов (к API)
import feedparser          # Библиотека для чтения RSS-лент (новости)
from datetime import datetime  # Модуль для работы с датой и временем

# === ⚠️ БЕЗОПАСНОСТЬ: Токены лучше выносить в переменные окружения ===
# Для локального тестирования можно оставить так, но НЕ коммить в публичный Git!
BOT_TOKEN = "8603377562:AAFyJY3KCqfLnZVgiPROQrKP72ZixgDrc8I"
CHAT_ID = "754628300"

def send_tg(text):
    """Отправляет сообщение в Telegram"""
    try:
        requests.post(
            f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage",
            data={"chat_id": CHAT_ID, "text": text},
            timeout=10
        )
    except Exception as e:
        print(f"❌ Telegram error: {e}")

# Список для хранения всех сообщений (буфер)
output_lines = []

def log(text=""):
    """Выводит текст на экран и сохраняет для отправки в Telegram"""
    print(text)                # Показываем в консоли
    output_lines.append(text)  # Сохраняем для ТГ

# === ЗАГОЛОВОК ===
log("=" * 50)  # Печатает разделительную линию (50 знаков =)
log(f"📊 Крипто-мониторинг | {datetime.now().strftime('%H:%M:%S')}")
log("=" * 50)
log()  # Пустая строка для красоты

# === ЧАСТЬ 1: КРИПТОВАЛЮТЫ ===
log("💰 КРИПТОВАЛЮТЫ")
log("-" * 50)  # Разделитель (тире)

# Список криптовалют для отслеживания
cryptos = ['bitcoin', 'ethereum']

# Цикл: для каждой криптовалюты в списке
for crypto in cryptos:
    try:        # Формируем URL API CoinGecko
        url = f'https://api.coingecko.com/api/v3/simple/price?ids={crypto}&vs_currencies=usd,rub'

        # Делаем GET-запрос к API (ждем не более 10 секунд)
        response = requests.get(url, timeout=10)

        # Парсим JSON-ответ (превращаем текст в словарь Python)
        data = response.json()

        # Проверяем, есть ли нужная крипта в ответе
        if crypto in data:
            # Достаём цену в USD
            price_usd = data[crypto]['usd']
            # Достаём цену в RUB (если есть, иначе 0)
            price_rub = data[crypto].get('rub', 0)

            # Выбираем красивое имя для вывода
            name = 'Bitcoin' if crypto == 'bitcoin' else 'Ethereum'

            # Выводим результат
            log(f"{name}")
            log(f"   USD: ${price_usd:,}")  # :, — добавляет запятые (1,000,000)
            if price_rub:  # Если цена в рублях не нулевая
                log(f"   RUB: {price_rub:,.0f} ₽")  # :,.0f — запятые, без десятичных
            log()  # Пустая строка

    # Если что-то пошло не так (нет интернета, API недоступен)
    except Exception as e:
        log(f"❌ Ошибка получения {crypto}: {e}\n")

# === ВАЛЮТЫ (ЦБ РФ) ===
log("💵 ВАЛЮТЫ (ЦБ РФ)")
log("-" * 50)
try:
    url = 'https://www.cbr-xml-daily.ru/daily_json.js'
    response = requests.get(url, timeout=10)
    data = response.json()
    usd = data['Valute']['USD']
    eur = data['Valute']['EUR']

    log(f"USD: {usd['Value']:,.2f} ₽ (номинал: {usd['Nominal']})")
    log(f"EUR: {eur['Value']:,.2f} ₽ (номинал: {eur['Nominal']})")
    log()
except Exception as e:
    log(f"❌ Ошибка получения курсов: {e}\n")

# === ЧАСТЬ 2: НОВОСТИ ===
log("📰 НОВОСТИ (VPN, блокировки)")
log("-" * 50)
# RSS-ленты для парсинга
feeds = [
    'https://habr.com/ru/rss/articles/',          # IT и технологии
    'http://www.interfax.ru/rss.asp',             # Новости
    'https://lenta.ru/rss/news',                  # Альтернатива
    'https://www.opennet.ru/opennews/opennews_all.rss'  # OpenNet
]

# Ключевые слова для фильтрации новостей
keywords = [
    # Интернет и блокировки
    'vpn', 'блокировк', 'заблок', 'тспу', 'dpi', 'vless', 'reality',
    'роскомнадзор', 'цензур', 'интернет', 'сеть', 'запрет',
    # Законы и изменения
    'закон', 'запрет', 'огранич', 'упраздн', 'ликвид', 'статья',
    'уголовн', 'штраф', 'обязательн', 'сексолог', 'професс',
    # Финансы
    'ипотек', 'налог', 'кредит', 'ставка', 'инфляц',
]

# Флаг: нашли ли хоть одну новость
news_found = False

# Заголовки, чтобы сайты не блокировали (User-Agent)
headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
}

for feed_url in feeds:
    try:
        response = requests.get(feed_url, headers=headers, timeout=20)
        feed = feedparser.parse(response.content)

        for post in feed.entries[:3]:  # Берём первые 3 новости из каждой ленты
            title = post.title.lower()

            # Фильтр по ключевым словам
            if any(keyword in title for keyword in keywords):
                log(f"🔥 {post.title}")
                log(f"   {post.link}")
                log()
                news_found = True

    except Exception as e:
        print(f"❌ Ошибка парсинга {feed_url}: {e}")
        pass  # Тихо пропускаем ошибки

if not news_found:
    log("⚪ Новостей по теме не найдено\n")
# === ФИНАЛ ===
log("=" * 50)
log("✅ Готово")

# === ОТПРАВКА В TELEGRAM ===
full_message = "\n".join(output_lines)

if full_message.strip():
    send_tg(full_message)
```

> [!warning] ⚠️ Безопасность #insight 
> 1. **Не коммить токен в публичный репозиторий!**
> 2. Для продакшена используй переменные окружения:
>    ```python
>    import os
>    BOT_TOKEN = os.getenv('TELEGRAM_BOT_TOKEN')
>    CHAT_ID = os.getenv('TELEGRAM_CHAT_ID')
>    ```
> 3. Запуск:
>    ```bash
>    export TELEGRAM_BOT_TOKEN="8603377562:AAF..."
>    export TELEGRAM_CHAT_ID="754628300"
>    python3 crypto_monitor.py
>    ```

> [!tip] Зависимости 
> ```bash
> pip install requests feedparser
> ```

> [!example] Запуск по расписанию (cron) 
> ```bash
> # Открываем crontab
> crontab -e
>
> # Запуск каждый день в 9:00 и 18:00
> 0 9,18 * * * /usr/bin/python3 /path/to/crypto_monitor.py
> ```