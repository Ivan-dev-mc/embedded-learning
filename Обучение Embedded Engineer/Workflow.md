## 🔄 Твой рабочий процесс (по шагам)

### **1. VS Code — Пишешь код**

**Что делаешь:**
- Создаёшь PlatformIO проект (`New Project` или через терминал)
- Пишешь код в `src/main.cpp`
- Редактируешь `platformio.ini` (добавляешь библиотеки, меняешь скорость порта)
- Собираешь проект (`Ctrl+Shift+B`)
- Загружаешь на Arduino/ESP32 (`Ctrl+Alt+U`)
- Открываешь Serial Monitor (`Ctrl+Alt+S`) для отладки

**Пример:**
```cpp
// В VS Code пишешь:
#include <Arduino.h>

void setup() {
  pinMode(13, OUTPUT);
  Serial.begin(9600);
}

void loop() {
  digitalWrite(13, HIGH);
  delay(1000);
  digitalWrite(13, LOW);
  delay(1000);
}
```

---

### **2. Obsidian — Ведёшь базу знаний**

**Что делаешь:**

#### **A. Dev-log (ежедневные записи)**
Создаёшь заметку в папке `dev-log/` с названием типа `2026-06-09-blink.md`:
09.06.2026 — Blink проект
**Задача:**
Зажечь встроенный LED на Arduino UNO
**Что сделал:**
Создал PlatformIO проект, написал код мигания
**Проблема #1:**
**Симптом:** LED не мигает, горит постоянно
**Причина:** Забыл `delay()` в loop(), LED переключается слишком быстро
**Решение:** Добавил delay(1000) после каждого digitalWrite
**Вывод:** Без delay() человеческий глаз не видит мигания
**Итог:**
LED мигает раз в секунду. Следующий шаг — внешний LED с резистором.
#### **B. Сниппеты (готовые решения)**
Создаёшь заметку в папке `snippets/arduino/` с названием типа `blink-led.md`:
**Blink LED**
**Подключение**

| Пин компонента | Пин МК |
|----------------|--------|
| LED (+)        | 13     |
| LED (-)        | GND    |

**Код**
```cpp
#include <Arduino.h>

#define LED_PIN 13

void setup() {
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  digitalWrite(LED_PIN, HIGH);
  delay(1000);
  digitalWrite(LED_PIN, LOW);
  delay(1000);
}
```
**Как это работает**
`digitalWrite(LED_PIN, HIGH)` подаёт 5V на пин 13. Встроенный резистор уже есть на плате.
**Что ломает**
1. Убрать delay() → LED мигает слишком быстро, не видно
2. Поменять HIGH на LOW → LED горит постоянно выключенным
**Адаптация под ESP32**
На ESP32 встроенный LED на GPIO2. Логика та же, но напряжение 3.3V.
#### **C. Теория (концепции)**
Создаёшь заметку в папке `theory/` с названием типа `ohm-law.md`:
**Закон Ома**
**Формула**
V = I × R
где:
- V — напряжение (Вольты, V)
- I — ток (Амперы, A)
- R — сопротивление (Омы, Ω)
**Пример расчёта резистора для LED**
LED: Vf = 2V, If = 20mA
Питание: 5V
R = (5V - 2V) / 0.02A = 150Ω
Берём ближайший стандартный: 220Ω (с запасом)

---

### **3. PowerShell — Git операции**

**Что делаешь:**
**Для репозитория проектов:**
```powershell
# Переходишь в папку проектов
cd "C:\Users\Manager\Documents\embedded-projects"

# Добавляешь изменения
git add .

# Коммитишь с понятным сообщением
git commit -m "Blink: добавил внешний LED с резистором 220Ω"

# Пушишь на GitHub
git push
```
**Для репозитория знаний (Obsidian vault):**
```powershell
# Переходишь в папку vault
cd "C:\Users\Manager\Documents\Obsidian Vault"

# Добавляешь новые заметки
git add .

# Коммитишь
git commit -m "Dev-log: добавил запись о проблеме с LCD RW pin"

# Пушишь
git push
```

---

## 📋 Типичная рабочая сессия (по шагам)

**1. Подготовка (5 мин):**
```powershell
# В PowerShell
cd "C:\Users\Manager\Documents\embedded-projects"
git pull

cd "C:\Users\Manager\Documents\Obsidian Vault"
git pull
```

**2. Код (2-3 часа):**
- В **VS Code**:
  - Открываешь проект (или создаёшь новый)
  - Пишешь код
  - Собираешь (`Ctrl+Shift+B`)
  - Загружаешь (`Ctrl+Alt+U`)
  - Тестируешь в Serial Monitor

**3. Dev-log (10-15 мин):**
- В **Obsidian**:
  - Создаёшь новую заметку `dev-log/2026-06-09-название.md`
  - Записываешь проблемы и решения
  - Сохраняешь

**4. Сниппет (10-15 мин, если сниппет готов):**
- В **Obsidian**:
  - Создаёшь заметку `snippets/arduino/название.md`
  - Копируешь код из VS Code
  - Добавляешь схему подключения
  - Описываешь, как работает

**5. Git (5 мин):**
```powershell
# В PowerShell — проекты
cd "C:\Users\Manager\Documents\embedded-projects"
git add .
git commit -m "Blink: рабочий код с объяснениями"
git push

# В PowerShell — знания
cd "C:\Users\Manager\Documents\Obsidian Vault"
git add .
git commit -m "Dev-log: 2026-06-09 blink"
git push
```

---

## ✅ Чек-лист завершения сессии

- [ ] Код работает и загружен на плату
- [ ] Dev-log обновлён (записаны проблемы и решения)
- [ ] Сниппет создан (если проект завершён)
- [ ] Git push сделан для проектов
- [ ] Git push сделан для vault
- [ ] Записана задача на следующую сессию

---

## 🎯 Главное правило

**VS Code** → только код и отладка  
**Obsidian** → только знания и dev-log  
**PowerShell** → только Git и навигация

Не смешивай! Не пиши код в Obsidian, не веди dev-log в VS Code.