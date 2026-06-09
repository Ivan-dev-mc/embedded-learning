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