---
status: 🔴 Broken (known issues)
hardware: [Arduino UNO, VL53L0X, LCD 16x2, RGB LED, Potentiometer]
created: 2026-06-05
tags: [prototype, arduino, i2c, lcd]
---
## ✅ Что работает
- VL53L0X по I2C (с pull-up 10кОм)
- RGB LED (общий катод, резисторы 220Ω)
- Потенциометр (analogRead A0)
- LCD 16x2 (подсветка горит)

## ❌ Известные проблемы
1. **LCD показывает чёрные квадраты** — текст не виден
   - Причина: контрастность V0 слишком высокая
   - Решение: крутить потенциометр V0 до появления текста
   
2. **RGB горит белым, не реагирует на датчик**
   - Причина: код не переключает цвета / пины перепутаны
   - Решение: проверить подключение D9/D10/D11, логику `analogWrite()`

3. **Сплетение проводов** — ненадёжные контакты
   - Решение: цветовая дисциплина, вторая макетка
## 📊 Полная схема подключения

### LCD 16x2
| Пин LCD | Подключение | Примечание |
|---------|-------------|------------|
| 1. GND | GND (чёрная шина) | Земля |
| 2. VDD | 5V (красная шина) | Питание |
| 3. V0 | Средняя ножка потенциометра | Контрастность |
| 4. RS | **D2** Arduino | Register Select |
| 5. RW | GND (чёрная шина) | ⚠️ **Только запись!** |
| 6. E | **D3** Arduino | Enable |
| 11. D4 | **D4** Arduino | Data 4 |
| 12. D5 | **D5** Arduino | Data 5 |
| 13. D6 | **D6** Arduino | Data 6 |
| 14. D7 | **D7** Arduino | Data 7 |
| 15. BLA | 5V через резистор **220Ω** | Подсветка (+) |
| 16. BLK | GND (чёрная шина) | Подсветка (-) |

---

### VL53L0X (I2C)
| Пин датчика | Подключение       | Примечание                    |
| ----------- | ----------------- | ----------------------------- |
| VIN         | 5V (красная шина) | Питание                       |
| GND         | GND (чёрная шина) | Земля                         |
| SCL         | **A5** Arduino    | I2C Clock + pull-up 10кΩ к 5V |
| SDA         | **A4** Arduino    | I2C Data + pull-up 10кΩ к 5V  |
| GPIO1       | Не подключать     | Не используется               |
| XSHUT       | Не подключать     | Не используется               |

---

### RGB LED (общий катод)
| Ножка LED | Подключение | Примечание |
|-----------|-------------|------------|
| R (Red) | Через резистор **220Ω** → **D9** | Красный канал |
| G (Green) | Через резистор **220Ω** → **D10** | Зелёный канал |
| B (Blue) | Через резистор **220Ω** → **D11** | Синий канал |
| COM (общий) | GND (чёрная шина) | Самый длинный пин |

---

### Потенциометр 10кΩ
| Ножка | Подключение | Примечание |
|-------|-------------|------------|
| Левая | 5V (красная шина) | Питание |
| **Средняя** | **A0** Arduino + LCD V0 | **Два провода!** |
| Правая | GND (чёрная шина) | Земля |

---

## ⚠️ Критические моменты:

1. **LCD RW (пин 5)** → **только GND**, иначе не работает
2. **Pull-up резисторы 10кΩ** на I2C:
   - A4 (SDA) → 10кΩ → 5V
   - A5 (SCL) → 10кΩ → 5V
3. **Потенциометр** подключён **двойным назначением**:
   - Средняя ножка → LCD V0 (контрастность)
   - Средняя ножка → A0 (чтение для порога)
4. **RGB резисторы** — **220Ω на каждый канал** (без них сожжёшь пины!)
5. **LCD подсветка** — резистор **220Ω на BLA** (пин 15)

---

## 📋 Итоговая распиновка Arduino UNO:

| Пин Arduino | Компонент | Тип |
|-------------|-----------|-----|
| **A0** | Потенциометр (средняя ножка) | Analog Input |
| **A4** | VL53L0X SDA + pull-up 10кΩ | I2C |
| **A5** | VL53L0X SCL + pull-up 10кΩ | I2C |
| **D2** | LCD RS | Digital Output |
| **D3** | LCD E | Digital Output |
| **D4** | LCD D4 | Digital Output |
| **D5** | LCD D5 | Digital Output |
| **D6** | LCD D6 | Digital Output |
| **D7** | LCD D7 | Digital Output |
| **D9** | RGB Red (через 220Ω) | PWM Output |
| **D10** | RGB Green (через 220Ω) | PWM Output |
| **D11** | RGB Blue (через 220Ω) | PWM Output |
| **5V** | Питание всех компонентов | Power |
| **GND** | Общая земля | Ground |

**Итого занято:** 12 пинов из 14 цифровых + 3 аналоговых. Свободно: D0, D1 (не используй — это Serial!), D8, D12, D13, A1-A3.

Без воды. Проверяй по таблице.

## 💻 Код
```cpp
#include <Arduino.h>
#include <Wire.h>
#include <VL53L0X.h>
#include <LiquidCrystal.h>

// Пины LCD (RS, E, D4, D5, D6, D7)
LiquidCrystal lcd(2, 3, 4, 5, 6, 7);

// Пины RGB (общий катод, поэтому 255 = горит, 0 = выкл)
#define R_PIN 9
#define G_PIN 10
#define B_PIN 11

// Пин потенциометра
#define POT_PIN A0

VL53L0X sensor;

void setup() {
    Serial.begin(9600);
    delay(1000);
    
    // Инициализация LCD
    lcd.begin(16, 2);
    lcd.print("System Boot...");
    delay(1000);
    
    // Инициализация RGB
    pinMode(R_PIN, OUTPUT);
    pinMode(G_PIN, OUTPUT);
    pinMode(B_PIN, OUTPUT);
    
    // Инициализация VL53L0X
    Wire.begin();
    sensor.setTimeout(500);
    if (!sensor.init()) {
        lcd.clear();
        lcd.print("VL53L0X ERROR!");
        Serial.println("VL53L0X не найден!");
        while (1); // Зависаем, если датчик не подключён
    }
    sensor.setMeasurementTimingBudget(33000);
    sensor.startContinuous(100);
    
    lcd.clear();
    lcd.print("Ready!");
    delay(1000);
}

void loop() {
    // 1. Читаем расстояние
    uint16_t distance = sensor.readRangeContinuousMillimeters();
    if (sensor.timeoutOccurred()) {
        distance = 0; // Если ошибка, считаем 0
    }

    // 2. Читаем потенциометр и переводим в мм (0 - 1000)
    int potValue = analogRead(POT_PIN);
    int threshold = map(potValue, 0, 1023, 0, 1000);

    // 3. Логика RGB
    if (distance < threshold) {
        // Близко - Красный
        analogWrite(R_PIN, 200); // Не 255, чтобы не слепило и не жрало ток
        analogWrite(G_PIN, 0);
        analogWrite(B_PIN, 0);
    } else {
        // Далеко - Зелёный
        analogWrite(R_PIN, 0);
        analogWrite(G_PIN, 200);
        analogWrite(B_PIN, 0);
    }

    // 4. Вывод на LCD (без lcd.clear(), чтобы не мерцало)
    lcd.setCursor(0, 0);
    lcd.print("Dist: ");
    lcd.print(distance);
    lcd.print(" mm   "); // Пробелы, чтобы затереть старые цифры

    lcd.setCursor(0, 1);
    lcd.print("Thr:  ");
    lcd.print(threshold);
    lcd.print(" mm   ");

    // 5. Дублируем в Serial Monitor для отладки
    Serial.print("Dist: ");
    Serial.print(distance);
    Serial.print(" | Threshold: ");
    Serial.println(threshold);

    delay(100);
}
```
## 🔧 Что нужно исправить
- [ ] Отрегулировать LCD контрастность
- [ ] Проверить пины RGB
- [ ] Переписать логику переключения цветов
- [ ] Добавить pull-down на кнопку (если будет)

## 💡 Инсайты
- I2C без pull-up = timeout
- LCD V0 критичен для видимости текста
- Макетка 830 точек — мало для 4 компонентов