# Youer: удаление Bukkit plugin overhead

**Цель:** убрать накладные расходы от поддержки плагинов (event dispatch, создание event-объектов, Bukkit API обёртки).
**Контекст:** Youer — форк NeoForge с Paper/Purpur патчами. Плагины не используются.
**Не трогаем:** патчи оптимизации, патчи геймплея без plugin API.

---

## Уже сделано

### ✅ Шаг 0 — Глобальное отключение event dispatch
**Файл:** `src/main/java/io/papermc/paper/plugin/manager/PaperEventManager.java`

В `callEvent()` добавлен early return ПОСЛЕ `YouerPlugin.registerListener()`:
```java
YouerPlugin.registerListener(event); // Youer-фичи (back, GUI, ban) работают
if (true) return; // YOUER-NOPLUGINS: skip plugin event dispatch — JIT removes as dead code
```
**Эффект:** все ~500+ вызовов callEvent становятся no-op. Покрывает ~80% overhead.
`YouerPlugin.registerListener()` оставлен — там реальные фичи (back, GUI, BanListener).

---

## Категория 1: EASY — безопасно убрать

Паттерн: event создаётся, callEvent вызывается, результат не читается обратно.
Действие: удалить строки создания event-объекта + вызов callEvent + любые Bukkit-обёртки созданные только для этого event.

| Патч | Что убрать |
|------|-----------|
| `0282-Add-PlayerPostRespawnEvent` | fire-and-forget, нет isCancelled, нет reads |
| `0216-Vanished-players-don-t-have-rights` | canSee() checks без чтения event результата |
| `0660-Fix-cancelled-powdered-snow-bucket-placement` | minimal event interaction |
| `0718-Add-various-missing-EntityDropItemEvent-calls` | fire-and-forget с skip если cancelled, нет value reads |
| `0727-Add-and-fix-missing-BlockFadeEvents` | callBlockFadeEvent() + isCancelled чтобы предотвратить block change, ванильная логика остаётся |
| `0735-Fire-EntityChangeBlockEvent-in-more-places` | fire-and-forget, skip block change if cancelled, нет value reads |
| `0739-Call-BlockPhysicsEvent-more-often` | callEvent() перед executeUpdate(), базовая отмена |
| `0743-use-BlockFormEvent-for-mud-converting-into-clay` | handleBlockFormEvent() boolean wrapper поверх существующей логики |
| `0820-Call-BlockGrowEvent-for-missing-blocks` | handleBlockGrowEvent() return check, блок просто не растёт если cancelled — ванильный рост |

---

## Категория 2: MEDIUM — нужно подумать

Паттерн A: есть isCancelled() но ванильная логика ниже не изменена — убрать блок отмены, ванильное поведение остаётся.
Паттерн B: читает незначительные данные из event, но с no-op dispatch данные = дефолтные.

| Патч | Паттерн | Что делать |
|------|---------|-----------|
| `0080-EntityPathfindEvent` | A | Убрать блок проверки callEvent() для позиций — pathfinding всегда разрешён (ванильное поведение) |
| `0114-Prevent-Pathfinding-out-of-World-Border` | A | Зависит от 0080, убрать вместе |
| `0194-Add-EntityTeleportEndGatewayEvent` | B | isCancelled всегда false, getTo() = исходный to — убрать блок, оставить оригинальный `to` |
| `0267-force-entity-dismount-during-teleportation` | сложный | Добавляет suppressCancellation overrides в иерархию Entity/LivingEntity/Player/Shulker — структурное изменение, не просто event call. Рассмотреть отдельно. |
| `0410-Extend-block-drop-capture-to-capture-all-items-added` | A | captureDrops list, небольшое структурное изменение |
| `0460-Add-WorldGameRuleChangeEvent` | B | Читает event.getValue() но с no-op = дефолт. Убрать блок отмены. |
| `0540-Add-cause-to-Weather-ThunderChangeEvents` | A | Убрать блок отмены, ванильная смена погоды остаётся |
| `0696-Fire-CauldronLevelChange-on-initial-fill` | A | Убрать check cancelled, котёл заполняется всегда |
| `0748-EntityPickupItemEvent-fixes` | A | isCancelled для skip advancement triggers — убрать блок, advancement всегда триггерится |
| `0777-Fix-inconsistencies-in-dispense-events` | B | Читает event.getItem() — с no-op = исходный item. Потенциально убираемо. |
| `0818-Fix-block-place-logic` | A | isCancelled предотвращает physics/sound — убрать блок, ванильная физика всегда |
| `0830-Call-missing-BlockDispenseEvent` | B | Читает result из event для alternate behavior — нужно проверить что alternate path не ломает логику |
| `0847-Only-capture-actual-tree-growth` | B | Читает BlockGrowEvent, логика независима — изучить |

---

## Категория 3: HARD — почти нереально убрать

Причины: читают значения из event которые меняют логику (drops, locations, velocity, items), или ванильный код структурно переставлен вокруг event lifecycle, или другие патчи зависят от этого API.

| Патч | Почему не трогать |
|------|-----------------|
| `0049-Add-BeaconEffectEvent` | Читает `event.getEffect()` и применяет его — effect определяется через event |
| `0175-Add-more-fields-to-AsyncPreLoginEvent` | Читает profile/UUID из event, модифицирует GameProfile |
| `0242-Improve-death-events` | Комплексно: isCancelled в нескольких местах, drops/equipment зависят от event flow |
| `0281-Fire-event-on-GS4-query` | Читает поля response из event и записывает в query ответ |
| `0284-PlayerDeathEvent-getItemsToKeep` | Читает `event.getItemsToKeep()`, применяет к inventory |
| `0554-Fix-potions-splash-events` | Читает toDamage/toExtinguish/toRehydrate — логика зависит от event данных |
| `0570-Add-PlayerSetSpawnEvent` | Читает `event.getLocation()`, `event.isForced()`, `event.willNotifyPlayer()` |
| `0582-Add-back-EntityPortalExitEvent` | Читает `event.getTo()` и `event.getAfter()` (velocity) |
| `0903-Fix-several-issues-with-EntityBreedEvent` | Читает `event.breedItem`, хранит копию item stack, влияет на breeding flow |

---

## Порядок работы

1. **Начать с Категории 1** — каждый патч отдельным коммитом, минимальный риск
2. **Категория 2** — по одному, после каждого убедиться что компилируется
3. **0267** — рассмотреть отдельно как архитектурное решение
4. **Категория 3** — не трогать; с Шагом 0 они уже не тратят время на dispatch, только выделяют объекты которые GC сразу собирает

## Чего НЕ делать
- Не удалять импорты Bukkit классов — сломает компиляцию (API используется в других местах)
- Не трогать HandlerList, PluginManager, CraftServer — нужны для инициализации
- Не трогать Purpur gameplay патчи
- Не трогать YouerPlugin.registerListener() — там реальные фичи (back, GUI, BanListener)
