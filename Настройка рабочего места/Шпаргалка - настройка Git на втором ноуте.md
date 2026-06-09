Когда придёшь домой, открывай PowerShell и делай по порядку:

**1. Установить Git** (если нет):
- https://git-scm.com/download/win → скачать → установить
- При установке: выбрать VS Code как редактор, "Git from command line", SChannel, MinTTY, Fast-forward or merge, Credential Manager
**2. Проверить:**
git --version
**3. Настроить один раз:**
git config --global user.name "IvanEmbedded"
git config --global user.email "ivan.khvalev2002@gmail.com"
git config --global init.defaultBranch main
**4. Скачать свои репозитории:**
cd C:\Users\Manager\Documents
git clone https://github.com/Ivan-dev-mc/embedded-learning.git
git clone https://github.com/Ivan-dev-mc/embedded-projects.git
**5. Открыть vault в Obsidian:**
- Open folder as vault → выбрать папку `embedded-learning`
**Всё.** Рабочее место готово.
[[Шпаргалка - что где делаем]], [[Промпт для DeepSeek (Преподаватель теории)]], [[Промпт для Qwen (Код-ревьюер и наставник)]],