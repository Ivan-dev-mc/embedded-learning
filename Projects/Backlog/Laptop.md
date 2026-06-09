**📦 Устройство для практики:**
- Плата от Haier (ноутбук/планшет), модель U3285131P-2S
- Состояние: плата живая, экран/батарея/корпус — мусор
- Разъемы: HDMI, USB 3.0, питание, батарея
- Медный радиатор на SoC

**🎯 Цель:**
Освоить **Embedded Hardware Engineering** + **Board Bring-up** (оживление плат)

**📚 Необходимые навыки:**

1. **Computer Architecture** — x86, SoC, шины данных
2. **Schematic Reading** — чтение схем и boardview
3. **Power Distribution** — цепи питания, power sequencing
4. **Protocols** — I2C, SPI, UART, PCIe, USB
5. **Embedded Controllers** — EC/KBC, PMIC
6. **Hardware Debugging** — мультиметр, осциллограф, логический анализатор
7. **Firmware** — работа с BIOS/EC, программаторы

**🔍 Roadmap для изучения:**
1. `x86 laptop power sequence tutorial`
2. `how to read boardview and schematic`
3. `laptop power rails debugging`
4. `I2C SPI protocol basics`
5. `UEFI BIOS architecture`

**🛠 Практический план:**
1. Сохранить плату (антистатик/мультифора + силикагель)
2. Найти схему по маркировке на текстолите
3. Изучить цепи питания: дежурка 3V/5V → кнопка Power → EC → основные напряжения
4. Научиться прозванивать ключевые точки
5. Разобраться с EC контроллером (даташит)
6. Попытаться запустить с внешним питанием + HDMI

**💡 Ресурсы:**
- Badcaps.net (форум)
- ElectroTanya (схемы)
- YouTube: NorthridgeFix, Learn Electronics Repair

**📌 Итог:** Плата — учебный полигон для отработки **board bring-up** и диагностики hardware. 