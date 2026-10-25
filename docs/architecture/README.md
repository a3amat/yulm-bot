# Архитектура yulm-bot

Каталог содержит архитектурные схемы проекта (Mermaid + PlantUML).
GitHub автоматически рендерит блоки `mermaid` в Markdown.

## Список схем

| Файл | Описание |
|---|---|
| [01-high-level.md](01-high-level.md) | High-Level архитектура: общий обзор системы |
| [02-component-diagram.md](02-component-diagram.md) | Диаграмма компонентов бота |
| [03-sequence-diagram.md](03-sequence-diagram.md) | Последовательность обработки сообщения пользователя |
| [04-deployment-diagram.md](04-deployment-diagram.md) | Схема развёртывания (deployment) |
| [05-data-model.md](05-data-model.md) | Модель данных и хранилище |
| [architecture.puml](architecture.puml) | Компонентная диаграмма в PlantUML |

## Как читать

- Все схемы — Mermaid, кроме `architecture.puml` (PlantUML, рендерится через плагин или `plantuml architecture.puml`).
- При изменении архитектуры обновляйте соответствующую схему в том же PR.
