
created: 2026-05-31
tags: [ai, backlog, embedded, ecosystem, ideas]
links: [[Pocket_engineer]], [[Runout_Scanner]], [[Pi400_workstation]], [[ESP32_Watch]], [[Embedded_Plan]]

# 🧠 AI Integration: Детальные спецификации идей

> [!summary] Философия
> Каждая идея — это не «фича ради фичи». Это ответ на боль: «нет ноутбука», «руки в масле», «нужно знать статус».
> Реализуем по приоритету: боль × навык / время.

---

## 🔴 Приоритет 1: Auto-Diagnose Mode (в Pocket Engineer)

### 🎯 Какую боль закрывает
«В цеху датчик не отвечает. Ноутбук далеко / занят / его нет. Нужно быстро понять: провод отошёл, питание пропало или датчик сгорел».

### 🏗 Архитектура
```
[Отлаживаемое устройство]
         │ (UART / I²C / GPIO)
         ▼
[Pocket Engineer: ESP32]
         │
         ├──> Сбор логов (буфер 256-512 байт)
         ├──> Парсинг по правилам (expert system)
         ├──> Вывод диагноза на OLED
         └──> (опционально) Отправка на Pi400 для LLM-анализа
```

### 🔧 Техническая реализация (уровень 1: экспертная система)

**Структура правил (`diagnosis_rules.h`):**
```c
struct DiagnosticRule {
    const char* pattern;      // Подстрока для поиска в логе
    const char* cause;        // Вероятная причина
    const char* solution[3];  // Шаги решения (массив, т.к. может быть несколько)
    uint8_t severity;         // 1=info, 2=warning, 3=critical
};

// Примеры правил
const DiagnosticRule rules[] = {
    {
        .pattern = "VL53L1X NOT FOUND",
        .cause = "Устройство не отвечает на шине I²C",
        .solution = {
            "Проверь питание 3.3В на датчике",
            "Проверь подтяжки 4.7кΩ на SDA/SCL",
            "Убедись, что адрес 0x29 не занят другим устройством"
        },
        .severity = 3
    },
    {
        .pattern = "TIMEOUT",
        .cause = "Превышено время ожидания ответа",
        .solution = {
            "Проверь тактирование и прерывания",
            "Добавь soft-reset перед повторной инициализацией",
            "Увеличи таймаут, если шина загружена"
        },
        .severity = 2
    },
    {
        .pattern = "NACK",
        .cause = "Устройство прислало NACK (не подтвердило байт)",
        .solution = {
            "Проверь адрес устройства (7-bit vs 8-bit)",
            "Убедись, что устройство не занято другой операцией",
            "Проверь целостность проводов"
        },
        .severity = 2
    }
};
```

**Алгоритм анализа (`diagnose.cpp`):**
```c
void analyzeLog(const char* logBuffer, DiagnosticResult* result) {
    for (const auto& rule : rules) {
        if (strstr(logBuffer, rule.pattern) != nullptr) {
            result->found = true;
            result->cause = rule.cause;
            result->severity = rule.severity;
            
            // Копируем массив решений
            for (int i = 0; i < 3 && rule.solution[i] != nullptr; i++) {
                result->solutions[i] = rule.solution[i];
            }
            return;
        }
    }
    
    // Если не нашли правило — «неизвестная ошибка»
    result->found = false;
    result->cause = "Неизвестная ошибка — отправь лог на сервер";
    result->severity = 1;
}
```

**Интеграция в LAB_PROFILE:**
```c
void lab_profile_loop() {
    // ... чтение из UART устройства ...
    
    if (newLogAvailable()) {
        const char* log = getLogBuffer();
        DiagnosticResult diagnosis;
        analyzeLog(log, &diagnosis);
        
        if (diagnosis.found) {
            display.clear();
            display.print("⚠️ Диагноз:");
            display.print(diagnosis.cause);
            
            // Показываем решения построчно, если есть место
            for (int i = 0; i < 3 && diagnosis.solutions[i]; i++) {
                display.print(fix::scroll(diagnosis.solutions[i]));
            }
            
            // Цвет по серьёзности
            if (diagnosis.severity >= 3) display.setColor(RED);
            else if (diagnosis.severity == 2) display.setColor(YELLOW);
        }
    }
}
```

### 📊 Что нужно для MVP (1-2 вечера)
- [ ] Создать файл `diagnosis_rules.h` с 3-5 правилами для `[[Runout_Scanner]]`
- [ ] Реализовать функцию `analyzeLog()` (50 строк кода)
- [ ] Добавить вывод на OLED в режиме LAB_PROFILE
- [ ] Протестировать: симулировать ошибку → увидеть диагноз

### 🚀 Следующий уровень (TinyML / LLM)
| Уровень | Что добавляет | Сложность |
|---------|--------------|-----------|
| **TinyML** | Распознавание новых паттернов, не описанных в правилах | 🔴 Высокая (нужен датасет, обучение, ESP32-S3) |
| **Гибрид (LLM)** | Понимание контекста, генерация решений «с нуля» | 🟡 Средняя (нужен Pi400 + Ollama + сеть) |

---

## 🟡 Приоритет 2: Голосовые макросы (для работы в шинке)

### 🎯 Какую боль закрывает
«Руки в масле / в перчатках. Нужно записать «балансировка ×4», но телефон доставать неудобно, клавиатура — не вариант».

### 🏗 Архитектура
```
[Голос] 
   │
   ▼
[Микрофон → ESP32-S3 / Pi400]
   │
   ├──> Вариант А (локально, оффлайн):
   │    Whisper Tiny (квантованный) → Intent Parser → MQTT
   │
   └──> Вариант Б (гибрид, точнее):
        Запись → отправка на Pi400 → Whisper Medium → LLM (распознавание намерения) → действие
```

### 🔧 Техническая реализация (вариант Б — реалистичный)

**Шаг 1: Запись и отправка (Pocket Engineer)**
```c
// При нажатии кнопки "голос":
void startVoiceCommand() {
    // Запись 3-5 секунд с микрофона (если подключен)
    // Или: активация режима "ожидание команды с телефона по BLE"
    
    // Отправка на Pi400
    http.begin("http://192.168.1.100:8000/voice");
    http.addHeader("Content-Type", "audio/wav");
    http.POST(audioBuffer, bufferSize);
}
```

**Шаг 2: Распознавание намерения (Pi400)**
```python
# FastAPI endpoint на Pi400
@app.post("/voice")
async def voice_command(audio: UploadFile):
    # 1. Распознавание речи (Whisper)
    import whisper
    model = whisper.load_model("tiny.ru")  # русская модель, ~700MB
    result = model.transcribe(audio.file, language="ru")
    text = result["text"].lower()
    
    # 2. Распознавание намерения (простой парсер или LLM)
    intent = parse_intent(text)
    
    # 3. Выполнение действия
    if intent["action"] == "log_work":
        mqtt.publish("work/log", json.dumps({
            "type": intent["work_type"],  # "балансировка"
            "count": intent["count"],     # 4
            "timestamp": time.time()
        }))
        return {"status": "ok", "message": f"Записано: {intent['work_type']} ×{intent['count']}"}
    
    elif intent["action"] == "check_scanner":
        # Запрос статуса сканера
        response = mqtt.request("scanner/status")
        return {"status": "ok", "message": f"Сканер: {response}"}
    
    return {"status": "unknown", "message": "Не понял команду"}
```

**Шаг 3: Простой парсер намерений (без LLM, для начала)**
```python
def parse_intent(text: str) -> dict:
    # Ключевые слова → действия
    if any(kw in text for kw in ["запиши", "записать", "лог"]):
        if "баланс" in text:
            count = extract_number(text) or 1
            return {"action": "log_work", "work_type": "балансировка", "count": count}
        elif "правк" in text:
            return {"action": "log_work", "work_type": "правка", "count": 1}
    
    if any(kw in text for kw in ["сканер", "датчик", "диск"]):
        if "статус" in text or "как" in text:
            return {"action": "check_scanner"}
    
    # Если не распознали — можно отправить в LLM для до-распознавания
    return {"action": "unknown", "raw": text}

def extract_number(text: str) -> Optional[int]:
    # Простой поиск цифр в тексте
    import re
    match = re.search(r'(\d+)', text)
    return int(match.group(1)) if match else None
```

### 📊 Что нужно для MVP (2-3 вечера)
- [ ] Настроить микрофон на Pi400 (USB или GPIO I2S)
- [ ] Установить `whisper` (tiny.ru модель) + FastAPI
- [ ] Написать парсер `parse_intent()` для 3-5 команд
- [ ] Протестировать: сказать «запиши балансировка четыре» → увидеть запись в логе

### ⚠️ Ограничения
- Распознавание русского — качество зависит от модели (tiny = быстро, но ошибки; medium = точнее, но медленнее)
- Шум в цеху — может потребоваться шумоподавление (RNNoise) или кнопка «начать запись»
- Оффлайн-режим — только если запускать Whisper на ESP32-S3 (очень ограниченные возможности)

---

## 🟡 Приоритет 3: Predictive Maintenance (для Runout Scanner)

### 🎯 Какую боль закрывает
«Диск отбалансировали, но через неделю клиент возвращается: снова бьёт. Хотелось бы понимать: это брак диска, износ подшипника станка или ошибка правки».

### 🏗 Архитектура
```
[Runout Scanner]
       │ (сбор данных)
       ▼
[Локальный буфер: 100-200 замеров на диск]
       │
       ├──> Простая статистика (среднее, σ, выбросы)
       │
       └──> (опционально) Отправка на Pi400 для анализа временных рядов
```

### 🔧 Техническая реализация (уровень 1: статистика)

**Сбор данных (`scanner_logger.cpp`):**
```c
struct DiscMeasurement {
    uint32_t timestamp;
    float baseline;      // Среднее расстояние
    float sigma;         // Стандартное отклонение
    float max_deviation; // Максимальное отклонение от baseline
    uint8_t quality;     // 0=good, 1=warn, 2=critical
};

// Кольцевой буфер на 100 измерений
RingBuffer<DiscMeasurement, 100> history;

void afterCalibration() {
    DiscMeasurement m;
    m.timestamp = millis();
    m.baseline = calculateBaseline();
    m.sigma = calculateSigma();
    m.max_deviation = findMaxDeviation();
    m.quality = (m.max_deviation > 4*m.sigma) ? 2 : 
                (m.max_deviation > 2*m.sigma) ? 1 : 0;
    
    history.push(m);
    
    // Простой анализ тренда
    if (history.size() >= 10) {
        float recent_avg = averageLastN(5);
        float older_avg = averageRange(5, 10);
        
        if (recent_avg > older_avg * 1.2) {
            alert("⚠️ Тренд: биение растёт — проверь подшипник станка");
        }
    }
}
```

**Визуализация тренда (на OLED или через Wi-Fi):**
```
Биение (мм)
   ^
   |     *     *
   |   *   * *   *
   |  *           *
   | *             *
   +----------------> Время
     1  2  3  4  5  (диски)
     
[Стрелка: ↑ растёт / ↓ падает / → стабильно]
```

### 🚀 Следующий уровень (машинное обучение)
- **Модель**: Простой LSTM или Random Forest на Pi400
- **Признаки**: σ, макс. отклонение, асимметрия распределения, частота выбросов
- **Цель**: Классификация «брак диска» / «износ станка» / «ошибка правки»
- **Датасет**: Нужно собрать ~100-200 размеченных измерений (это работа на 1-2 месяца)

### 📊 Что нужно для MVP (1 вечер)
- [ ] Добавить структуру `DiscMeasurement` в `[[Runout_Scanner]]`
- [ ] Реализовать кольцевой буфер на 50-100 записей
- [ ] Добавить простой тренд-анализ (сравнение последних 5 с предыдущими 5)
- [ ] Вывод предупреждения на экран / в лог

---

## 🟢 Приоритет 4: Контекстные профили (авто-переключение)

### 🎯 Какую боль закрывает
«Забыл переключить Pocket Engineer из ДОМ в РАБОТА — и теперь не работает логгер, потому что нет нужных настроек».

### 🏗 Архитектура
```
[Источники контекста]
       │
       ├──> BLE-маяки (дома: телефон Сильвиной, дома: пи400)
       ├──> Wi-Fi SSID (дома: "HomeWiFi", работа: "ShinaNet")
       ├──> GPS (если добавить модуль)
       └──> Время (ночью — спящий режим)
       │
       ▼
[Контекстный движок]
       │
       ▼
[Авто-переключение профиля в Pocket Engineer]
```

### 🔧 Техническая реализация

**Сканирование окружения (`context_manager.cpp`):**
```c
struct Context {
    bool isHome;      // Определяется по BLE / Wi-Fi
    bool isWork;
    bool isNight;     // По времени
    uint8_t confidence; // 0-100% уверенность
};

Context detectContext() {
    Context ctx = {false, false, false, 0};
    
    // Проверка Wi-Fi
    String ssid = WiFi.SSID();
    if (ssid == "HomeWiFi") { ctx.isHome = true; ctx.confidence += 40; }
    if (ssid == "ShinaNet") { ctx.isWork = true; ctx.confidence += 40; }
    
    // Проверка BLE-маяков
    auto knownDevices = scanBLE(10); // сканировать 10 сек
    if (knownDevices.contains("SilvinaPhone")) { ctx.isHome = true; ctx.confidence += 30; }
    if (knownDevices.contains("Pi400")) { ctx.isHome = true; ctx.confidence += 30; }
    
    // Время
    if (hour() >= 23 || hour() <= 6) { ctx.isNight = true; ctx.confidence += 20; }
    
    return ctx;
}
```

**Авто-переключение:**
```c
void checkContextChange() {
    static Context lastCtx;
    Context current = detectContext();
    
    if (current.confidence < 50) return; // Не уверены — не переключаем
    
    if (current.isHome && !lastCtx.isHome) {
        switchProfile(HOME_PROFILE);
        display.toast("🏠 Режим: ДОМ");
    }
    else if (current.isWork && !lastCtx.isWork) {
        switchProfile(WORK_PROFILE);
        display.toast("🔧 Режим: РАБОТА");
    }
    
    lastCtx = current;
}

// Вызов каждые 2-5 минут (не чаще, чтобы не жрать батарею)
xTaskCreatePeriodic(checkContextChange, 300000); // 5 мин
```

### 📊 Что нужно для MVP (1 вечер)
- [ ] Добавить функцию `detectContext()` с проверкой только Wi-Fi
- [ ] Реализовать авто-переключение между 2 профилями (ДОМ/РАБОТА)
- [ ] Добавить уведомление на экран при смене
- [ ] Протестировать: подключиться к разной сети → увидеть смену профиля

---

## 🟢 Приоритет 5: Централизованный лог (на Pi400)

### 🎯 Какую боль закрывает
«Данные с сканера, часов и карманного хаба разбросаны. Хочется видеть всё в одном месте: что случилось, когда, с каким устройством».

### 🏗 Архитектура
```
[Все устройства]
       │ (MQTT / HTTP)
       ▼
[Pi400: Mosquitto + FastAPI + SQLite]
       │
       ├──> Веб-интерфейс (простой дашборд)
       ├──> Экспорт в CSV / JSON
       └──> Алерты в Telegram при критических событиях
```

### 🔧 Техническая реализация (минимальная)

**Структура лога (`log_entry`):**
```json
{
  "device": "runout_scanner_01",
  "timestamp": 1717123456,
  "event": "calibration_complete",
  "data": {
    "baseline": 125.3,
    "sigma": 0.8,
    "quality": "good"
  },
  "severity": 1
}
```

**Приём на Pi400 (`logger_api.py`):**
```python
from fastapi import FastAPI
import sqlite3, json, time

app = FastAPI()

def init_db():
    conn = sqlite3.connect('ecosystem.db')
    c = conn.cursor()
    c.execute('''
        CREATE TABLE IF NOT EXISTS logs (
            id INTEGER PRIMARY KEY,
            device TEXT,
            timestamp INTEGER,
            event TEXT,
            data TEXT,
            severity INTEGER
        )
    ''')
    conn.commit()
    return conn

@app.post("/log")
async def receive_log(entry: dict):
    conn = init_db()
    c = conn.cursor()
    c.execute('''
        INSERT INTO logs (device, timestamp, event, data, severity)
        VALUES (?, ?, ?, ?, ?)
    ''', (
        entry['device'],
        entry['timestamp'],
        entry['event'],
        json.dumps(entry.get('data', {})),
        entry.get('severity', 1)
    ))
    conn.commit()
    conn.close()
    return {"status": "ok"}

@app.get("/logs/{device}")
async def get_logs(device: str, limit: int = 50):
    conn = init_db()
    c = conn.cursor()
    c.execute('''
        SELECT timestamp, event, data, severity 
        FROM logs 
        WHERE device = ? 
        ORDER BY timestamp DESC 
        LIMIT ?
    ''', (device, limit))
    rows = c.fetchall()
    conn.close()
    return [{"timestamp": r[0], "event": r[1], "data": json.loads(r[2]), "severity": r[3]} for r in rows]
```

**Отправка с ESP32 (`logger_client.cpp`):**
```c
void sendLog(const char* device, const char* event, JsonObject data, uint8_t severity) {
    StaticJsonDocument<512> doc;
    doc["device"] = device;
    doc["timestamp"] = time(nullptr);
    doc["event"] = event;
    doc["severity"] = severity;
    
    JsonObject dataObj = doc.createNestedObject("data");
    for (auto kv : data) {
        dataObj[kv.key] = kv.value;
    }
    
    String payload;
    serializeJson(doc, payload);
    
    http.begin("http://192.168.1.100:8000/log");
    http.addHeader("Content-Type", "application/json");
    http.POST(payload);
}
```

### 📊 Что нужно для MVP (2 вечера)
- [ ] Настроить FastAPI + SQLite на Pi400 (1 вечер)
- [ ] Реализовать `sendLog()` на ESP32 (полвечера)
- [ ] Протестировать: отправить тестовый лог → увидеть в базе / через GET-запрос

---

## 🔵 Остальные идеи (кратко)

| Идея | Суть | Минимальный шаг |
|------|------|----------------|
| **Self-healing configs** | Устройство видит битый конфиг → тянет бэкап с Pi400 | Добавить проверку CRC конфига + HTTP-запрос к `http://pi400/config/{device}` |
| **Координированный сон** | Устройства договариваются о расписании сна через MQTT | Добавить топик `ecosystem/sleep_schedule` и простую логику «спим с 00:00 до 06:00» |
| **Текст → команда** | Пишешь «проверь сканер» в консоль → парсер → действие | Написать простой CLI на Python, который парсит строку и публикует в MQTT |
| **Визуальная отладка** | Камера на Pi400 снимает LED устройства → корреляция с логами | Начать с ручного: «включил отладку → моргнуло 3 раза → записал в лог» |

---

## 🎯 Алгоритм выбора: что делать следующим

```
1. Открой этот файл
2. Пройдись по приоритетам (🔴 → 🟡 → 🟢)
3. Для каждой идеи спроси:
   • Какую БОЛЬ это закрывает МНЕ прямо сейчас?
   • Какой НАВЫК я прокачаю?
   • Можно ли сделать МИКРО-версию за 1-2 вечера?
4. Выбери одну. Запиши микро-шаг в [[Dev_log]].
5. Сделай. Зафиксируй результат.
6. Вернись сюда — выбери следующую.
```

> [!tip] Главное правило
> Не «сделать всё». А «сделать одно — но до конца».
> Каждая закрытая идея — это +1 кирпич в твою экосистему.

Версия: 1.0
Статус: 🟡 Backlog (выбери одну → начни)