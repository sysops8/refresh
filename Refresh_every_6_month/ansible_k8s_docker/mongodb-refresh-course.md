# MongoDB Refresh: Ежегодный/Полугодовой курс для DevOps/SysAdmin

**Цель:** Освежить в памяти ключевые концепции MongoDB за 2-3 часа практики и узнать 1-2 новые продвинутые техники.

**Формат:** Каждый раздел состоит из:
1. **Краткой теории (Напоминалка)**: Самое главное, что вы могли забыть
2. **Практического задания**: Реальная задача, которую нужно решить
3. **Бонусного задания (для роста)**: Задача посложнее или с использованием новой фичи

**Предварительные требования:**
- MongoDB установлен (локально или Docker)
- mongosh (MongoDB Shell) доступен
- Базовые знания JSON и командной строки

---

## Модуль 1: Установка и базовая настройка (20 минут)

### 🎯 Напоминалка

**Установка MongoDB:**
```bash
# Docker (рекомендуется для тестирования)
docker run --name mongodb -d \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=password123 \
  -v mongodb_data:/data/db \
  mongo:7.0

# Или через Docker Compose
cat > docker-compose.yml <<EOF
version: '3.8'
services:
  mongodb:
    image: mongo:7.0
    container_name: mongodb
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password123
    volumes:
      - mongodb_data:/data/db
    restart: unless-stopped

volumes:
  mongodb_data:
EOF

docker-compose up -d

# Linux (Ubuntu/Debian)
wget -qO - https://www.mongodb.org/static/pgp/server-7.0.asc | sudo apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
sudo apt update
sudo apt install -y mongodb-org
sudo systemctl start mongod
sudo systemctl enable mongod

# macOS
brew tap mongodb/brew
brew install mongodb-community@7.0
brew services start mongodb-community@7.0

# Проверка установки
mongosh --version
```

**Подключение к MongoDB:**
```bash
# Локальное подключение
mongosh

# С аутентификацией
mongosh "mongodb://admin:password123@localhost:27017"

# С URI
mongosh "mongodb://admin:password123@localhost:27017/?authSource=admin"

# К удаленному серверу
mongosh "mongodb://user:pass@host:27017/dbname"

# MongoDB Atlas
mongosh "mongodb+srv://cluster0.xxxxx.mongodb.net/mydb" --username user
```

**Основные команды mongosh:**
```javascript
// Базовые операции
show dbs                    // Список баз данных
use mydb                    // Переключиться на БД (создаст если нет)
db                          // Текущая БД
show collections            // Список коллекций
db.dropDatabase()          // Удалить БД

// Коллекции
db.createCollection("users")
db.users.drop()            // Удалить коллекцию
db.users.renameCollection("customers")

// Статистика
db.stats()
db.users.stats()

// Помощь
help
db.help()
db.users.help()

// Выход
exit
```

**Конфигурационный файл MongoDB** (`/etc/mongod.conf`):
```yaml
# Основные настройки
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  logRotate: reopen

net:
  port: 27017
  bindIp: 127.0.0.1
  maxIncomingConnections: 1000

processManagement:
  timeZoneInfo: /usr/share/zoneinfo
  fork: true
  pidFilePath: /var/run/mongodb/mongod.pid

security:
  authorization: enabled
  javascriptEnabled: true

setParameter:
  enableLocalhostAuthBypass: false

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
```

**Переменные окружения:**
```bash
export MONGODB_URI="mongodb://admin:password123@localhost:27017"
export MONGODB_DATABASE="myapp"
```

### 💻 Задание

Подготовь тестовое окружение:

1. Установи MongoDB через Docker:
   ```bash
   docker run --name mongodb-test -d \
     -p 27017:27017 \
     -e MONGO_INITDB_ROOT_USERNAME=admin \
     -e MONGO_INITDB_ROOT_PASSWORD=password123 \
     mongo:7.0
   ```

2. Подключись к MongoDB:
   ```bash
   mongosh "mongodb://admin:password123@localhost:27017/?authSource=admin"
   ```

3. Создай тестовую базу и коллекцию:
   ```javascript
   use testdb
   db.createCollection("products")
   ```

4. Вставь тестовые данные:
   ```javascript
   db.products.insertMany([
     { name: "Laptop", price: 1200, category: "Electronics", stock: 15 },
     { name: "Mouse", price: 25, category: "Electronics", stock: 150 },
     { name: "Desk", price: 300, category: "Furniture", stock: 30 }
   ])
   ```

5. Проверь данные:
   ```javascript
   db.products.find()
   db.products.countDocuments()
   ```

### 🚀 Бонус (новое)

**Настрой MongoDB Compass** (GUI для MongoDB):
```bash
# Скачать с https://www.mongodb.com/products/compass
# Или через Homebrew на macOS
brew install --cask mongodb-compass

# Connection string:
mongodb://admin:password123@localhost:27017/?authSource=admin
```

**Используй mongosh с JavaScript файлами:**
```javascript
// Создай файл seed.js
// seed.js
db = db.getSiblingDB('testdb');

db.products.drop();

db.products.insertMany([
  { name: "Laptop", price: 1200, category: "Electronics", stock: 15, tags: ["computer", "portable"] },
  { name: "Mouse", price: 25, category: "Electronics", stock: 150, tags: ["accessory"] },
  { name: "Desk", price: 300, category: "Furniture", stock: 30, tags: ["office"] },
  { name: "Chair", price: 200, category: "Furniture", stock: 45, tags: ["office", "ergonomic"] }
]);

print("Database seeded successfully!");
```

```bash
# Выполни скрипт
mongosh "mongodb://admin:password123@localhost:27017/?authSource=admin" seed.js
```

---

## Модуль 2: CRUD операции (25 минут)

### 🎯 Напоминалка

**Create (Вставка документов):**
```javascript
// Вставка одного документа
db.users.insertOne({
  name: "John Doe",
  email: "john@example.com",
  age: 30,
  created_at: new Date()
})

// Вставка нескольких документов
db.users.insertMany([
  { name: "Alice", email: "alice@example.com", age: 25 },
  { name: "Bob", email: "bob@example.com", age: 35 }
])

// Вставка с _id
db.users.insertOne({
  _id: "user001",
  name: "Custom ID User"
})
```

**Read (Чтение документов):**
```javascript
// Все документы
db.users.find()

// С условием
db.users.find({ age: 30 })
db.users.find({ age: { $gt: 25 } })  // больше 25
db.users.find({ age: { $gte: 25 } }) // больше или равно
db.users.find({ age: { $lt: 35 } })  // меньше 35
db.users.find({ age: { $lte: 35 } }) // меньше или равно
db.users.find({ age: { $ne: 30 } })  // не равно 30

// Логические операторы
db.users.find({
  $and: [
    { age: { $gte: 25 } },
    { age: { $lte: 35 } }
  ]
})

db.users.find({
  $or: [
    { age: { $lt: 25 } },
    { age: { $gt: 35 } }
  ]
})

// Поиск в массиве
db.products.find({ tags: "computer" })
db.products.find({ tags: { $in: ["computer", "portable"] } })

// Регулярные выражения
db.users.find({ name: /^John/i })  // начинается с John (case-insensitive)

// Проекция (выбор полей)
db.users.find({}, { name: 1, email: 1, _id: 0 })

// Сортировка
db.users.find().sort({ age: 1 })   // по возрастанию
db.users.find().sort({ age: -1 })  // по убыванию

// Лимит и skip
db.users.find().limit(10)
db.users.find().skip(10).limit(10)  // пагинация

// Подсчет
db.users.countDocuments()
db.users.countDocuments({ age: { $gt: 25 } })

// Один документ
db.users.findOne({ email: "john@example.com" })
```

**Update (Обновление документов):**
```javascript
// Обновление одного документа
db.users.updateOne(
  { email: "john@example.com" },
  { $set: { age: 31 } }
)

// Обновление нескольких документов
db.users.updateMany(
  { age: { $lt: 25 } },
  { $set: { status: "young" } }
)

// Операторы обновления
db.users.updateOne(
  { email: "john@example.com" },
  {
    $set: { age: 31 },              // установить значение
    $unset: { temp_field: "" },     // удалить поле
    $inc: { login_count: 1 },       // увеличить на 1
    $mul: { score: 1.1 },           // умножить на 1.1
    $rename: { old_name: "new_name" }, // переименовать поле
    $currentDate: { last_modified: true }, // установить текущую дату
    $push: { tags: "new_tag" },     // добавить в массив
    $pull: { tags: "old_tag" },     // удалить из массива
    $addToSet: { tags: "unique_tag" } // добавить если нет
  }
)

// Upsert (вставка если не найдено)
db.users.updateOne(
  { email: "new@example.com" },
  { $set: { name: "New User", age: 28 } },
  { upsert: true }
)

// Replace (полная замена документа)
db.users.replaceOne(
  { email: "john@example.com" },
  { name: "John Smith", email: "john@example.com", age: 32 }
)
```

**Delete (Удаление документов):**
```javascript
// Удаление одного документа
db.users.deleteOne({ email: "john@example.com" })

// Удаление нескольких документов
db.users.deleteMany({ age: { $lt: 18 } })

// Удаление всех документов
db.users.deleteMany({})
```

**Операторы запросов:**
```javascript
// Сравнение
$eq    // равно
$ne    // не равно
$gt    // больше
$gte   // больше или равно
$lt    // меньше
$lte   // меньше или равно
$in    // в массиве значений
$nin   // не в массиве значений

// Логические
$and   // И
$or    // ИЛИ
$not   // НЕ
$nor   // НИ ... НИ

// Элементы
$exists // поле существует
$type   // тип поля

// Массивы
$all       // все элементы присутствуют
$elemMatch // хотя бы один элемент соответствует
$size      // размер массива

// Примеры
db.products.find({ price: { $exists: true } })
db.products.find({ tags: { $all: ["computer", "portable"] } })
db.products.find({ tags: { $size: 2 } })
```

### 💻 Задание

Работа с коллекцией продуктов:

1. Создай коллекцию с продуктами:
   ```javascript
   use shop
   db.products.insertMany([
     { name: "iPhone 15", price: 999, category: "Electronics", stock: 50, tags: ["smartphone", "apple"], rating: 4.8 },
     { name: "Samsung Galaxy S24", price: 899, category: "Electronics", stock: 45, tags: ["smartphone", "samsung"], rating: 4.7 },
     { name: "MacBook Pro", price: 2499, category: "Computers", stock: 20, tags: ["laptop", "apple"], rating: 4.9 },
     { name: "Dell XPS 15", price: 1799, category: "Computers", stock: 30, tags: ["laptop", "dell"], rating: 4.6 },
     { name: "iPad Air", price: 599, category: "Tablets", stock: 60, tags: ["tablet", "apple"], rating: 4.7 }
   ])
   ```

2. Выполни следующие запросы:
   ```javascript
   // Найди все продукты дороже $1000
   db.products.find({ price: { $gt: 1000 } })
   
   // Найди все продукты Apple
   db.products.find({ tags: "apple" })
   
   // Найди продукты категории Electronics с рейтингом выше 4.7
   db.products.find({
     category: "Electronics",
     rating: { $gt: 4.7 }
   })
   
   // Получи только названия и цены (без _id)
   db.products.find({}, { name: 1, price: 1, _id: 0 })
   
   // Найди 3 самых дорогих продукта
   db.products.find().sort({ price: -1 }).limit(3)
   ```

3. Обнови данные:
   ```javascript
   // Увеличь цену всех продуктов Apple на 10%
   db.products.updateMany(
     { tags: "apple" },
     { $mul: { price: 1.1 } }
   )
   
   // Уменьши stock на 1 для iPhone 15
   db.products.updateOne(
     { name: "iPhone 15" },
     { $inc: { stock: -1 } }
   )
   
   // Добавь новый тег "premium" для всех продуктов дороже $1500
   db.products.updateMany(
     { price: { $gt: 1500 } },
     { $addToSet: { tags: "premium" } }
   )
   ```

4. Удали продукты:
   ```javascript
   // Удали продукты с нулевым stock (если есть)
   db.products.deleteMany({ stock: 0 })
   ```

### 🚀 Бонус (новое)

**Используй Aggregation Pipeline для сложных запросов:**
```javascript
// Средняя цена по категориям
db.products.aggregate([
  {
    $group: {
      _id: "$category",
      avgPrice: { $avg: "$price" },
      totalStock: { $sum: "$stock" },
      count: { $sum: 1 }
    }
  },
  { $sort: { avgPrice: -1 } }
])

// Продукты с ценой выше средней
db.products.aggregate([
  {
    $group: {
      _id: null,
      avgPrice: { $avg: "$price" }
    }
  },
  {
    $lookup: {
      from: "products",
      let: { avg: "$avgPrice" },
      pipeline: [
        { $match: { $expr: { $gt: ["$price", "$$avg"] } } }
      ],
      as: "expensive_products"
    }
  }
])
```

**Используй Transaction для атомарных операций:**
```javascript
// Требуется replica set
const session = db.getMongo().startSession()
session.startTransaction()

try {
  const productsCol = session.getDatabase("shop").products
  const ordersCol = session.getDatabase("shop").orders
  
  // Уменьши stock
  productsCol.updateOne(
    { name: "iPhone 15" },
    { $inc: { stock: -1 } }
  )
  
  // Создай заказ
  ordersCol.insertOne({
    product: "iPhone 15",
    quantity: 1,
    date: new Date()
  })
  
  session.commitTransaction()
} catch (error) {
  session.abortTransaction()
  print("Transaction aborted: " + error)
} finally {
  session.endSession()
}
```

---

## Модуль 3: Индексы и производительность (30 минут)

### 🎯 Напоминалка

**Индексы в MongoDB:**
```javascript
// Создание индексов
db.users.createIndex({ email: 1 })        // По возрастанию
db.users.createIndex({ age: -1 })         // По убыванию
db.users.createIndex({ email: 1, age: 1 }) // Составной индекс

// Уникальный индекс
db.users.createIndex({ email: 1 }, { unique: true })

// Partial index (индекс с условием)
db.users.createIndex(
  { email: 1 },
  { partialFilterExpression: { age: { $gte: 18 } } }
)

// TTL индекс (автоудаление)
db.sessions.createIndex(
  { created_at: 1 },
  { expireAfterSeconds: 3600 }  // Удалить через 1 час
)

// Text индекс (полнотекстовый поиск)
db.articles.createIndex({ title: "text", content: "text" })

// Geospatial индекс
db.places.createIndex({ location: "2dsphere" })

// Просмотр индексов
db.users.getIndexes()

// Удаление индексов
db.users.dropIndex("email_1")
db.users.dropIndexes()  // Удалить все кроме _id

// Информация об индексе
db.users.getIndexes()
db.users.totalIndexSize()
```

**Explain - анализ запросов:**
```javascript
// Базовый explain
db.users.find({ email: "john@example.com" }).explain()

// Детальный explain
db.users.find({ email: "john@example.com" }).explain("executionStats")

// Полный explain
db.users.find({ email: "john@example.com" }).explain("allPlansExecution")

// Важные метрики:
// - executionTimeMillis: время выполнения
// - totalDocsExamined: просканировано документов
// - totalKeysExamined: использовано ключей индекса
// - stage: IXSCAN (использован индекс) или COLLSCAN (сканирование коллекции)
```

**Оптимизация запросов:**
```javascript
// Плохо - полное сканирование
db.users.find({ age: 30 }).explain("executionStats")
// stage: "COLLSCAN" - плохо

// Хорошо - использует индекс
db.users.createIndex({ age: 1 })
db.users.find({ age: 30 }).explain("executionStats")
// stage: "IXSCAN" - хорошо

// Покрывающий индекс (Covered Query)
db.users.createIndex({ email: 1, name: 1 })
db.users.find(
  { email: "john@example.com" },
  { email: 1, name: 1, _id: 0 }
).explain("executionStats")
// totalDocsExamined: 0 - отлично!
```

**Профайлер MongoDB:**
```javascript
// Включить профайлер
db.setProfilingLevel(2)  // 0=off, 1=slow ops, 2=all ops

// Посмотреть медленные запросы
db.system.profile.find().sort({ ts: -1 }).limit(5)

// Медленные запросы с условием
db.system.profile.find({
  millis: { $gt: 100 }
}).sort({ millis: -1 })

// Выключить профайлер
db.setProfilingLevel(0)
```

**CurrentOp - текущие операции:**
```javascript
// Текущие операции
db.currentOp()

// Медленные операции
db.currentOp({ "secs_running": { $gt: 5 } })

// Убить операцию
db.killOp(opid)
```

**Статистика коллекций:**
```javascript
// Статистика коллекции
db.users.stats()

// Размер коллекции
db.users.dataSize()
db.users.storageSize()
db.users.totalIndexSize()

// Количество документов
db.users.countDocuments()
db.users.estimatedDocumentCount()  // Быстрее, но приблизительно
```

### 💻 Задание

Оптимизация производительности:

1. Создай тестовую коллекцию с большим количеством документов:
   ```javascript
   use performance_test
   
   // Функция генерации тестовых данных
   function generateUsers(count) {
     let users = [];
     for (let i = 0; i < count; i++) {
       users.push({
         email: `user${i}@example.com`,
         name: `User ${i}`,
         age: 18 + Math.floor(Math.random() * 50),
         city: ["New York", "London", "Tokyo", "Paris", "Berlin"][Math.floor(Math.random() * 5)],
         created_at: new Date(Date.now() - Math.random() * 365 * 24 * 60 * 60 * 1000)
       });
       
       if (users.length >= 1000) {
         db.users.insertMany(users);
         users = [];
       }
     }
     if (users.length > 0) {
       db.users.insertMany(users);
     }
   }
   
   // Создай 10000 пользователей
   generateUsers(10000)
   ```

2. Анализируй запросы без индексов:
   ```javascript
   // Проверь производительность
   db.users.find({ email: "user5000@example.com" }).explain("executionStats")
   // Обрати внимание на executionTimeMillis и totalDocsExamined
   
   db.users.find({ age: { $gte: 30, $lte: 40 } }).explain("executionStats")
   ```

3. Создай индексы и сравни производительность:
   ```javascript
   // Создай индекс на email
   db.users.createIndex({ email: 1 })
   
   // Проверь снова
   db.users.find({ email: "user5000@example.com" }).explain("executionStats")
   // Должен использовать IXSCAN
   
   // Составной индекс
   db.users.createIndex({ city: 1, age: 1 })
   
   // Тест составного индекса
   db.users.find({
     city: "Tokyo",
     age: { $gte: 30 }
   }).explain("executionStats")
   ```

4. Проанализируй использование индексов:
   ```javascript
   // Все индексы
   db.users.getIndexes()
   
   // Размер индексов
   db.users.totalIndexSize()
   
   // Статистика индексов
   db.users.aggregate([
     { $indexStats: {} }
   ])
   ```

### 🚀 Бонус (новое)

**Используй hint() для принудительного использования индекса:**
```javascript
// Принудительно использовать определенный индекс
db.users.find({
  city: "Tokyo",
  age: { $gte: 30 }
}).hint({ city: 1, age: 1 }).explain("executionStats")

// Принудительно не использовать индексы (для тестирования)
db.users.find({ email: "user5000@example.com" }).hint({ $natural: 1 })
```

**Создай wildcard индекс для гибких схем:**
```javascript
// Индекс для всех полей
db.flexible.createIndex({ "$**": 1 })

// Wildcard индекс с условием
db.flexible.createIndex(
  { "metadata.$**": 1 },
  { wildcardProjection: { "metadata.internal": 0 } }
)
```

**Используй $indexStats для анализа:**
```javascript
// Статистика использования индексов
db.users.aggregate([
  { $indexStats: {} },
  { $sort: { "accesses.ops": -1 } }
])

// Найди неиспользуемые индексы
db.users.aggregate([
  { $indexStats: {} },
  { $match: { "accesses.ops": 0 } }
])
```

---

## Модуль 4: Агрегация (Aggregation Pipeline) (30 минут)

### 🎯 Напоминалка

**Aggregation Pipeline - основные стадии:**
```javascript
// $match - фильтрация (как find)
db.orders.aggregate([
  { $match: { status: "completed" } }
])

// $project - выбор и преобразование полей
db.orders.aggregate([
  {
    $project: {
      order_id: "$_id",
      total: 1,
      _id: 0
    }
  }
])

// $group - группировка
db.orders.aggregate([
  {
    $group: {
      _id: "$customer_id",
      totalSpent: { $sum: "$total" },
      orderCount: { $sum: 1 },
      avgOrder: { $avg: "$total" }
    }
  }
])

// $sort - сортировка
db.orders.aggregate([
  { $sort: { total: -1 } }
])

// $limit и $skip - пагинация
db.orders.aggregate([
  { $sort: { total: -1 } },
  { $skip: 10 },
  { $limit: 10 }
])

// $lookup - JOIN (связь коллекций)
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",
      localField: "customer_id",
      foreignField: "_id",
      as: "customer"
    }
  }
])

// $unwind - развернуть массив
db.orders.aggregate([
  { $unwind: "$items" }
])

// $addFields - добавить поля
db.orders.aggregate([
  {
    $addFields: {
      totalWithTax: { $multiply: ["$total", 1.2] }
    }
  }
])

// $bucket - группировка по диапазонам
db.products.aggregate([
  {
    $bucket: {
      groupBy: "$price",
      boundaries: [0, 100, 500, 1000, 5000],
      default: "5000+",
      output: {
        count: { $sum: 1 },
        products: { $push: "$name" }
      }
    }
  }
])
```

**Операторы агрегации:**
```javascript
// Арифметические
$add, $subtract, $multiply, $divide, $mod

// Сравнение
$eq, $ne, $gt, $gte, $lt, $lte

// Логические
$and, $or, $not

// Массивы
$size, $arrayElemAt, $slice, $concatArrays, $filter

// Строки
$concat, $substr, $toLower, $toUpper, $split

// Даты
$year, $month, $dayOfMonth, $hour, $minute, $dateToString

// Условные
$cond, $ifNull, $switch

// Примеры использования
db.orders.aggregate([
  {
    $project: {
      year: { $year: "$order_date" },
      month: { $month: "$order_date" },
      isExpensive: {
        $cond: {
          if: { $gte: ["$total", 1000] },
          then: "yes",
          else: "no"
        }
      }
    }
  }
])
```

**Сложные агрегации:**
```javascript
// Пример: анализ продаж
db.orders.aggregate([
  // 1. Фильтр по году
  {
    $match: {
      order_date: {
        $gte: new Date("2024-01-01"),
        $lt: new Date("2025-01-01")
      }
    }
  },
  // 2. Развернуть items
  { $unwind: "$items" },
  // 3. JOIN с products
  {
    $lookup: {
      from: "products",
      localField: "items.product_id",
      foreignField: "_id",
      as: "product_info"
    }
  },
  // 4. Группировка по категориям
  {
    $group: {
      _id: "$product_info.category",
      totalRevenue: { $sum: { $multiply: ["$items.quantity", "$items.price"] } },
      orderCount: { $sum: 1 },
      avgOrderValue: { $avg: "$total" }
    }
  },
  // 5. Сортировка по выручке
  { $sort: { totalRevenue: -1 } },
  // 6. Форматирование результата
  {
    $project: {
      category: "$_id",
      revenue: { $round: ["$totalRevenue", 2] },
      orders: "$orderCount",
      avgOrder: { $round: ["$avgOrderValue", 2] },
      _id: 0
    }
  }
])
```

### 💻 Задание

Создай систему анализа заказов:

1. Создай коллекции с данными:
   ```javascript
   use ecommerce
   
   // Клиенты
   db.customers.insertMany([
     { _id: 1, name: "Alice Johnson", email: "alice@example.com", city: "New York" },
     { _id: 2, name: "Bob Smith", email: "bob@example.com", city: "London" },
     { _id: 3, name: "Charlie Brown", email: "charlie@example.com", city: "Tokyo" }
   ])
   
   // Продукты
   db.products.insertMany([
     { _id: 101, name: "Laptop", price: 1200, category: "Electronics" },
     { _id: 102, name: "Mouse", price: 25, category: "Electronics" },
     { _id: 103, name: "Desk", price: 300, category: "Furniture" },
     { _id: 104, name: "Chair", price: 200, category: "Furniture" },
     { _id: 105, name: "Monitor", price: 400, category: "Electronics" }
   ])
   
   // Заказы
   db.orders.insertMany([
     {
       _id: 1001,
       customer_id: 1,
       order_date: new Date("2024-01-15"),
       status: "completed",
       items: [
         { product_id: 101, quantity: 1, price: 1200 },
         { product_id: 102, quantity: 2, price: 25 }
       ]
     },
     {
       _id: 1002,
       customer_id: 2,
       order_date: new Date("2024-02-20"),
       status: "completed",
       items: [
         { product_id: 103, quantity: 1, price: 300 },
         { product_id: 104, quantity: 2, price: 200 }
       ]
     },
     {
       _id: 1003,
       customer_id: 3,
       order_date: new Date("2024-03-10"),
       status: "completed",
       items: [
         { product_id: 105, quantity: 1, price: 400 },
         { product_id: 102, quantity: 1, price: 25 }
       ]
     },
     {
       _id: 1004,
       customer_id: 1,
       order_date: new Date("2024-04-05"),
       status: "pending",
       items: [
         { product_id: 104, quantity: 1, price: 200 }
       ]
     }
   ])
   ```

2. Выполни базовые агрегации:
   ```javascript
   // Общая сумма по каждому клиенту
   db.orders.aggregate([
     { $match: { status: "completed" } },
     { $unwind: "$items" },
     {
       $group: {
         _id: "$customer_id",
         totalSpent: { $sum: { $multiply: ["$items.quantity", "$items.price"] } },
         orderCount: { $sum: 1 }
       }
     },
     { $sort: { totalSpent: -1 } }
   ])
   
   // Самые популярные продукты
   db.orders.aggregate([
     { $unwind: "$items" },
     {
       $group: {
         _id: "$items.product_id",
         totalQuantity: { $sum: "$items.quantity" },
         revenue: { $sum: { $multiply: ["$items.quantity", "$items.price"] } }
       }
     },
     { $sort: { totalQuantity: -1 } },
     {
       $lookup: {
         from: "products",
         localField: "_id",
         foreignField: "_id",
         as: "product"
       }
     },
     { $unwind: "$product" },
     {
       $project: {
         product_name: "$product.name",
         quantity_sold: "$totalQuantity",
         revenue: 1,
         _id: 0
       }
     }
   ])
   ```

3. Сложная агрегация - отчет по продажам:
   ```javascript
   // Продажи по категориям с деталями клиентов
   db.orders.aggregate([
     { $match: { status: "completed" } },
     { $unwind: "$items" },
     {
       $lookup: {
         from: "products",
         localField: "items.product_id",
         foreignField: "_id",
         as: "product"
       }
     },
     { $unwind: "$product" },
     {
       $lookup: {
         from: "customers",
         localField: "customer_id",
         foreignField: "_id",
         as: "customer"
       }
     },
     { $unwind: "$customer" },
     {
       $group: {
         _id: {
           category: "$product.category",
           city: "$customer.city"
         },
         revenue: { $sum: { $multiply: ["$items.quantity", "$items.price"] } },
         orders: { $sum: 1 },
         unique_customers: { $addToSet: "$customer_id" }
       }
     },
     {
       $project: {
         category: "$_id.category",
         city: "$_id.city",
         revenue: 1,
         orders: 1,
         customer_count: { $size: "$unique_customers" },
         _id: 0
       }
     },
     { $sort: { revenue: -1 } }
   ])
   ```

### 🚀 Бонус (новое)

**Используй $facet для множественных агрегаций:**
```javascript
// Получи несколько отчетов за один запрос
db.orders.aggregate([
  {
    $facet: {
      // Отчет 1: по месяцам
      byMonth: [
        {
          $group: {
            _id: { $month: "$order_date" },
            orders: { $sum: 1 }
          }
        },
        { $sort: { _id: 1 } }
      ],
      // Отчет 2: по статусам
      byStatus: [
        {
          $group: {
            _id: "$status",
            count: { $sum: 1 }
          }
        }
      ],
      // Отчет 3: топ клиентов
      topCustomers: [
        { $unwind: "$items" },
        {
          $group: {
            _id: "$customer_id",
            total: { $sum: { $multiply: ["$items.quantity", "$items.price"] } }
          }
        },
        { $sort: { total: -1 } },
        { $limit: 3 }
      ]
    }
  }
])
```

**Используй $graphLookup для рекурсивных связей:**
```javascript
// Для иерархических структур (например, категории с подкатегориями)
db.categories.aggregate([
  {
    $graphLookup: {
      from: "categories",
      startWith: "$_id",
      connectFromField: "_id",
      connectToField: "parent_id",
      as: "subcategories",
      maxDepth: 3
    }
  }
])
```

---

## Модуль 5: Репликация и отказоустойчивость (25 минут)

### 🎯 Напоминалка

**Replica Set - основные концепции:**
```
┌─────────────────────────────────────────┐
│           Replica Set                   │
├─────────────────────────────────────────┤
│                                         │
│  ┌──────────┐  Replication  ┌────────┐ │
│  │ Primary  │ ────────────> │Secondary││
│  │  (r/w)   │               │  (r)   │ │
│  └──────────┘               └────────┘ │
│       │                          │      │
│       │    Election              │      │
│       ▼                          ▼      │
│  ┌────────┐                 ┌────────┐ │
│  │Secondary│                │Secondary││
│  │  (r)   │                │  (r)   │ │
│  └────────┘                └────────┘ │
└─────────────────────────────────────────┘
```

**Настройка Replica Set (Docker Compose):**
```yaml
# docker-compose-replica.yml
version: '3.8'

services:
  mongo1:
    image: mongo:7.0
    container_name: mongo1
    command: mongod --replSet rs0 --port 27017
    ports:
      - "27017:27017"
    volumes:
      - mongo1_data:/data/db
    networks:
      - mongo_cluster

  mongo2:
    image: mongo:7.0
    container_name: mongo2
    command: mongod --replSet rs0 --port 27017
    ports:
      - "27018:27017"
    volumes:
      - mongo2_data:/data/db
    networks:
      - mongo_cluster

  mongo3:
    image: mongo:7.0
    container_name: mongo3
    command: mongod --replSet rs0 --port 27017
    ports:
      - "27019:27017"
    volumes:
      - mongo3_data:/data/db
    networks:
      - mongo_cluster

volumes:
  mongo1_data:
  mongo2_data:
  mongo3_data:

networks:
  mongo_cluster:
    driver: bridge
```

**Инициализация Replica Set:**
```javascript
// Подключись к первому узлу
mongosh --port 27017

// Инициализируй replica set
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo1:27017" },
    { _id: 1, host: "mongo2:27017" },
    { _id: 2, host: "mongo3:27017" }
  ]
})

// Проверь статус
rs.status()

// Конфигурация
rs.conf()

// Проверка синхронизации
rs.printReplicationInfo()
rs.printSecondaryReplicationInfo()
```

**Управление Replica Set:**
```javascript
// Добавить узел
rs.add("mongo4:27017")

// Добавить arbiter (голосующий узел без данных)
rs.addArb("mongo-arbiter:27017")

// Удалить узел
rs.remove("mongo4:27017")

// Изменить приоритет узла
cfg = rs.conf()
cfg.members[1].priority = 2  // Выше приоритет для выборов
rs.reconfig(cfg)

// Пошаговая смена Primary
rs.stepDown()

// Заморозить Secondary (не участвует в выборах)
rs.freeze(120)  // 120 секунд
```

**Read Preferences:**
```javascript
// Чтение с Primary (по умолчанию)
db.users.find().readPref("primary")

// Чтение с Primary или Secondary
db.users.find().readPref("primaryPreferred")

// Чтение только с Secondary
db.users.find().readPref("secondary")

// Чтение с Secondary или Primary
db.users.find().readPref("secondaryPreferred")

// Чтение с ближайшего узла
db.users.find().readPref("nearest")

// В URI
mongodb://host1,host2,host3/?replicaSet=rs0&readPreference=secondaryPreferred
```

**Write Concerns:**
```javascript
// Write concern - гарантии записи
db.users.insertOne(
  { name: "John" },
  { writeConcern: { w: "majority", j: true, wtimeout: 5000 } }
)

// w: "majority" - большинство узлов подтвердило
// j: true - записано в journal
// wtimeout: время ожидания

// Уровни w:
// w: 0 - не ждать подтверждения
// w: 1 - подтверждение от primary
// w: 2 - подтверждение от primary + 1 secondary
// w: "majority" - от большинства узлов
```

**Мониторинг репликации:**
```javascript
// Статус репликации
rs.status()

// Задержка репликации
rs.printSecondaryReplicationInfo()

// Является ли узел Primary
db.isMaster()

// Oplog информация
db.getReplicationInfo()

// Размер oplog
db.oplog.rs.stats()
```

### 💻 Задание

Настрой Replica Set:

1. Запусти Replica Set через Docker Compose:
   ```bash
   # Создай docker-compose-replica.yml (см. выше)
   docker-compose -f docker-compose-replica.yml up -d
   
   # Подожди 10-15 секунд для запуска
   sleep 15
   ```

2. Инициализируй Replica Set:
   ```bash
   mongosh --port 27017
   ```
   
   ```javascript
   rs.initiate({
     _id: "rs0",
     members: [
       { _id: 0, host: "localhost:27017" },
       { _id: 1, host: "localhost:27018" },
       { _id: 2, host: "localhost:27019" }
     ]
   })
   
   // Подожди ~30 секунд
   // Проверь статус
   rs.status()
   ```

3. Протестируй репликацию:
   ```javascript
   // На Primary создай данные
   use testdb
   db.replication_test.insertOne({ 
     message: "Hello from Primary",
     timestamp: new Date()
   })
   
   // Подключись к Secondary
   // mongosh --port 27018
   // Включи чтение с Secondary
   db.getMongo().setReadPref("secondary")
   
   use testdb
   db.replication_test.find()
   // Должен увидеть данные
   ```

4. Протестируй отказоустойчивость:
   ```bash
   # Останови Primary
   docker stop mongo1
   
   # Подключись к другому узлу
   mongosh --port 27018
   
   # Проверь, что новый Primary выбран
   rs.status()
   
   # Запиши данные
   db.replication_test.insertOne({ 
     message: "New Primary is working",
     timestamp: new Date()
   })
   
   # Верни первый узел
   docker start mongo1
   
   # Он станет Secondary и синхронизирует данные
   ```

### 🚀 Бонус (новое)

**Настрой Read Concern для строгой консистентности:**
```javascript
// Linearizable read - строжайшая консистентность
db.users.find({ _id: 123 }).readConcern("linearizable")

// Majority - прочитано большинством узлов
db.users.find().readConcern("majority")

// Комбинация write и read concern
db.users.insertOne(
  { name: "John" },
  { 
    writeConcern: { w: "majority" },
    readConcern: { level: "majority" }
  }
)
```

**Используй Casual Consistency для сессий:**
```javascript
// Создай сессию с causal consistency
const session = db.getMongo().startSession({
  causalConsistency: true
})

const sessionDB = session.getDatabase("testdb")

// Запись
sessionDB.users.insertOne({ name: "Alice" })

// Чтение увидит запись даже на Secondary
sessionDB.users.findOne({ name: "Alice" })

session.endSession()
```

---

## Модуль 6: Шардинг и масштабирование (30 минут)

### 🎯 Напоминалка

**Sharding - горизонтальное масштабирование:**
```
┌──────────────────────────────────────────────┐
│          Sharded Cluster                     │
├──────────────────────────────────────────────┤
│                                              │
│  ┌──────────┐  Query    ┌──────────┐        │
│  │  mongos  │ ◄────────►│  mongos  │        │
│  │ (Router) │           │ (Router) │        │
│  └─────┬────┘           └────┬─────┘        │
│        │                     │               │
│        │    ┌───────────────┐│               │
│        └───►│ Config Server ││◄──────────────┘
│             │  Replica Set  │                │
│             └───────────────┘                │
│                     │                        │
│        ┌────────────┼────────────┐           │
│        ▼            ▼            ▼           │
│   ┌────────┐   ┌────────┐   ┌────────┐      │
│   │ Shard1 │   │ Shard2 │   │ Shard3 │      │
│   │  RS1   │   │  RS2   │   │  RS3   │      │
│   └────────┘   └────────┘   └────────┘      │
└──────────────────────────────────────────────┘
```

**Настройка Sharding (базовая):**
```bash
# 1. Config Server Replica Set
mongod --configsvr --replSet configRS --port 27019 --dbpath /data/configdb

# 2. Shard Replica Sets
mongod --shardsvr --replSet shard1RS --port 27018 --dbpath /data/shard1
mongod --shardsvr --replSet shard2RS --port 27028 --dbpath /data/shard2

# 3. mongos Router
mongos --configdb configRS/localhost:27019 --port 27017
```

**Инициализация шардинга:**
```javascript
// Подключись к mongos
mongosh --port 27017

// Добавь шарды
sh.addShard("shard1RS/localhost:27018")
sh.addShard("shard2RS/localhost:27028")

// Включи шардинг для БД
sh.enableSharding("mydb")

// Создай индекс для shard key
db.users.createIndex({ user_id: 1 })

// Зашардируй коллекцию
sh.shardCollection("mydb.users", { user_id: 1 })

// Статус шардинга
sh.status()
```

**Shard Key - ключ шардирования:**
```javascript
// Ranged sharding (по диапазонам)
sh.shardCollection("mydb.users", { age: 1 })

// Hashed sharding (хэш, равномерное распределение)
sh.shardCollection("mydb.logs", { _id: "hashed" })

// Compound shard key
sh.shardCollection("mydb.orders", { customer_id: 1, order_date: 1 })

// Zone sharding (географическое распределение)
sh.addShardToZone("shard1RS", "US")
sh.addShardToZone("shard2RS", "EU")

sh.updateZoneKeyRange(
  "mydb.users",
  { country: "US" },
  { country: "US\uffff" },
  "US"
)
```

**Управление chunks:**
```javascript
// Информация о chunks
db.printShardingStatus()

// Ручное разделение chunk
sh.splitAt("mydb.users", { user_id: 10000 })

// Баланс chunks
sh.enableBalancing("mydb.users")
sh.disableBalancing("mydb.users")

// Статус балансера
sh.getBalancerState()
sh.isBalancerRunning()

// Настройка окна балансировки
db.settings.update(
  { _id: "balancer" },
  { $set: { activeWindow: { start: "23:00", stop: "06:00" } } },
  { upsert: true }
)
```

**Мониторинг шардинга:**
```javascript
// Общий статус
sh.status()

// Распределение данных
db.users.getShardDistribution()

// Статистика по шардам
db.stats()

// Какие коллекции зашардированы
use config
db.collections.find()

// Информация о chunks
db.chunks.find({ ns: "mydb.users" }).count()
```

### 💻 Задание

Симуляция шардированного кластера:

1. Создай упрощенную конфигурацию (для практики):
   ```bash
   # В production используй replica sets для каждого компонента!
   # Это упрощенная версия для обучения
   
   # Создай директории
   mkdir -p /tmp/mongodb/{config,shard1,shard2}
   
   # Config Server
   mongod --configsvr --replSet configRS \
     --port 27019 --dbpath /tmp/mongodb/config \
     --logpath /tmp/mongodb/config.log --fork
   
   # Инициализируй config RS
   mongosh --port 27019
   ```
   
   ```javascript
   rs.initiate({
     _id: "configRS",
     configsvr: true,
     members: [{ _id: 0, host: "localhost:27019" }]
   })
   exit
   ```

2. Запусти шарды:
   ```bash
   # Shard 1
   mongod --shardsvr --replSet shard1RS \
     --port 27018 --dbpath /tmp/mongodb/shard1 \
     --logpath /tmp/mongodb/shard1.log --fork
   
   # Shard 2
   mongod --shardsvr --replSet shard2RS \
     --port 27028 --dbpath /tmp/mongodb/shard2 \
     --logpath /tmp/mongodb/shard2.log --fork
   
   # Инициализируй shard RS
   mongosh --port 27018
   ```
   
   ```javascript
   rs.initiate({
     _id: "shard1RS",
     members: [{ _id: 0, host: "localhost:27018" }]
   })
   exit
   ```
   
   ```bash
   mongosh --port 27028
   ```
   
   ```javascript
   rs.initiate({
     _id: "shard2RS",
     members: [{ _id: 0, host: "localhost:27028" }]
   })
   exit
   ```

3. Запусти mongos router:
   ```bash
   mongos --configdb configRS/localhost:27019 \
     --port 27017 --logpath /tmp/mongodb/mongos.log --fork
   ```

4. Настрой шардинг:
   ```bash
   mongosh --port 27017
   ```
   
   ```javascript
   // Добавь шарды
   sh.addShard("shard1RS/localhost:27018")
   sh.addShard("shard2RS/localhost:27028")
   
   // Проверь статус
   sh.status()
   
   // Включи шардинг для БД
   sh.enableSharding("testdb")
   
   // Создай коллекцию с данными
   use testdb
   for(let i = 0; i < 10000; i++) {
     db.users.insertOne({
       user_id: i,
       name: "User " + i,
       created_at: new Date()
     })
   }
   
   // Создай индекс
   db.users.createIndex({ user_id: 1 })
   
   // Зашардируй коллекцию (hashed для равномерного распределения)
   sh.shardCollection("testdb.users", { user_id: "hashed" })
   
   // Проверь распределение
   db.users.getShardDistribution()
   ```

### 🚀 Бонус (новое)

**Используй Zone Sharding для географического распределения:**
```javascript
// Создай зоны
sh.addShardToZone("shard1RS", "US_EAST")
sh.addShardToZone("shard2RS", "EU_WEST")

// Определи диапазоны для зон
sh.updateZoneKeyRange(
  "testdb.users",
  { user_id: MinKey },
  { user_id: 5000 },
  "US_EAST"
)

sh.updateZoneKeyRange(
  "testdb.users",
  { user_id: 5000 },
  { user_id: MaxKey },
  "EU_WEST"
)

// Проверь зоны
sh.status()
```

**Настрой Jumbo Chunks обработку:**
```javascript
// Для больших chunks, которые не балансируются
db.adminCommand({
  splitChunk: "testdb.users",
  keyPattern: { user_id: 1 },
  bounds: [{ user_id: MinKey }, { user_id: 5000 }]
})

// Принудительная миграция chunk
db.adminCommand({
  moveChunk: "testdb.users",
  find: { user_id: 1000 },
  to: "shard2RS"
})
```

---

## Модуль 7: Безопасность (25 минут)

### 🎯 Напоминалка

**Аутентификация и авторизация:**
```javascript
// Создай администратора
use admin
db.createUser({
  user: "admin",
  pwd: "SecurePassword123!",
  roles: [{ role: "userAdminAnyDatabase", db: "admin" }]
})

// Создай пользователя БД
use myapp
db.createUser({
  user: "app_user",
  pwd: "AppPassword123!",
  roles: [
    { role: "readWrite", db: "myapp" },
    { role: "read", db: "analytics" }
  ]
})

// Встроенные роли
// read - чтение
// readWrite - чтение и запись
// dbAdmin - администрирование БД
// userAdmin - управление пользователями
// clusterAdmin - администрирование кластера
// root - полный доступ

// Управление пользователями
db.getUsers()
db.getUser("app_user")
db.dropUser("app_user")

// Изменить пароль
db.changeUserPassword("app_user", "NewPassword123!")

// Добавить роль
db.grantRolesToUser("app_user", [{ role: "dbAdmin", db: "myapp" }])

// Удалить роль
db.revokeRolesFromUser("app_user", [{ role: "dbAdmin", db: "myapp" }])
```

**Кастомные роли:**
```javascript
// Создай кастомную роль
use admin
db.createRole({
  role: "readOnlyLogs",
  privileges: [
    {
      resource: { db: "logs", collection: "" },
      actions: ["find", "listCollections"]
    }
  ],
  roles: []
})

// Роль с ограниченным доступом
db.createRole({
  role: "reportUser",
  privileges: [
    {
      resource: { db: "analytics", collection: "reports" },
      actions: ["find", "aggregate"]
    }
  ],
  roles: []
})

// Назначь роль пользователю
db.grantRolesToUser("analyst", ["reportUser"])
```

**Включение аутентификации:**
```yaml
# В mongod.conf
security:
  authorization: enabled
  
# Или при запуске
mongod --auth --dbpath /data/db
```

**TLS/SSL шифрование:**
```yaml
# mongod.conf
net:
  ssl:
    mode: requireSSL
    PEMKeyFile: /path/to/mongodb.pem
    CAFile: /path/to/ca.pem
    allowConnectionsWithoutCertificates: false
```

```bash
# Подключение с TLS
mongosh "mongodb://host:27017" \
  --tls \
  --tlsCAFile /path/to/ca.pem \
  --tlsCertificateKeyFile /path/to/client.pem
```

**Шифрование данных (Encryption at Rest):**
```yaml
# mongod.conf (только Enterprise)
security:
  enableEncryption: true
  encryptionKeyFile: /path/to/keyfile

# Или с KMIP
security:
  enableEncryption: true
  kmip:
    serverName: kmip.example.com
    port: 5696
    clientCertificateFile: /path/to/client.pem
```

**Аудит (только Enterprise):**
```yaml
# mongod.conf
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.json
  filter: '{ atype: { $in: ["authenticate", "createUser", "dropDatabase"] } }'
```

**IP Whitelisting:**
```yaml
# mongod.conf
net:
  bindIp: 127.0.0.1,192.168.1.0/24
  
# Или через firewall
# iptables -A INPUT -p tcp --dport 27017 -s 192.168.1.0/24 -j ACCEPT
# iptables -A INPUT -p tcp --dport 27017 -j DROP
```

**Field-Level Encryption (Client-Side):**
```javascript
// Автоматическое шифрование полей
const clientEncryption = new ClientEncryption(keyVaultClient, {
  keyVaultNamespace: 'encryption.__keyVault',
  kmsProviders: {
    local: {
      key: localMasterKey
    }
  }
})

// Создай data key
const dataKeyId = await clientEncryption.createDataKey('local')

// Шифруй данные
const encryptedValue = await clientEncryption.encrypt(
  'sensitive data',
  {
    keyId: dataKeyId,
    algorithm: 'AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic'
  }
)

db.users.insertOne({
  name: 'John',
  ssn: encryptedValue
})
```

### 💻 Задание

Настрой безопасность MongoDB:

1. Создай пользователей с разными ролями:
   ```javascript
   // Подключись без аутентификации
   mongosh
   
   use admin
   
   // Создай root пользователя
   db.createUser({
     user: "dbadmin",
     pwd: "SuperSecure123!",
     roles: [
       { role: "root", db: "admin" }
     ]
   })
   
   // Перезапусти MongoDB с --auth
   // docker restart mongodb (или systemctl restart mongod)
   
   // Подключись с аутентификацией
   mongosh -u dbadmin -p "SuperSecure123!" --authenticationDatabase admin
   
   // Создай пользователя приложения
   use myapp
   db.createUser({
     user: "app_user",
     pwd: "AppPassword123!",
     roles: [
       { role: "readWrite", db: "myapp" }
     ]
   })
   
   // Создай пользователя только для чтения
   db.createUser({
     user: "readonly_user",
     pwd: "ReadPassword123!",
     roles: [
       { role: "read", db: "myapp" }
     ]
   })
   
   // Создай аналитика
   db.createUser({
     user: "analyst",
     pwd: "AnalystPassword123!",
     roles: [
       { role: "read", db: "myapp" },
       { role: "read", db: "analytics" }
     ]
   })
   ```

2. Протестируй разграничение доступа:
   ```bash
   # Подключись как app_user
   mongosh -u app_user -p "AppPassword123!" myapp
   ```
   
   ```javascript
   // Должно работать
   db.test.insertOne({ message: "Hello" })
   db.test.find()
   
   // Не должно работать (нет прав на другие БД)
   use admin
   show collections  // Ошибка
   
   exit
   ```
   
   ```bash
   # Подключись как readonly_user
   mongosh -u readonly_user -p "ReadPassword123!" myapp
   ```
   
   ```javascript
   // Должно работать
   db.test.find()
   
   // Не должно работать
   db.test.insertOne({ message: "Try to write" })  // Ошибка
   ```

3. Создай кастомную роль:
   ```bash
   mongosh -u dbadmin -p "SuperSecure123!" --authenticationDatabase admin
   ```
   
   ```javascript
   use admin
   
   // Роль для просмотра только логов
   db.createRole({
     role: "logViewer",
     privileges: [
       {
         resource: { db: "myapp", collection: "logs" },
         actions: ["find", "listIndexes"]
       }
     ],
     roles: []
   })
   
   // Создай пользователя с этой ролью
   use myapp
   db.createUser({
     user: "log_viewer",
     pwd: "LogPassword123!",
     roles: ["logViewer"]
   })
   
   // Тест
   exit
   ```
   
   ```bash
   mongosh -u log_viewer -p "LogPassword123!" myapp
   ```
   
   ```javascript
   // Создай коллекцию logs
   use myapp
   db.createCollection("logs")
   
   // Должно работать
   db.logs.find()
   
   // Не должно работать
   db.test.find()  // Ошибка
   ```

### 🚀 Бонус (новое)

**Настрой Connection String с параметрами безопасности:**
```bash
# Базовый
mongodb://user:pass@host:27017/dbname

# С репликой
mongodb://user:pass@host1:27017,host2:27017,host3:27017/dbname?replicaSet=rs0

# С TLS
mongodb://user:pass@host:27017/dbname?tls=true&tlsCAFile=/path/to/ca.pem

# С опциями безопасности
mongodb://user:pass@host:27017/dbname?authSource=admin&authMechanism=SCRAM-SHA-256

# MongoDB Atlas
mongodb+srv://user:pass@cluster.mongodb.net/dbname?retryWrites=true&w=majority
```

**Используй MongoDB Secrets в Kubernetes:**
```yaml
# mongodb-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
type: Opaque
stringData:
  mongodb-root-password: SuperSecure123!
  mongodb-user: app_user
  mongodb-password: AppPassword123!
  mongodb-database: myapp

---
# mongodb-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:7.0
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          value: admin
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongodb-root-password
        - name: MONGO_INITDB_DATABASE
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongodb-database
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: mongodb-data
          mountPath: /data/db
      volumes:
      - name: mongodb-data
        persistentVolumeClaim:
          claimName: mongodb-pvc
```

---

## Модуль 8: Backup и Recovery (25 минут)

### 🎯 Напоминалка

**Методы бэкапа MongoDB:**

1. **mongodump/mongorestore** - логический бэкап:
```bash
# Полный бэкап всех БД
mongodump --uri="mongodb://admin:pass@localhost:27017" --out=/backup/$(date +%Y%m%d)

# Бэкап одной БД
mongodump --uri="mongodb://admin:pass@localhost:27017" --db=myapp --out=/backup/myapp

# Бэкап одной коллекции
mongodump --uri="mongodb://admin:pass@localhost:27017" \
  --db=myapp --collection=users --out=/backup/users

# С gzip сжатием
mongodump --uri="mongodb://admin:pass@localhost:27017" --gzip --out=/backup/compressed

# Только схема (без данных)
mongodump --uri="mongodb://admin:pass@localhost:27017" --db=myapp --excludeCollection=logs

# Восстановление
mongorestore --uri="mongodb://admin:pass@localhost:27017" /backup/20241221

# Восстановление одной БД
mongorestore --uri="mongodb://admin:pass@localhost:27017" --db=myapp /backup/myapp

# Восстановление с заменой данных
mongorestore --uri="mongodb://admin:pass@localhost:27017" --drop /backup/20241221

# Восстановление в другую БД
mongorestore --uri="mongodb://admin:pass@localhost:27017" \
  --nsFrom="myapp.*" --nsTo="myapp_test.*" /backup/myapp
```

2. **Снапшоты файловой системы** (требует остановки записи):
```bash
# Заморозь записи
mongosh --eval "db.fsyncLock()"

# Создай снапшот (зависит от ФС/облака)
# LVM snapshot
lvcreate -L 10G -s -n mongodb_snap /dev/vg/mongodb

# AWS EBS snapshot
aws ec2 create-snapshot --volume-id vol-xxxxx

# Разморозь записи
mongosh --eval "db.fsyncUnlock()"
```

3. **Continuous Backup (Replica Set Oplog):**
```bash
# Бэкап oplog для Point-in-Time Recovery
mongodump --uri="mongodb://admin:pass@localhost:27017" \
  --oplog --out=/backup/oplog_$(date +%Y%m%d_%H%M%S)

# Восстановление на определенный момент
mongorestore --uri="mongodb://admin:pass@localhost:27017" \
  --oplogReplay --oplogLimit=1640995200:1 /backup/oplog
```

4. **MongoDB Cloud Manager / Ops Manager** (Enterprise):
- Автоматические снапшоты
- Point-in-Time Recovery
- Мониторинг и алерты

**Стратегии бэкапа:**
```bash
#!/bin/bash
# backup-mongodb.sh - Скрипт ежедневного бэкапа

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup/mongodb"
MONGO_URI="mongodb://backup_user:password@localhost:27017"
RETENTION_DAYS=7

# Создай директорию
mkdir -p "$BACKUP_DIR"

# Выполни бэкап
mongodump --uri="$MONGO_URI" --gzip --out="$BACKUP_DIR/$DATE"

# Проверь успешность
if [ $? -eq 0 ]; then
  echo "Backup completed successfully: $BACKUP_DIR/$DATE"
  
  # Загрузи в S3 (опционально)
  aws s3 sync "$BACKUP_DIR/$DATE" "s3://my-mongodb-backups/$DATE/"
  
  # Удали старые бэкапы
  find "$BACKUP_DIR" -type d -mtime +$RETENTION_DAYS -exec rm -rf {} \;
else
  echo "Backup failed!"
  exit 1
fi
```

**Cron задание для автоматического бэкапа:**
```bash
# Добавь в crontab
# crontab -e

# Ежедневно в 2:00
0 2 * * * /scripts/backup-mongodb.sh >> /var/log/mongodb-backup.log 2>&1

# Еженедельно полный бэкап в воскресенье
0 3 * * 0 /scripts/full-backup-mongodb.sh >> /var/log/mongodb-backup.log 2>&1
```

**Export/Import данных:**
```bash
# mongoexport - экспорт в JSON/CSV
mongoexport --uri="mongodb://admin:pass@localhost:27017" \
  --db=myapp --collection=users --out=users.json

# С query фильтром
mongoexport --uri="mongodb://admin:pass@localhost:27017" \
  --db=myapp --collection=users \
  --query='{"age":{"$gte":18}}' --out=adult_users.json

# В CSV формате
mongoexport --uri="mongodb://admin:pass@localhost:27017" \
  --db=myapp --collection=users \
  --type=csv --fields=name,email,age --out=users.csv

# mongoimport - импорт данных
mongoimport --uri="mongodb://admin:pass@localhost:27017" \
  --db=myapp --collection=users --file=users.json

# CSV импорт
mongoimport --uri="mongodb://admin:pass@localhost:27017" \
  --db=myapp --collection=users \
  --type=csv --headerline --file=users.csv

# Режим upsert (обновление или вставка)
mongoimport --uri="mongodb://admin:pass@localhost:27017" \
  --db=myapp --collection=users \
  --mode=upsert --upsertFields=email --file=users.json
```

### 💻 Задание

Настрой систему бэкапов:

1. Создай тестовые данные:
   ```javascript
   mongosh
   
   use backup_test
   
   // Создай данные для бэкапа
   for(let i = 0; i < 1000; i++) {
     db.users.insertOne({
       user_id: i,
       name: "User " + i,
       email: `user${i}@example.com`,
       created_at: new Date()
     })
   }
   
   db.orders.insertMany([
     { order_id: 1, user_id: 1, total: 100, date: new Date() },
     { order_id: 2, user_id: 2, total: 250, date: new Date() }
   ])
   ```

2. Выполни бэкап:
   ```bash
   # Создай директорию для бэкапов
   mkdir -p ~/mongodb_backups
   
   # Полный бэкап
   mongodump --db=backup_test --out=~/mongodb_backups/full_$(date +%Y%m%d)
   
   # Сжатый бэкап
   mongodump --db=backup_test --gzip --out=~/mongodb_backups/compressed_$(date +%Y%m%d)
   
   # Проверь размер
   du -sh ~/mongodb_backups/*
   
   # Бэкап только коллекции users
   mongodump --db=backup_test --collection=users --out=~/mongodb_backups/users_only
   ```

3. Протестируй восстановление:
   ```bash
   # Удали данные
   mongosh
   ```
   
   ```javascript
   use backup_test
   db.dropDatabase()
   show dbs  // backup_test исчезла
   exit
   ```
   
   ```bash
   # Восстанови из бэкапа
   mongorestore --db=backup_test ~/mongodb_backups/full_20241221/backup_test
   
   # Проверь
   mongosh
   ```
   
   ```javascript
   use backup_test
   db.users.countDocuments()  // Должно быть 1000
   db.orders.countDocuments()  // Должно быть 2
   ```

4. Экспорт/импорт данных:
   ```bash
   # Экспорт в JSON
   mongoexport --db=backup_test --collection=users \
     --query='{"user_id":{"$lt":10}}' \
     --out=~/mongodb_backups/first_10_users.json
   
   # Экспорт в CSV
   mongoexport --db=backup_test --collection=users \
     --type=csv --fields=user_id,name,email \
     --out=~/mongodb_backups/users.csv
   
   # Посмотри результат
   head -20 ~/mongodb_backups/users.csv
   
   # Импорт обратно
   mongoimport --db=test_import --collection=users \
     --file=~/mongodb_backups/first_10_users.json
   ```

5. Создай скрипт автоматического бэкапа:
   ```bash
   cat > ~/backup-mongodb.sh <<'EOF'
#!/bin/bash

BACKUP_DIR="$HOME/mongodb_backups"
DATE=$(date +%Y%m%d_%H%M%S)
MONGO_URI="mongodb://localhost:27017"
RETENTION_DAYS=7

echo "Starting backup at $(date)"

# Создай бэкап
mongodump --uri="$MONGO_URI" --gzip --out="$BACKUP_DIR/$DATE"

if [ $? -eq 0 ]; then
  echo "Backup completed: $BACKUP_DIR/$DATE"
  
  # Удали старые бэкапы
  find "$BACKUP_DIR" -type d -mtime +$RETENTION_DAYS -exec rm -rf {} \; 2>/dev/null
  
  echo "Old backups cleaned up (retention: $RETENTION_DAYS days)"
else
  echo "Backup failed!"
  exit 1
fi
EOF

   chmod +x ~/backup-mongodb.sh
   
   # Запусти скрипт
   ~/backup-mongodb.sh
   ```

### 🚀 Бонус (новое)

**Настрой бэкап с загрузкой в S3:**
```bash
#!/bin/bash
# backup-to-s3.sh

BACKUP_DIR="/tmp/mongodb_backup"
DATE=$(date +%Y%m%d_%H%M%S)
S3_BUCKET="s3://my-mongodb-backups"
MONGO_URI="mongodb://backup_user:password@localhost:27017"

# Создай бэкап
mongodump --uri="$MONGO_URI" --gzip --out="$BACKUP_DIR/$DATE"

# Загрузи в S3
aws s3 sync "$BACKUP_DIR/$DATE" "$S3_BUCKET/$DATE/"

# Удали локальный бэкап
rm -rf "$BACKUP_DIR/$DATE"

# Оставь только последние 30 дней в S3
aws s3 ls "$S3_BUCKET/" | while read -r line; do
  backup_date=$(echo $line | awk '{print $2}' | sed 's/\///')
  backup_epoch=$(date -d "${backup_date:0:8}" +%s)
  current_epoch=$(date +%s)
  days_old=$(( ($current_epoch - $backup_epoch) / 86400 ))
  
  if [ $days_old -gt 30 ]; then
    aws s3 rm "$S3_BUCKET/$backup_date" --recursive
    echo "Deleted old backup: $backup_date"
  fi
done
```

**Point-in-Time Recovery с Oplog:**
```bash
# 1. Создай базовый бэкап с oplog
mongodump --uri="mongodb://localhost:27017" \
  --oplog --out=/backup/base

# 2. Периодически бэкапируй oplog
mongodump --uri="mongodb://localhost:27017" \
  -d local -c oplog.rs --out=/backup/oplog_incremental_$(date +%Y%m%d_%H%M%S)

# 3. Восстановление на определенный момент времени
# Предположим, нужно восстановить на 2024-12-21 15:30:00

# Сначала базовый бэкап
mongorestore --drop /backup/base

# Затем применяем oplog до нужного момента
# timestamp в формате секунды:инкремент
TIMESTAMP=$(date -d "2024-12-21 15:30:00" +%s):1

mongorestore --oplogReplay \
  --oplogLimit=$TIMESTAMP \
  /backup/base/oplog.bson
```

---

## Модуль 9: Мониторинг и troubleshooting (30 минут)

### 🎯 Напоминалка

**Встроенные команды мониторинга:**
```javascript
// Статистика сервера
db.serverStatus()

// Краткая статистика
db.serverStatus().connections
db.serverStatus().opcounters
db.serverStatus().network

// Статистика БД
db.stats()

// Статистика коллекции
db.users.stats()

// Текущие операции
db.currentOp()

// Медленные операции
db.currentOp({ "secs_running": { "$gt": 5 } })

// Убить операцию
db.killOp(12345)

// Статистика индексов
db.users.aggregate([{ $indexStats: {} }])

// Использование памяти
db.serverStatus().mem

// Использование диска
db.stats().dataSize
db.stats().storageSize
```

**Профайлер запросов:**
```javascript
// Включить профайлер (все операции)
db.setProfilingLevel(2)

// Профилировать только медленные (>100ms)
db.setProfilingLevel(1, { slowms: 100 })

// Выключить
db.setProfilingLevel(0)

// Посмотреть медленные запросы
db.system.profile.find().sort({ ts: -1 }).limit(10).pretty()

// Запросы медленнее 500ms
db.system.profile.find({ millis: { $gt: 500 } }).sort({ millis: -1 })

// Группировка по типу операций
db.system.profile.aggregate([
  {
    $group: {
      _id: "$op",
      count: { $sum: 1 },
      avgDuration: { $avg: "$millis" }
    }
  },
  { $sort: { avgDuration: -1 } }
])

// Очистить профайлер
db.system.profile.drop()
```

**Логи MongoDB:**
```bash
# Просмотр логов
tail -f /var/log/mongodb/mongod.log

# С Docker
docker logs -f mongodb

# Фильтр медленных запросов
grep "slow query" /var/log/mongodb/mongod.log

# Последние ошибки
grep "error" /var/log/mongodb/mongod.log | tail -20

# Уровень логирования
db.setLogLevel(1)  # 0-5, где 5 самый детальный

# Логирование определенного компонента
db.setLogLevel(2, "query")
db.setLogLevel(3, "replication")
```

**Метрики для мониторинга:**
```javascript
// Connections
db.serverStatus().connections.current
db.serverStatus().connections.available

// Operations per second
db.serverStatus().opcounters

// Replication lag (для replica set)
rs.printSecondaryReplicationInfo()

// Memory usage
db.serverStatus().mem.resident
db.serverStatus().mem.virtual

// Lock statistics
db.serverStatus().locks

// Page faults
db.serverStatus().extra_info.page_faults

// WiredTiger cache
db.serverStatus().wiredTiger.cache["bytes currently in the cache"]
db.serverStatus().wiredTiger.cache["maximum bytes configured"]
```

**MongoDB Monitoring Tools:**

1. **MongoDB Compass** - GUI с мониторингом
2. **MongoDB Atlas** - облачный мониторинг
3. **mongostat** - консольная статистика:
```bash
# Статистика в реальном времени
mongostat --uri="mongodb://localhost:27017" 5  # каждые 5 секунд

# С аутентификацией
mongostat --uri="mongodb://admin:pass@localhost:27017" --authenticationDatabase admin
```

4. **mongotop** - активность по коллекциям:
```bash
# Топ активных коллекций
mongotop --uri="mongodb://localhost:27017" 5

# С сортировкой
mongotop --uri="mongodb://localhost:27017" --locks
```

5. **Prometheus + Grafana**:
```yaml
# mongodb-exporter для Prometheus
docker run -d \
  -p 9216:9216 \
  percona/mongodb_exporter:0.40 \
  --mongodb.uri=mongodb://exporter:password@mongodb:27017
```

**Типичные проблемы и решения:**

1. **Медленные запросы:**
```javascript
// Найти проблемные запросы
db.system.profile.find({ millis: { $gt: 1000 } }).sort({ millis: -1 }).limit(5)

// Проверить индексы
db.users.find({ email: "test@example.com" }).explain("executionStats")

// Создать индекс
db.users.createIndex({ email: 1 })
```

2. **Высокое использование памяти:**
```javascript
// Проверить размер кэша
db.serverStatus().wiredTiger.cache

// Ограничить кэш в конфиге
// storage.wiredTiger.engineConfig.cacheSizeGB: 2
```

3. **Проблемы с репликацией:**
```javascript
// Проверить статус
rs.status()

// Проверить задержку
rs.printSecondaryReplicationInfo()

// Проверить oplog
db.getReplicationInfo()
```

4. **Исчерпание connections:**
```javascript
// Текущие подключения
db.serverStatus().connections

// Убить idle подключения
db.currentOp({ "active": false, "secs_running": { "$gt": 300 } }).inprog.forEach(function(op) {
  if(op.op == "query" || op.op == "getmore") {
    db.killOp(op.opid);
  }
})
```

### 💻 Задание

Настрой мониторинг и найди проблемы:

1. Создай проблемную коллекцию:
   ```javascript
   mongosh
   
   use performance_test
   
   // Создай большую коллекцию без индексов
   for(let i = 0; i < 50000; i++) {
     db.slow_collection.insertOne({
       user_id: i,
       email: `user${i}@example.com`,
       data: "x".repeat(1000),  // 1KB текста
       created_at: new Date()
     })
   }
   ```

2. Включи профайлер:
   ```javascript
   // Профилировать запросы медленнее 50ms
   db.setProfilingLevel(1, { slowms: 50 })
   
   // Выполни медленные запросы
   db.slow_collection.find({ email: "user25000@example.com" })
   db.slow_collection.find({ user_id: { $gt: 40000 } })
   db.slow_collection.find({ created_at: { $gte: new Date("2024-01-01") } })
   
   // Посмотри профайл
   db.system.profile.find().sort({ ts: -1 }).limit(5).pretty()
   
   // Найди самые медленные
   db.system.profile.find().sort({ millis: -1 }).limit(5)
   ```

3. Проанализируй запросы:
   ```javascript
   // Детальный анализ
   db.slow_collection.find({ email: "user25000@example.com" }).explain("executionStats")
   
   // Обрати внимание на:
   // - executionTimeMillis
   // - totalDocsExamined (должно быть близко к nReturned)
   // - stage: "COLLSCAN" (плохо) vs "IXSCAN" (хорошо)
   
   // Создай индекс
   db.slow_collection.createIndex({ email: 1 })
   
   // Проверь снова
   db.slow_collection.find({ email: "user25000@example.com" }).explain("executionStats")
   
   // Сравни время выполнения
   ```

4. Используй mongostat:
   ```bash
   # В другом терминале
   mongostat --uri="mongodb://localhost:27017" 2
   
   # Наблюдай за:
   # - insert/query/update/delete - операции в секунду
   # - dirty/used - использование WiredTiger cache
   # - res - resident memory
   # - qr|qw - очередь на чтение/запись
   ```

5. Используй mongotop:
   ```bash
   # Топ активных коллекций
   mongotop --uri="mongodb://localhost:27017" 5
   
   # Выполни запросы и посмотри активность
   ```

6. Диагностика текущих операций:
   ```javascript
   // Текущие операции
   db.currentOp()
   
   // Только активные
   db.currentOp({ "active": true })
   
   // Медленные операции
   db.currentOp({ "secs_running": { "$gt": 3 } })
   
   // Операции на определенной коллекции
   db.currentOp({ "ns": "performance_test.slow_collection" })
   ```

### 🚀 Бонус (новое)

**Настрой Prometheus + Grafana для MongoDB:**
```yaml
# docker-compose-monitoring.yml
version: '3.8'

services:
  mongodb-exporter:
    image: percona/mongodb_exporter:0.40
    container_name: mongodb-exporter
    command:
      - '--mongodb.uri=mongodb://admin:password123@mongodb:27017'
      - '--collect-all'
    ports:
      - "9216:9216"
    depends_on:
      - mongodb

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - prometheus

volumes:
  prometheus_data:
  grafana_data:
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'mongodb'
    static_configs:
      - targets: ['mongodb-exporter:9216']
```

**Создай алерты для критических метрик:**
```yaml
# alerts.yml
groups:
  - name: mongodb_alerts
    interval: 30s
    rules:
      - alert: MongoDBDown
        expr: mongodb_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MongoDB instance is down"
          
      - alert: MongoDBHighConnections
        expr: mongodb_connections{state="current"} > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MongoDB has too many connections"
          
      - alert: MongoDBReplicationLag
        expr: mongodb_mongod_replset_member_replication_lag > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MongoDB replication lag is high"
```

---

## Модуль 10: Best Practices и оптимизация (25 минут)

### 🎯 Напоминалка

**Дизайн схемы данных:**

**1. Embedded Documents (Встраивание):**
```javascript
// Хорошо для One-to-Few отношений
{
  _id: 1,
  name: "John Doe",
  addresses: [
    { street: "123 Main St", city: "New York" },
    { street: "456 Oak Ave", city: "Boston" }
  ]
}

// Плюсы: одна операция чтения, атомарность
// Минусы: документ может расти, 16MB лимит
```

**2. References (Ссылки):**
```javascript
// Хорошо для One-to-Many и Many-to-Many
// Users коллекция
{
  _id: 1,
  name: "John Doe"
}

// Orders коллекция
{
  _id: 101,
  user_id: 1,  // Ссылка
  total: 150
}

// Плюсы: нет дублирования, размер документа контролируется
// Минусы: требуется JOIN ($lookup)
```

**3. Extended Reference (Денормализация):**
```javascript
// Золотая середина - храним часто используемые поля
{
  _id: 101,
  user_id: 1,
  user_name: "John Doe",  // Денормализация
  user_email: "john@example.com",
  total: 150
}

// Плюсы: меньше JOIN'ов
// Минусы: нужно обновлять в нескольких местах
```

**Правила дизайна схемы:**
```
1. Embed когда:
   - Отношение один-к-нескольким (One-to-Few)
   - Данные всегда читаются вместе
   - Вложенные данные не обновляются часто
   
2. Reference когда:
   - Отношение один-ко-многим (One-to-Many)
   - Данные могут расти неограниченно
   - Данные используются независимо
   
3. Денормализуй когда:
   - Производительность критична
   - Данные обновляются редко
   - Допустима eventual consistency
```

**Оптимизация индексов:**
```javascript
// ESR Rule: Equality, Sort, Range
// 1. Equality (=) поля первыми
// 2. Sort поля вторыми
// 3. Range (<, >, !=) поля последними

// Плохо
db.orders.createIndex({ total: 1, customer_id: 1, date: 1 })

// Хорошо (если query: customer_id=X, sort by date, total > Y)
db.orders.createIndex({ customer_id: 1, date: 1, total: 1 })

// Покрывающие индексы (Covered Queries)
db.users.createIndex({ email: 1, name: 1, age: 1 })

db.users.find(
  { email: "john@example.com" },
  { email: 1, name: 1, age: 1, _id: 0 }
)
// Все данные из индекса, не нужно читать документ!

// Избегай
// - Слишком много индексов (замедляют запись)
// - Неиспользуемые индексы
// - Дублирующиеся индексы
```

**Оптимизация запросов:**
```javascript
// ❌ Плохо - выборка всех полей
db.users.find({ age: 25 })

// ✅ Хорошо - проекция нужных полей
db.users.find({ age: 25 }, { name: 1, email: 1 })

// ❌ Плохо - skip на больших числах
db.users.find().skip(100000).limit(10)

// ✅ Хорошо - range query с последним ID
db.users.find({ _id: { $gt: lastSeenId } }).limit(10)

// ❌ Плохо - $where и JavaScript
db.users.find({ $where: "this.age > 25" })

// ✅ Хорошо - нативные операторы
db.users.find({ age: { $gt: 25 } })

// ❌ Плохо - регулярки без якоря
db.users.find({ name: /john/i })

// ✅ Хорошо - с якорем в начале
db.users.find({ name: /^john/i })
db.users.createIndex({ name: 1 })  // Может использовать индекс!

// ❌ Плохо - $ne и $nin не используют индексы эффективно
db.users.find({ status: { $ne: "deleted" } })

// ✅ Хорошо - положительная логика
db.users.find({ status: { $in: ["active", "pending"] } })
```

**Connection Pooling:**
```javascript
// Node.js с правильным пулом подключений
const { MongoClient } = require('mongodb');

const client = new MongoClient(uri, {
  maxPoolSize: 50,        // Максимум подключений
  minPoolSize: 10,        // Минимум подключений
  maxIdleTimeMS: 30000,   // Timeout для idle подключений
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000
});

// Переиспользуй подключение!
let cachedClient = null;

async function connectToDatabase() {
  if (cachedClient) {
    return cachedClient;
  }
  
  cachedClient = await client.connect();
  return cachedClient;
}
```

**Пакетная обработка:**
```javascript
// ❌ Плохо - много отдельных операций
for (let i = 0; i < 1000; i++) {
  await db.users.insertOne({ name: `User ${i}` });
}

// ✅ Хорошо - batch операции
const batch = [];
for (let i = 0; i < 1000; i++) {
  batch.push({ name: `User ${i}` });
  
  if (batch.length >= 100) {
    await db.users.insertMany(batch);
    batch.length = 0;
  }
}
if (batch.length > 0) {
  await db.users.insertMany(batch);
}

// Bulk операции
const bulk = db.users.initializeUnorderedBulkOp();

for (let i = 0; i < 1000; i++) {
  bulk.insert({ name: `User ${i}` });
}

await bulk.execute();
```

**Работа с большими документами:**
```javascript
// GridFS для файлов >16MB
const { GridFSBucket } = require('mongodb');

const bucket = new GridFSBucket(db, {
  bucketName: 'uploads'
});

// Загрузка файла
const uploadStream = bucket.openUploadStream('myfile.pdf');
fs.createReadStream('./myfile.pdf').pipe(uploadStream);

// Скачивание файла
const downloadStream = bucket.openDownloadStreamByName('myfile.pdf');
downloadStream.pipe(fs.createWriteStream('./downloaded.pdf'));

// Удаление файла
const files = await bucket.find({ filename: 'myfile.pdf' }).toArray();
await bucket.delete(files[0]._id);
```

**Настройки производительности:**
```yaml
# mongod.conf
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 2  # 50% RAM рекомендуется
      journalCompressor: snappy
    collectionConfig:
      blockCompressor: snappy

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100

setParameter:
  cursorTimeoutMillis: 600000
  notablescan: 0  # Установи в 1 чтобы блокировать COLLSCAN в production
```

### 💻 Задание

Оптимизируй реальное приложение:

1. Создай неоптимизированную схему:
   ```javascript
   mongosh
   
   use optimization_test
   
   // Плохая схема - все в одной коллекции
   for(let i = 0; i < 10000; i++) {
     db.bad_design.insertOne({
       order_id: i,
       customer_name: `Customer ${i % 100}`,
       customer_email: `customer${i % 100}@example.com`,
       customer_address: `${i} Main St`,
       product_name: `Product ${i % 50}`,
       product_price: Math.floor(Math.random() * 1000),
       product_category: ["Electronics", "Furniture", "Clothing"][i % 3],
       order_date: new Date(),
       quantity: Math.floor(Math.random() * 10) + 1
     })
   }
   ```

2. Найди проблемы:
   ```javascript
   // Включи профайлер
   db.setProfilingLevel(2)
   
   // Типичный запрос
   db.bad_design.find({ 
     customer_email: "customer50@example.com" 
   })
   
   // Анализируй
   db.bad_design.find({ 
     customer_email: "customer50@example.com" 
   }).explain("executionStats")
   
   // Проверь профайл
   db.system.profile.find().sort({ millis: -1 }).limit(5)
   ```

3. Оптимизируй схему:
   ```javascript
   // Правильная нормализация
   
   // Customers
   db.customers.insertMany([...Array(100)].map((_, i) => ({
     _id: i,
     name: `Customer ${i}`,
     email: `customer${i}@example.com`,
     address: `${i} Main St`
   })))
   
   // Products
   db.products.insertMany([...Array(50)].map((_, i) => ({
     _id: i,
     name: `Product ${i}`,
     price: Math.floor(Math.random() * 1000),
     category: ["Electronics", "Furniture", "Clothing"][i % 3]
   })))
   
   // Orders (с денормализацией)
   for(let i = 0; i < 10000; i++) {
     const customerId = i % 100;
     const productId = i % 50;
     
     db.orders.insertOne({
       _id: i,
       customer_id: customerId,
       customer_name: `Customer ${customerId}`,  // Денормализация
       product_id: productId,
       product_name: `Product ${productId}`,     // Денормализация
       product_price: Math.floor(Math.random() * 1000),
       quantity: Math.floor(Math.random() * 10) + 1,
       order_date: new Date()
     })
   }
   ```

4. Создай правильные индексы:
   ```javascript
   // На основе реальных запросов
   db.customers.createIndex({ email: 1 })
   db.products.createIndex({ category: 1, price: 1 })
   db.orders.createIndex({ customer_id: 1, order_date: -1 })
   db.orders.createIndex({ product_id: 1 })
   
   // Покрывающий индекс для частого запроса
   db.orders.createIndex({ 
     customer_id: 1, 
     order_date: -1, 
     product_name: 1, 
     quantity: 1 
   })
   
   // Проверь использование
   db.orders.find(
     { customer_id: 50 },
     { product_name: 1, quantity: 1, order_date: 1, _id: 0 }
   ).sort({ order_date: -1 }).explain("executionStats")
   ```

5. Оптимизируй агрегацию:
   ```javascript
   // ❌ Плохо - без индексов и фильтрации
   db.orders.aggregate([
     { $group: { _id: "$product_id", total: { $sum: "$quantity" } } }
   ])
   
   // ✅ Хорошо - с ранней фильтрацией и индексом
   db.orders.createIndex({ order_date: 1, product_id: 1 })
   
   db.orders.aggregate([
     { 
       $match: { 
         order_date: { $gte: new Date("2024-01-01") } 
       } 
     },
     { 
       $group: { 
         _id: "$product_id", 
         total: { $sum: "$quantity" } 
       } 
     },
     { $sort: { total: -1 } },
     { $limit: 10 }
   ])
   ```

### 🚀 Бонус (новое)

**Используй Change Streams для реактивных приложений:**
```javascript
// Отслеживай изменения в реальном времени
const changeStream = db.orders.watch([
  { $match: { 'fullDocument.total': { $gte: 1000 } } }
]);

changeStream.on('change', (change) => {
  console.log('High value order:', change.fullDocument);
  // Отправь уведомление, обнови кэш и т.д.
});

// С фильтрацией
const pipeline = [
  { $match: { 
    'operationType': { $in: ['insert', 'update'] },
    'fullDocument.status': 'pending'
  }}
];

db.orders.watch(pipeline);
```

**Используй Collation для правильной сортировки:**
```javascript
// Создай коллекцию с collation
db.createCollection("users", {
  collation: { locale: "ru", strength: 2 }
})

// Сортировка с учетом языка
db.users.find().sort({ name: 1 }).collation({ locale: "ru" })

// Индекс с collation
db.users.createIndex(
  { name: 1 },
  { collation: { locale: "ru", strength: 2 } }
)
```

**Используй Capped Collections для логов:**
```javascript
// Коллекция фиксированного размера (FIFO)
db.createCollection("logs", {
  capped: true,
  size: 100000000,  // 100MB
  max: 10000        // Максимум документов
})

// Автоматически удаляются старые документы
db.logs.insertOne({ message: "Log entry", timestamp: new Date() })

// Tailable cursor (как tail -f)
const cursor = db.logs.find().tailable().awaitData();
```

---

## Финальный проект (60 минут)

### Задача: Развернуть production-ready приложение

Создай систему управления заказами интернет-магазина с полным циклом:

**Архитектура:**
- MongoDB Replica Set (3 узла)
- Sharded кластер (опционально)
- Автоматические бэкапы
- Мониторинг (Prometheus + Grafana)
- Безопасность (роли, TLS)

**Требования:**

1. **База данных:**
   - 4 коллекции: customers, products, orders, order_items
   - Оптимизированная схема с денормализацией
   - Индексы для всех частых запросов

2. **Replica Set:**
   - 3 узла (Primary + 2 Secondary)
   - Автоматическое переключение при сбое
   - Read preference для аналитики

3. **Безопасность:**
   - 3 роли: admin, app_user, analyst
   - Разграничение прав доступа
   - Аутентификация включена

4. **Backup:**
   - Автоматический ежедневный бэкап
   - Retention 7 дней
   - Скрипт восстановления

5. **Мониторинг:**
   - Prometheus exporter
   - Grafana дашборд
   - Алерты на критичные метрики

6. **API операции:**
   - Создание заказа (транзакция)
   - Поиск заказов с пагинацией
   - Аналитика продаж (aggregation)
   - Отчеты по клиентам

**Структура проекта:**
```
mongodb-ecommerce/
├── docker-compose.yml
├── scripts/
│   ├── init-replica-set.js
│   ├── create-users.js
│   ├── seed-data.js
│   ├── backup.sh
│   └── restore.sh
├── config/
│   ├── mongod.conf
│   └── prometheus.yml
├── queries/
│   ├── create-order.js
│   ├── analytics.js
│   └── reports.js
└── README.md
```

**Начни с Docker Compose:**
```yaml
version: '3.8'

services:
  mongo1:
    image: mongo:7.0
    container_name: mongo1
    command: mongod --replSet rs0 --port 27017 --auth --keyFile /etc/mongodb-keyfile
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: SecurePassword123!
    ports:
      - "27017:27017"
    volumes:
      - mongo1_data:/data/db
      - ./config/mongodb-keyfile:/etc/mongodb-keyfile
    networks:
      - mongo_cluster

  mongo2:
    image: mongo:7.0
    container_name: mongo2
    command: mongod --replSet rs0 --port 27017 --auth --keyFile /etc/mongodb-keyfile
    ports:
      - "27018:27017"
    volumes:
      - mongo2_data:/data/db
      - ./config/mongodb-keyfile:/etc/mongodb-keyfile
    networks:
      - mongo_cluster

  mongo3:
    image: mongo:7.0
    container_name: mongo3
    command: mongod --replSet rs0 --port 27017 --auth --keyFile /etc/mongodb-keyfile
    ports:
      - "27019:27017"
    volumes:
      - mongo3_data:/data/db
      - ./config/mongodb-keyfile:/etc/mongodb-keyfile
    networks:
      - mongo_cluster

  mongodb-exporter:
    image: percona/mongodb_exporter:0.40
    container_name: mongodb-exporter
    command:
      - '--mongodb.uri=mongodb://admin:SecurePassword123!@mongo1:27017'
      - '--collect-all'
    ports:
      - "9216:9216"
    networks:
      - mongo_cluster
    depends_on:
      - mongo1

volumes:
  mongo1_data:
  mongo2_data:
  mongo3_data:

networks:
  mongo_cluster:
    driver: bridge
```

**Дополнительные улучшения (опционально):**
- CI/CD пайплайн для миграций
- Интеграция с Kubernetes
- Connection pooling оптимизация
- Automated failover тестирование
- Load testing с k6
- Blue-Green deployment стратегия
- Disaster recovery план
- Документация API

---

## Справочная секция: Быстрые шпаргалки

### MongoDB Shell Commands

```javascript
// Базовые операции
show dbs
use mydb
show collections
db.dropDatabase()

// CRUD
db.collection.insertOne({})
db.collection.find({})
db.collection.updateOne({}, { $set: {} })
db.collection.deleteOne({})

// Индексы
db.collection.createIndex({ field: 1 })
db.collection.getIndexes()
db.collection.dropIndex("index_name")

// Aggregation
db.collection.aggregate([
  { $match: {} },
  { $group: {} },
  { $sort: {} }
])

// Администрирование
db.serverStatus()
db.stats()
rs.status()
sh.status()
```

### Операторы запросов

```javascript
// Сравнение
$eq, $ne, $gt, $gte, $lt, $lte, $in, $nin

// Логические
$and, $or, $not, $nor

// Элементы
$exists, $type

// Массивы
$all, $elemMatch, $size

// Обновление
$set, $unset, $inc, $mul, $rename, $push, $pull, $addToSet

// Aggregation
$match, $group, $project, $sort, $limit, $skip, $lookup, $unwind
```

### Connection Strings

```bash
# Базовый
mongodb://localhost:27017/mydb

# С аутентификацией
mongodb://user:pass@localhost:27017/mydb?authSource=admin

# Replica Set
mongodb://host1:27017,host2:27017,host3:27017/mydb?replicaSet=rs0

# Sharded Cluster
mongodb://mongos1:27017,mongos2:27017/mydb

# MongoDB Atlas
mongodb+srv://user:pass@cluster.mongodb.net/mydb
```

### Полезные команды

```bash
# mongodump/mongorestore
mongodump --uri="mongodb://localhost:27017" --out=/backup
mongorestore --uri="mongodb://localhost:27017" /backup

# mongoexport/mongoimport
mongoexport --uri="mongodb://localhost:27017" --db=mydb --collection=users --out=users.json
mongoimport --uri="mongodb://localhost:27017" --db=mydb --collection=users --file=users.json

# mongostat/mongotop
mongostat --uri="mongodb://localhost:27017" 5
mongotop --uri="mongodb://localhost:27017" 5
```

---

## Чек-лист навыков

После прохождения курса ты должен уметь:

### Базовые навыки:
- ✅ Устанавливать и настраивать MongoDB
- ✅ Выполнять CRUD операции уверенно
- ✅ Создавать и использовать индексы
- ✅ Писать aggregation pipelines
- ✅ Работать с mongosh

### Продвинутые навыки:
- ✅ Настраивать Replica Set
- ✅ Проектировать оптимальные схемы данных
- ✅ Создавать сложные агрегации
- ✅ Настраивать безопасность (RBAC)
- ✅ Выполнять бэкапы и восстановление
- ✅ Мониторить производительность

### Expert навыки:
- ✅ Настраивать Sharded Cluster
- ✅ Оптимизировать производительность
- ✅ Troubleshooting сложных проблем
- ✅ Настраивать мониторинг (Prometheus/Grafana)
- ✅ Использовать Transactions
- ✅ Работать с Change Streams

---

## Что нового в последних версиях MongoDB

**MongoDB 7.0 (2023):**
- Queryable Encryption GA
- Улучшенная производительность Time Series
- Compound Wildcard Indexes
- Улучшения Change Streams

**MongoDB 6.0 (2022):**
- Clustered Collections
- Time Series оптимизации
- Encrypted at Rest улучшения

**MongoDB 5.0 (2021):**
- Time Series Collections
- Native Time Series Functions
- Versioned API
- Window Functions в aggregation

**MongoDB 8.0+ (upcoming):**
- Следи за release notes на mongodb.com

---

## Заключение

Поздравляю! Ты прошел курс по освежению знаний MongoDB.

**Следующие шаги:**
1. Практикуйся регулярно - используй MongoDB в своих проектах
2. Автоматизируй backup и monitoring
3. Изучи смежные технологии: Mongoose, Motor, Redis
4. Получи сертификацию MongoDB Professional
5. Делись знаниями - пиши посты, помогай новичкам

**Помни:**
- MongoDB - мощный инструмент, используй правильно
- Начинай с простого, усложняй постепенно
- Документация - твой лучший друг
- Community очень дружелюбное и готово помочь
- Replica Set обязателен для production
- Бэкапы - это святое!

Проходи этот курс каждые 6-12 месяцев, чтобы оставаться в форме!

**Дополнительные ресурсы:**
- **MongoDB University** - бесплатные курсы
- **MongoDB Documentation** - официальная документация
- **MongoDB Blog** - новости и лучшие практики
- **MongoDB Community Forums** - помощь сообщества
- **Stack Overflow** - вопросы и ответы
- **GitHub** - примеры и проекты

Happy MongoDB learning! 🍃🚀
