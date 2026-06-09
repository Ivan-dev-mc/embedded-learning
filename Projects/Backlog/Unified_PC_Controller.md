project: Unified PC Controller (Macro + Automation Hub)
status: 🟡 Backlog (Merged Concept)
hardware: [ESP32-S3, 6x Buttons, KY-040 Encoder, OLED 0.96" I2C, WS2812B RGB, Buzzer, USB-C]
budget: ~1600₽
stack: [ESP-IDF/Arduino, TinyUSB, FreeRTOS, Wi-Fi/HTTP, Python, Flask/FastAPI, HID]
tags: [project, pc, automation, hid, hub, unified, esp32-s3]
created: 2026-06-01
updated: 2026-06-01
links: [[Embedded_Plan]], [[Pi400_workstation]], [[PC_Macro_Pad]], [[PC_Automation_Hub]], [[Offline_Voice_Assistant]], [[Insights]]
---

# 🎛️ Unified PC Controller (Macro + Automation Hub)

##  Назначение
Единое устройство на базе ESP32-S3, работающее в двух режимах:
1. **HID (Macro Pad):** Эмуляция клавиатуры/мыши для быстрых хоткеев.
2. **AUTO (Automation Hub):** Отправка HTTP-триггеров на ПК для запуска сложных сценариев (Focus, Backup, Voice).

Режим переключается Long-Press на энкодере. OLED показывает текущий профиль и статус.

## 🎯 Боль
- Дублирование устройств: отдельно стрим-дек, отдельно кнопка фокуса/бэкапа.
- Расточительство: 2 USB-порта, 2 корпуса, 2 прошивки.
- Когнитивная нагрузка: искать разные кнопки под разные задачи.
- Отсутствие обратной связи: не понятно, выполнился ли скрипт или просто отправился хоткей.

## 💡 Решение
Одна плата ESP32-S3 + единый корпус.
**Режим MACRO:** Кнопки шлют HID-коды (Ctrl+S, F5...). Энкодер → громкость/скролл.
**Режим AUTO:** Кнопки шлют HTTP POST на ПК (`/focus`, `/backup`). ПК выполняет сценарий, ESP32 показывает статус на OLED + LED/звук.
**Переключение:** Long-Press энкодера → смена режима → сохранение в NVS.

---

## 🏗 Архитектура устройства

### Компоненты
| Компонент | Функция | Подключение |
|-----------|---------|-------------|
| ESP32-S3 | Native USB HID + Wi-Fi HTTP клиент | USB-C (питание + данные) |
| 6x Тактовые кнопки | Триггеры действий | GPIO 1–6, internal pull-up |
| Энкодер KY-040 | Громкость / Скролл / Переключение режимов | GPIO 7 (CLK), GPIO 8 (DT), GPIO 9 (SW) |
| OLED 0.96" I2C | Отображение режима, профиля, статуса | GPIO 21 (SDA), GPIO 22 (SCL) |
| WS2812B RGB | Статус выполнения (🟡 В процессе / 🟢 Успех / 🔴 Ошибка) | GPIO 12 |
| Пьезо-пищалка | Звуковое подтверждение | GPIO 18 |

### Логика взаимодействия
```
[Кнопки/Энкодер] 
      │
      ▼
[ESP32-S3: State Machine]
      ├──> MODE_MACRO ──(USB HID)──► [ОС: Стандартный драйвер клавиатуры]
      ──> MODE_AUTO  ──(Wi-Fi HTTP)──► [ПК: Flask Server]
                                              │
                                              ├── /focus  → PyAutoGUI + Apps
                                              ├── /backup → Git + Tar
                                              └── /voice  → Запись + Whisper
```
⚠️ **Важно:** ESP32-S3 поддерживает Native USB и Wi-Fi одновременно. Конфликтов нет, если разделять задачи в FreeRTOS.

---

## 💻 Программное обеспечение

### 1. Логика ESP32 (State Machine + Routing)
```cpp
#include <WiFi.h>
#include <HTTPClient.h>
#include <TinyUSB.h>
#include <Adafruit_TinyUSB.h>
#include <FastLED.h>
#include <nvs_flash.h>

#define NUM_LEDS 1
#define DATA_PIN 12
CRGB leds[NUM_LEDS];

enum Mode { MACRO, AUTO };
Mode currentMode = MACRO;
const char* ssid = "YOUR_WIFI";
const char* password = "YOUR_PASS";
const char* pc_ip = "http://192.168.1.100:5000";

Adafruit_USBD_HID usb_hid;
uint8_t keymap[6] = {HID_KEY_A, HID_KEY_B, HID_KEY_C, HID_KEY_D, HID_KEY_E, HID_KEY_F}; // Заменить на реальные хоткеи
String autoRoutes[6] = {"/focus", "/backup", "/voice", "/mute", "/screenshot", "/unknown"};

void setup() {
  Serial.begin(115200);
  nvs_flash_init();
  FastLED.addLeds<WS2812, DATA_PIN, GRB>(leds, NUM_LEDS);
  
  pinMode(4, INPUT_PULLUP); pinMode(5, INPUT_PULLUP); pinMode(6, INPUT_PULLUP);
  pinMode(7, INPUT_PULLUP); pinMode(8, INPUT_PULLUP); pinMode(9, INPUT_PULLUP);
  pinMode(12, OUTPUT); pinMode(18, OUTPUT);
  
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) delay(500);
  
  usb_hid.begin();
  loadModeFromNVS();
  updateOLED();
}

void loop() {
  if (encoderLongPress()) { toggleMode(); updateOLED(); }
  
  for (int i = 0; i < 6; i++) {
    if (digitalRead(i+4) == LOW) {
      if (currentMode == MACRO) sendHID(keymap[i]);
      else triggerHTTP(autoRoutes[i]);
      delay(300); // Антидребезг
    }
  }
}

void triggerHTTP(String route) {
  FastLED.showColor(CRGB::Yellow);
  HTTPClient http;
  http.begin(String(pc_ip) + route);
  int code = http.POST("");
  http.end();
  
  if (code == 200) {
    FastLED.showColor(CRGB::Green); tone(18, 1000, 100);
  } else {
    FastLED.showColor(CRGB::Red); tone(18, 200, 300);
  }
  delay(1000); FastLED.showColor(CRGB::Black);
}
```

### 2. Сервер на ПК (Flask Endpoints)
```python
from flask import Flask, jsonify
import subprocess, os, datetime

app = Flask(__name__)
PROJECT_DIR = "/home/vanya/projects"
BACKUP_DIR = "/home/vanya/backups"

def run(cmd):
    return subprocess.run(cmd, shell=True, capture_output=True, text=True)

@app.route('/focus', methods=['POST'])
def start_focus():
    run("pkill -f chrome"); time.sleep(0.5)
    subprocess.Popen(['code', PROJECT_DIR])
    subprocess.Popen(['obsidian'])
    return jsonify({"status": "ok"}), 200

@app.route('/backup', methods=['POST'])
def trigger_backup():
    ts = datetime.datetime.now().strftime("%Y%m%d_%H%M")
    os.chdir(PROJECT_DIR)
    run("git add .")
    if run("git status --porcelain").stdout.strip():
        run(f'git commit -m "auto-backup {ts}" && git push')
    run(f"tar -czf {BACKUP_DIR}/configs_{ts}.tar.gz ~/.ssh ~/.obsidian")
    run(f"find {BACKUP_DIR} -name 'configs_*.tar.gz' -mtime +7 -delete")
    return jsonify({"status": "ok"}), 200

@app.route('/voice', methods=['POST'])
def start_voice_rec():
    # Триггер для запуска скрипта записи + whisper
    subprocess.Popen(['python3', '/home/vanya/scripts/voice_rec.py'])
    return jsonify({"status": "recording_started"}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

## 🔄 Режимы и маппинг

| Кнопка | Режим MACRO (HID) | Режим AUTO (HTTP) | OLED Иконка |
|--------|------------------|------------------|-------------|
| **1** | `Ctrl+Shift+P` (Command Palette) | `/focus` (Подготовка места) | 💻 / 🎯 |
| **2** | `Ctrl+S` (Save) | `/backup` (Git + Архив) | 💾 / 🔒 |
| **3** | `F5` (Refresh/Run) | `/voice` (Голосовой ввод) | 🔄 / ️ |
| **4** | `Alt+Tab` (Switch) | `/mute` (Выкл. звук/уведомления) |  / 🔇 |
| **5** | `Ctrl+C` (Copy) | `/screenshot` (Снимок экрана) |  / 📸 |
| **6** | `Ctrl+V` (Paste) | `/unknown` (Кастом) | 📄 / ️ |
| **Энкодер** | Громкость / Скролл | Long-Press → Смена режима | 🔊 / 🔄 |

---

##  Отладка и диагностика

| Инструмент | Назначение |
|------------|------------|
| `idf.py monitor` | Лог переключений режимов, Wi-Fi статус, HTTP коды |
| `lsusb -v` (Linux) / USB Tree Viewer (Win) | Проверка HID дескрипторов и репортов |
| `curl -X POST http://IP:5000/focus` | Тест HTTP роутов без нажатия кнопок |
| Key Tester (web) | Визуальная проверка HID-кодов |
| Консоль Python | Логи выполнения сценариев, ошибки `subprocess` |

---

## ⚠️ Риски и решения

| Риск | Вероятность | Решение |
|------|-------------|---------|
| **Конфликт USB и Wi-Fi** | 🟢 Низкая | ESP32-S3 аппаратно разделяет контроллеры. В коде — разделение задач по FreeRTOS таскам |
| **Случайная смена режима** | 🟡 Средняя | Long-Press > 1.5 сек + подтверждение на OLED ("MODE: AUTO?") |
| **HTTP недоступен при старте** | 🟡 Средняя | ESP32 кэширует последний режим. При нажатии в AUTO → мигает красным, если нет ответа |
| **Дребезг ломает HID** | 🔴 Высокая | Аппаратный RC-фильтр + программный таймаут 50мс в таске INPUT |
| **NVS слетает при прошивке** | 🟢 Низкая | Выделять отдельную партицию под конфиги, не стирать при `idf.py flash` |

---

## ✅ Чек-лист перед сборкой

### Электрика
- [ ] ESP32-S3 DevKit проверен, Native USB определяется как HID
- [ ] Кнопки 1-6 подключены к GPIO с pull-up
- [ ] Энкодер CLK/DT/SW подключены к GPIO 7-9
- [ ] OLED I2C работает на 3.3В, адрес 0x3C/0x3D определён
- [ ] WS2812B и пищалка не перегружают пины, GND общая

### Программирование (ESP32)
- [ ] Реализован State Machine: MACRO ↔ AUTO
- [ ] HID репорт формируется корректно (modifier + keycode)
- [ ] HTTP клиент отправляет POST, обрабатывает 200/500
- [ ] Переключение режимов сохраняется в NVS
- [ ] OLED обновляется без мерцания, показывает режим и профиль

### Программирование (ПК)
- [ ] Flask сервер запускается автоматически, слушает `0.0.0.0:5000`
- [ ] Роуты `/focus`, `/backup`, `/voice` работают изолированно
- [ ] `git` настроен без пароля, пути к конфигам верны
- [ ] Логирование ошибок в `automation.log`

---

## 📅 План выполнения

### Этап 1: Базовый прототип (2-3 вечера)
- [ ] Собрать схему: ESP32-S3 + 2 кнопки + OLED + энкодер
- [ ] Прошить HID: кнопка 1 → `Ctrl+S`, энкодер → громкость
- [ ] Проверить определение как клавиатуры в ОС

### Этап 2: Добавление AUTO режима (2 вечера)
- [ ] Написать Flask сервер на ПК с роутами `/focus` и `/backup`
- [ ] Добавить в ESP32 логику переключения режимов (Long-Press)
- [ ] Реализовать HTTP POST при нажатии кнопок в режиме AUTO
- [ ] Настроить обратную связь: LED + OLED статус

### Этап 3: Полная интеграция (2 вечера)
- [ ] Подключить все 6 кнопок, настроить маппинг
- [ ] Добавить NVS сохранение режима и профиля
- [ ] Оптимизировать FreeRTOS таски (HID / UI / INPUT / NETWORK)
- [ ] Стресс-тест: 500 переключений, проверка на зависание

### Этап 4: Корпус и финализация (1-2 вечера)
- [ ] Сборка в корпус, гравировка кнопок (MACRO / AUTO)
- [ ] Финальная прошивка, калибровка энкодера
- [ ] README с инструкцией по кастомизации маппинга
- [ ] Коммит в Git

---

## 🔗 Связи и ресурсы

### Внутренние ссылки
- [[Embedded_Plan]] → Глава 1.4 (Прерывания), Глава 2.2 (FreeRTOS), Глава 3.1 (Wi-Fi/HTTP)
- [[Pi400_workstation]] → Серверная часть может быть перенесена на Pi400
- [[PC_Macro_Pad]] → Исходная спецификация HID-режима
- [[PC_Automation_Hub]] → Исходная спецификация HTTP-триггеров
- [[Offline_Voice_Assistant]] → Интеграция кнопки 3 для запуска записи
- [[Insights]] → Фиксация проблем с дескрипторами и NVS

### Внешние ресурсы
- [ESP-IDF USB HID Example](https://github.com/espressif/esp-idf/tree/master/examples/peripherals/usb/device/hid)
- [TinyUSB Documentation](https://docs.tinyusb.org/en/latest/)
- [Flask Quickstart](https://flask.palletsprojects.com/)
- [FastLED Library](https://github.com/FastLED/FastLED)
- [NVS Documentation (ESP-IDF)](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/storage/nvs_flash.html)

### Закупка компонентов
| Компонент | Кол-во | Цена | Где |
|-----------|--------|------|-----|
| ESP32-S3 DevKit (Native USB) | 1 | ~700₽ | AliExpress/Ozon |
| Тактовые кнопки 6x6mm | 6 | ~50₽ | Набор |
| Энкодер KY-040 | 1 | ~100₽ | AliExpress |
| OLED 0.96" I2C | 1 | ~300₽ | AliExpress/ЧипДип |
| WS2812B LED + Пищалка | 1 | ~50₽ | Набор |
| USB-C кабель (Data) | 1 | ~150₽ | Любой |
| Корпус (3D печать) | 1 | ~200₽ | Своё / сервис |
| **Итого** | | **~1550₽** | |

---

## 💡 Инсайты

> [!tip] Про двойную природу устройства #insight
> «HID работает на уровне драйверов ОС. HTTP работает на уровне приложений. Объединение их в одном контроллере не конфликтует, потому что это разные стеки. ESP32-S3 просто маршрутизирует нажатие в нужный протокол в зависимости от флага режима».

> [!example] Почему State Machine, а не отдельные прошивки? #insight
> «Перепрошивать устройство при смене задачи — антипаттерн. Флаговая машина состояний + NVS позволяет переключать контекст за 200мс без потери данных и без подключения к ПК».

> [!success] Про обратную связь в AUTO режиме #insight
> «В MACRO режиме ОС сама подтверждает действие (символ появился). В AUTO режиме ПК выполняет фоновый скрипт. Без LED/OLED пользователь не знает, выполнен ли бэкап или сеть оборвалась. Индикация закрывает петлю доверия».

---

## 📊 Метрики успеха

| Критерий | Цель | Факт |
|----------|------|------|
| Время переключения режима | < 200 мс | 🟡 TBD |
| Задержка HID (нажатие → символ) | < 15 мс | 🟡 TBD |
| Время отклика HTTP (нажатие → старт скрипта) | < 500 мс | 🟡 TBD |
| Стабильность NVS | 0 потерь настроек при ребуте | 🟡 TBD |
| Успешность AUTO сценариев | > 95% | 🟡 TBD |

---

## 🎯 MVP (Минимально жизнеспособный продукт)

**Срок:** 3-4 вечера  
**Функции:**
1. Режим MACRO: 2 кнопки шлют `Ctrl+S` и `Ctrl+Shift+P`, энкодер → громкость
2. Режим AUTO: те же кнопки шлют `/focus` и `/backup` на ПК
3. Переключение Long-Press на энкодере, индикация на OLED
4. Обратная связь: LED жёлтый (старт) → зелёный/красный (результат)

**Чек-лист MVP:**
- [ ] ESP32-S3 определяется как HID при подключении USB
- [ ] Нажатие кнопки 1 в MACRO → сохраняет файл
- [ ] Нажатие кнопки 1 в AUTO → Flask принимает POST, LED мигает зелёным
- [ ] Long-Press энкодера меняет надпись на OLED с "MACRO" на "AUTO"
- [ ] Состояние сохраняется после отключения питания (NVS)

**После MVP:** Добавлять остальные 4 кнопки, сложные сценарии, голосовой триггер, конфигуратор.

---
Версия: 1.0 (Unified)
Статус: 🟡 Backlog (Ждёт сборки прототипа и старта Этапа 1)