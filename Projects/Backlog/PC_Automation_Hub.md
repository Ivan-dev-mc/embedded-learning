project: PC Automation Hub (Focus & Backup)
status: 🟡 Backlog (Объединённая идея №3 и №4)
hardware: [ESP32 WROOM, 2x Тактовые кнопки 30мм, RGB LED (WS2812B), Пьезо-пищалка, Корпус]
budget: ~600₽
stack: [ESP-IDF/Arduino, Wi-Fi/HTTP, Python, Flask, PyAutoGUI, Git, subprocess]
tags: [project, pc, automation, focus, backup, productivity, hub]
created: 2026-05-31
updated: 2026-06-01
links: [[Embedded_Plan]], [[Pi400_workstation]], [[Insights]], [[AI_Integration_Strategy]], [[Auto_Backup_Trigger]], [[Focus_Controller]]

# 🎛️ PC Automation Hub (Focus & Backup)

## 📋 Назначение
Единая физическая панель управления рабочим столом.
Две независимые кнопки, которые запускают сложные сценарии на ПК:
1.  **FOCUS:** Подготовка рабочего места (IDE, музыка, блокировка отвлечений).
2.  **BACKUP:** Мгновенное сохранение кода и конфигов (Git commit/push, архивация).

## 🎯 Боль
- **Контекст:** Уходит 5-10 минут на "раскачку" перед кодингом.
- **Страх:** Забываешь сделать `git push` или бэкап конфигов.
- **Рутина:** Лень запускать скрипты через терминал.
- **Железо:** Занимать два USB-порта под два устройства — расточительство.

##  Решение
Одна плата ESP32 + корпус.
**Кнопка 1 (Focus):** Жёлтая индикация → ПК закрывает лишнее, открывает VS Code/Obsidian, включает Lo-Fi.
**Кнопка 2 (Backup):** Синяя индикация → ПК коммитит код, пушит, архивирует конфиги.
**Обратная связь:** LED + звук подтверждают успех или ошибку.

---

## 🏗 Архитектура устройства

### Схема компонентов
| Компонент | Функция | Подключение (ESP32) |
|-----------|---------|---------------------|
| ESP32 WROOM | Ядро, Wi-Fi, HTTP клиент | USB 5V |
| Кнопка 1 (Focus) | Триггер сценария фокуса | GPIO 4 (INPUT_PULLUP) |
| Кнопка 2 (Backup) | Триггер сценария бэкапа | GPIO 5 (INPUT_PULLUP) |
| RGB LED (WS2812B) | Статус: Жёлтый/Синий/Зелёный/Красный | GPIO 12 (FastLED) |
| Пьезо-пищалка | Звуковой сигнал | GPIO 18 |

### Логика взаимодействия
```
[Кнопка 1] ──
             ├─► [ESP32] ──(HTTP POST /focus)──► [ПК: Flask Server]
[Кнопка 2] ──┘                                        │
                                                      ├── Focus Logic (PyAutoGUI, Apps)
                                                      └── Backup Logic (Git, Tar)

️ **Важно:** На ПК работает **один** Python-скрипт с двумя роутами. Это экономит ресурсы и упрощает автозапуск.

---

## 💻 Программное обеспечение

### 1. Код ESP32 (Управление двумя кнопками)
```cpp
#include <WiFi.h>
#include <HTTPClient.h>
#include <FastLED.h>

#define NUM_LEDS 1
#define DATA_PIN 12
CRGB leds[NUM_LEDS];

const char* ssid = "YOUR_WIFI";
const char* password = "YOUR_PASS";
const char* pc_ip = "http://192.168.1.100:5000"; // IP твоего ПК

void setup() {
  Serial.begin(115200);
  FastLED.addLeds<WS2812, DATA_PIN, GRB>(leds, NUM_LEDS);
  pinMode(4, INPUT_PULLUP); // Focus Btn
  pinMode(5, INPUT_PULLUP); // Backup Btn
  pinMode(18, OUTPUT);      // Buzzer
  
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) delay(500);
}

void loop() {
  // 1. Проверка кнопки FOCUS
  if (digitalRead(4) == LOW) {
    trigger_action("focus", CRGB::Yellow, 1000, 100); // Жёлтый, 1 писк
    delay(500); // Антидребезг
  }
  
  // 2. Проверка кнопки BACKUP
  if (digitalRead(5) == LOW) {
    trigger_action("backup", CRGB::Blue, 200, 300); // Синий, 2 писка
    delay(500); // Антидребезг
  }
}

void trigger_action(String route, CRGB color, int freq, int duration) {
  FastLED.showColor(color); // Индикация старта
  
  HTTPClient http;
  http.begin(String(pc_ip) + "/" + route);
  int code = http.POST("");
  http.end();
  
  if (code == 200) {
    FastLED.showColor(CRGB::Green); // Успех
    tone(18, freq, duration);
  } else {
    FastLED.showColor(CRGB::Red);   // Ошибка
    tone(18, 150, 500);             // Длинный писк ошибки
  }
  
  delay(1000);
  FastLED.showColor(CRGB::Black);   // Выкл
}
```

### 2. Код ПК (Flask Server + Логика)
```python
from flask import Flask, jsonify
import subprocess, pyautogui, time, os, datetime

app = Flask(__name__)

# === КОНФИГУРАЦИЯ ===
PROJECT_DIR = "/home/vanya/projects" # Путь к твоим проектам
BACKUP_DIR  = "/home/vanya/backups"
CONFIGS     = ["/home/vanya/.ssh", "/home/vanya/.obsidian"]

def run(cmd):
    return subprocess.run(cmd, shell=True, capture_output=True, text=True)

# === РОУТ 1: FOCUS ===
@app.route('/focus', methods=['POST'])
def start_focus():
    try:
        # 1. Закрыть лишнее (привет Chrome и Discord)
        run("pkill -f chrome")
        run("pkill -f discord")
        time.sleep(0.5)
        
        # 2. Открыть рабочее
        subprocess.Popen(['code', PROJECT_DIR])       # VS Code
        subprocess.Popen(['obsidian'])                # Obsidian
        subprocess.Popen(['gnome-terminal'])          # Терминал
        
        # 3. Музыка (Ссылка на плейлист)
        subprocess.Popen(['xdg-open', 'https://open.spotify.com/playlist/37i9dQZF1DWWQRwui0ExPn'])
        
        return jsonify({"status": "focus_active"}), 200
    except Exception as e:
        return jsonify({"error": str(e)}), 500

# === РОУТ 2: BACKUP ===
@app.route('/backup', methods=['POST'])
def trigger_backup():
    try:
        ts = datetime.datetime.now().strftime("%Y%m%d_%H%M")
        
        # 1. Git (только если есть изменения)
        os.chdir(PROJECT_DIR)
        run("git add .")
        status = run("git status --porcelain")
        
        if status.stdout.strip():
            run(f'git commit -m "auto-backup {ts}"')
            run("git push origin main")
            
        # 2. Архивация конфигов
        os.makedirs(BACKUP_DIR, exist_ok=True)
        tar_path = os.path.join(BACKUP_DIR, f"configs_{ts}.tar.gz")
        # Используем tar с пробелами между путями
        tar_cmd = f"tar -czf {tar_path} {' '.join(CONFIGS)}"
        run(tar_cmd)
        
        # 3. Очистка старых ( >7 дней)
        run(f"find {BACKUP_DIR} -name 'configs_*.tar.gz' -mtime +7 -delete")
        
        return jsonify({"status": "backup_done"}), 200
    except Exception as e:
        return jsonify({"error": str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

## 🐛 Отладка и диагностика

| Инструмент | Назначение |
|------------|------------|
| `Serial Monitor` | Лог Wi-Fi, статус отправки HTTP, нажатия кнопок |
| `curl -X POST http://IP:5000/focus` | Тест роута Focus без кнопки |
| `curl -X POST http://IP:5000/backup` | Тест роута Backup без кнопки |
| Консоль Python | Вывод ошибок `subprocess`, прав доступа, путей |
| `git log --oneline` | Проверка, что бэкапы не дублируются |

---

## ⚠️ Риски и решения

| Риск | Вероятность | Решение |
|------|-------------|---------|
| **Случайное нажатие обеих кнопок** | 🟡 Средняя | Программный таймаут 500мс. В коде ESP32 кнопки проверяются последовательно |
| **Сервер Flask не запущен** | 🔴 Высокая | ESP32 ловит ошибку соединения → мигает красным. Пользователь знает, что проблема в ПК |
| **Git конфликтует (нечего коммитить)** | 🟢 Низкая | В коде Python проверка `git status --porcelain`. Если чисто — просто выход с кодом 200 |
| **PyAutoGUI сбивается** | 🟡 Средняя | Не использовать `pyautogui` для критичных действий. Лучше запускать процессы напрямую (`subprocess.Popen`) |

---

## ✅ Чек-лист перед сборкой

### Электрика
- [ ] ESP32 получает стабильное 5V (USB)
- [ ] Кнопки подключены к GPIO 4 и 5 (с Pull-Up, т.е. просто к земле)
- [ ] LED WS2812B подключен через резистор (опционально) и имеет конденсатор 100µF по питанию
- [ ] Пищалка подключена к GPIO 18

### Программирование (ESP32)
- [ ] Wi-Fi подключается автоматически при старте
- [ ] LED загорается Жёлтым при Focus и Синим при Backup
- [ ] При ошибке сервера LED становится Красным

### Программирование (ПК)
- [ ] Скрипт `flask_server.py` запускается от обычного пользователя
- [ ] Пути (`PROJECT_DIR`, `CONFIGS`) прописаны верно
- [ ] `git` настроен так, что не требует пароля (SSH keys)
- [ ] Сервер добавлен в автозагрузку (Task Scheduler / systemd)

---

## 📅 План выполнения

### Этап 1: Прототип на столе (1 вечер)
- [ ] Собрать схему: ESP32 + 2 кнопки + LED + Пищалка
- [ ] Прошить ESP32: при нажатии кнопки 1 → Жёлтый, кнопки 2 → Синий
- [ ] Настроить Flask на ПК: проверить ответы 200 OK через `curl`

### Этап 2: Логика ПК (1 вечер)
- [ ] Реализовать роут `/focus`: открытие VS Code и музыки
- [ ] Реализовать роут `/backup`: git commit/push + архивация
- [ ] Протестировать: нажал → произошло

### Этап 3: Интеграция (1 вечер)
- [ ] Связать железо и софт
- [ ] Настроить цвета LED (Жёлтый=Focus, Синий=Backup)
- [ ] Добавить звуковые сигналы (1 писк=Focus, 2 писка=Backup)

### Этап 4: Корпус и Финал (1 вечер)
- [ ] Установить в корпус (кнопки с гравировкой FOCUS/BACKUP)
- [ ] Коммит в Git, README

---

##  Связи и ресурсы

### Внутренние ссылки
- [[Embedded_Plan]] → Глава 3.1 (Wi-Fi, HTTP клиент)
- [[Pi400_workstation]] → Серверная часть (Flask) может жить на Pi400
- [[Insights]] → Фиксация проблем с PyAutoGUI и правами доступа
- [[AI_Integration_Strategy]] → В будущем кнопки могут триггерить голосовой ассистент
- [[Auto_Backup_Trigger]] → Исходный вариант кнопки бэкапа
- [[Focus_Controller]] → Исходный вариант кнопки фокуса 

### Внешние ресурсы
- [Flask Documentation](https://flask.palletsprojects.com/)
- [PyAutoGUI Documentation](https://pyautogui.readthedocs.io/)
- [FastLED Library](https://github.com/FastLED/FastLED)
- [GitPython Documentation](https://gitpython.readthedocs.io/)

### Закупка компонентов
| Компонент | Цена | Где |
|-----------|------|-----|
| ESP32 WROOM | ~400₽ | AliExpress/Ozon |
| Тактовые кнопки 30мм | 2x ~50₽ | Набор |
| WS2812B LED | ~30₽ | Набор |
| Пищалка | ~20₽ | Набор |
| Корус | ~150₽ | 3D печать |
| **Итого** | **~650₽** | |

---

## 💡 Инсайты

> [!tip] Про единство интерфейса #insight
> «Две функции на одной плате — это не экономия места. Это создание единой точки входа. "Пульт управления жизнью инженера". Это снижает когнитивную нагрузку: ты знаешь, где искать кнопку "Начать" и кнопку "Сохранить"».

> [!example] Почему один сервер? #insight
> «Запускать два Flask-приложения на разных портах — это оверхед. Один процесс, два роута. Это архитектурно чище и легче в обслуживании (один файл автозагрузки)».

> [!warning] Про идемпотентность Backup #insight
> «Кнопка бэкапа должна быть безопасной. Если ты нажмёшь её 5 раз подряд, сервер не должен создать 5 коммитов с одним и тем же сообщением или 5 архивов. Код должен проверять `git status` перед действием».

---

## 📊 Метрики успеха
| Критерий | Цель | Факт |
|----------|------|------|
| Время реакции (кнопка → старт) | < 500 мс | 🟡 TBD |
| Время выполнения Focus | < 15 сек | 🟡 TBD |
| Время выполнения Backup | < 30 сек (зависит от гигабитности) | 🟡 TBD |
| Стабильность | 0 ложных срабатываний в день | 🟡 TBD |

Версия: 1.0 (Merged)
Статус: 🟡 Backlog