# Патч: Ограничение области поиска памяти (Memory Discovery) в gemini-cli

## Описание

Этот патч добавляет возможность ограничить область поиска файлов памяти только текущей директорией, без поиска вверх/вниз по дереву каталогов. Это ускоряет работу и повышает релевантность в больших проектах.

- Добавлен флаг `currentDirectoryOnly` в настройки CLI (по умолчанию: **true**).
- Изменена логика поиска памяти в core.
- Обновлены тесты для проверки новой функциональности.

## Затронутые файлы

- packages/core/src/utils/memoryDiscovery.ts
- packages/cli/src/config/config.ts
- packages/cli/src/settings/schema.ts
- packages/cli/src/__tests__/memoryDiscovery.test.ts (или аналогичный файл с тестами)

## Изменения по файлам

### 1. packages/core/src/utils/memoryDiscovery.ts

**Что меняется:**
- В функцию memory discovery добавлен новый аргумент `currentDirectoryOnly`.
- Если он true — поиск памяти только в текущей директории, без сканирования вверх/вниз.

**Пример до:**
```ts
export function discoverMemoryFiles(cwd: string, ...другиеПараметры) {
  // ... поиск вверх и вниз по дереву
}
```

**Пример после:**
```ts
export function discoverMemoryFiles(cwd: string, currentDirectoryOnly: boolean = true, ...другиеПараметры) {
  if (currentDirectoryOnly) {
    // ищем только в cwd
  } else {
    // стандартный поиск вверх/вниз
  }
}
```

**Особенности:**
- В core больше не используется "пустая строка" как сигнал для поиска из home — теперь CLI всегда передает валидный путь.

### 2. packages/cli/src/config/config.ts

**Что меняется:**
- В конфиг добавлен флаг `currentDirectoryOnly` (по умолчанию: true).
- Логика: если пользователь в home и не включен currentDirectoryOnly — поиск памяти не выполняется (во избежание долгого сканирования home).

**Пример до/после:**
```ts
// ... до
const memoryFiles = discoverMemoryFiles(cwd, ...);
// ... после
const memoryFiles = discoverMemoryFiles(
  currentDirectoryOnly ? cwd : (inHomeDir ? "" : cwd),
  currentDirectoryOnly ?? true,
  ...
);
```

### 3. packages/cli/src/settings/schema.ts

**Что меняется:**
- В схему настроек добавлен новый boolean-параметр `currentDirectoryOnly` с описанием и default: true.

**Пример:**
```ts
currentDirectoryOnly: {
  type: "boolean",
  description: "If true, restrict memory discovery to the current directory only.",
  default: true
}
```

### 4. Тесты

- Добавлен тест, который проверяет, что при включенном currentDirectoryOnly память ищется только в текущей директории.
- Обновлены существующие тесты с учетом нового аргумента и значения по умолчанию.

**Пример:**
```ts
it("should only load memory from current directory when currentDirectoryOnly is true (default)", () => {
  // ... тестовая логика
});
```

## Edge Cases

- Если пользователь запускает CLI из home, и currentDirectoryOnly=false — память не ищется (во избежание сканирования home).
- Если currentDirectoryOnly=true — память ищется только в home, если запуск из home.

## Ссылки

- Оригинальный PR: https://github.com/google-gemini/gemini-cli/pull/8331
- Обсуждение и ревью: см. комментарии в PR

---

**Примечание:**  
Этот патч меняет поведение по умолчанию: теперь память ищется только в текущей директории, если явно не указано иное.  
Для интеграции в другие версии gemini-cli требуется ручная проверка совместимости сигнатур функций и схемы настроек.

---

## Применение патча вручную

```sh
curl -L -o 8331.patch https://patch-diff.githubusercontent.com/raw/google-gemini/gemini-cli/pull/8331.patch
git apply -v 8331.patch
git apply --3way 8331.patch
# git apply --reject --whitespace=fix 8331.patch
# git apply -C1 8331.patch

git format-patch -1 HEAD --no-stat
git apply --3way 0001-feat-should-only-load-memory-from-current-directory-.patch
```

---
