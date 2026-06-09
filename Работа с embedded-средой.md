## 🏠 ДОМА (первая настройка)

### 1. Установить софт
```powershell
# Git
https://git-scm.com/download/win → установить

# VS Code
https://code.visualstudio.com/ → установить

# PlatformIO (в VS Code)
VS Code → Extensions → PlatformIO IDE → Install
```

### 2. Скачать репозитории
```powershell
cd C:\Users\Manager\Documents

# База знаний
git clone https://github.com/Ivan-dev-mc/embedded-learning.git

# Проекты
git clone https://github.com/Ivan-dev-mc/embedded-projects.git
```

### 3. Открыть в Obsidian
- Obsidian → Open folder as vault
- Выбрать: `C:\Users\Manager\Documents\embedded-learning`

---

## 💼 НА РАБОТЕ (ежедневный workflow)

### Утром (начало сессии)

**PowerShell:**
```powershell
# Обновить всё
cd C:\Users\Manager\Documents\embedded-projects
git pull

cd C:\Users\Manager\Documents\embedded-learning
git pull
```

**VS Code:**
- Открыть папку проекта: `embedded-projects/arduino/blink`
- Или создать новый проект

**Obsidian:**
- Открыть vault `embedded-learning`
- Создать dev-log запись (если есть проблемы)

---

### Вечером (конец сессии)

**PowerShell:**
```powershell
# Сохранить проекты
cd C:\Users\Manager\Documents\embedded-projects
git add .
git commit -m "Описание что сделал"
git push

# Сохранить знания
cd C:\Users\Manager\Documents\embedded-learning
git add .
git commit -m "Dev-log: дата"
git push
```

---

## ⌨️ БЫСТРЫЕ КОМАНДЫ

### Git (база)
```powershell
git status           # Что изменилось
git add .            # Добавить все файлы
git commit -m "..."  # Закоммитить
git push             # Отправить на GitHub
git pull             # Скачать изменения
```

### PlatformIO (в VS Code)
```
Ctrl+Shift+B   # Собрать проект
Ctrl+Alt+U     # Залить прошивку
Ctrl+Alt+S     # Serial Monitor
Ctrl+Shift+P   # Command Palette
```

### Создать PlatformIO проект
```powershell
cd C:\Users\Manager\Documents\embedded-projects\arduino
# VS Code → Ctrl+Shift+P → PlatformIO: New Project
# Имя: blink, Плата: Arduino Uno, Папка: текущая
```

---

## 📁 СТРУКТУРА ПАПOK

```
embedded-learning/     ← Obsidian (знания, dev-log, план)
embedded-projects/     ← PlatformIO (код проектов)
```

---

## 🔧 ЧТО ГДЕ ДЕЛАТЬ

| Задача | Инструмент |
|--------|------------|
| Писать код | VS Code + PlatformIO |
| Вести dev-log | Obsidian |
| Сохранять прогресс | PowerShell (Git) |
| Объяснить строку кода | Qwen |
| Объяснить теорию | DeepSeek |
| Найти функцию/docs | Perplexity |

---

## ⚡ ЕСЛИ ЧТО-ТО СЛОМАЛОСЬ

**Git не пушит:**
```powershell
git pull --rebase
git push
```

**PlatformIO не видит порт:**
- Переподключи Arduino USB
- Смени USB-кабель
- Проверь Диспетчер устройств → Порты COM

**Serial Monitor кракозябры:**
- Проверь `monitor_speed = 9600` в `platformio.ini`

---

## ✅ ЧЕК-ЛИСТ НАЧАЛА РАБОТЫ

- [ ] PowerShell: `git pull` в обоих репо
- [ ] VS Code: открыт проект
- [ ] Obsidian: открыт vault, создан dev-log
- [ ] Arduino подключена

## ✅ ЧЕК-ЛИСТ КОНЦА РАБОТЫ

- [ ] Код работает
- [ ] `git add . → commit → push` (проекты)
- [ ] `git add . → commit → push` (знания)
- [ ] Записана задача на завтра
