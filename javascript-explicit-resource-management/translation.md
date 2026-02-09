# Перевод: Явное управление ресурсами в JavaScript (Explicit Resource Management)

Оригинал: [Explicit resource management in JavaScript - Matt Smith](https://allthingssmitty.com/2026/02/02/explicit-resource-management-in-javascript/)

### Основная идея
Написание JavaScript-кода, который открывает ресурсы (файлы, стримы, блокировки, соединения с БД), всегда требовало от разработчика помнить о необходимости их закрытия. Традиционно это решалось через громоздкие конструкции `try / finally`. 

В 2026 году "Явное управление ресурсами" (Explicit Resource Management) становится полноценной частью языка, позволяя рантайму гарантировать очистку ресурсов автоматически.

### 1. Проблема: Мы плохи в очистке ресурсов
Обычный паттерн:
```javascript
const file = await openFile("data.txt");
try {
  // работа с файлом
} finally {
  await file.close();
}
```
Это многословно, повторяемо и легко ломается при рефакторинге. Если ресурсов несколько, порядок их закрытия становится критичным и запутанным.

### 2. Ключевое решение: оператор `using`
Оператор `using` объявляет ресурс, который будет автоматически очищен, как только он выйдет из области видимости (scope). Очистка привязана к **времени жизни** переменной, а не к потоку управления.

```javascript
using file = await openFile("data.txt");
// работа с файлом
// файл автоматически закроется в конце этого блока {}
```

### 3. Как это работает под капотом
Ресурсы должны реализовать специальные символы:
- `Symbol.dispose` для синхронной очистки.
- `Symbol.asyncDispose` для асинхронной очистки.

Пример класса с поддержкой `using`:
```javascript
class FileHandle {
  async [Symbol.asyncDispose]() {
    await this.close();
  }
}
```

### 4. `await using`
Если очистка асинхронна, используется `await using`. JavaScript дождется завершения процесса удаления (disposal), прежде чем продолжить выполнение кода за пределами области видимости.

### 5. Стек ресурсов (Stacking)
При использовании нескольких `using`, ресурсы очищаются в обратном порядке (как в стеке):
```javascript
await using file = await openFile("data.txt");
using lock = await acquireLock();
// Сначала освободится lock, затем закроется file.
```

### 6. Escape-hatch: DisposableStack
Если ресурс не вписывается в блок `{}` (например, условное создание или сложный рефакторинг), можно использовать `DisposableStack` или `AsyncDisposableStack`:
```javascript
const stack = new AsyncDisposableStack();
const file = stack.use(await openFile("data.txt"));
// ...
await stack.disposeAsync(); // явный вызов очистки для всего стека
```

### 7. Где это применять?
Не только на бэкенде, но и во фронтенде:
- Web Streams
- navigator.locks
- Обсерверы и подписки (Observers/Subscriptions)
- Транзакции IndexedDB

### Статус поддержки (на начало 2026 г.)
- **Chrome 123+**, **Firefox 119+**, **Node.js 20.9+** — полная поддержка.
- Safari — в процессе.

### Итог
`using` не заменяет `try / finally` полностью, но дает лучший "выбор по умолчанию": меньше шаблонного кода, меньше утечек памяти и более явные намерения в коде.
