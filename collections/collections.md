# Проектирование схем коллекций для шардирования данных

## Описание коллекций

### Orders

#### Коллекция

    {
      _id: ObjectId,
      user_id: ObjectId,
      order_date: Date,
      items: [ // список товаров
        {
          product_id: ObjectId,
          product_name: String,
          price: Number,
          quantity: Number,
          category: String
        }
      ],
      status: String, // статус заказа
      total_amount: Number, // общая сумма
      geozone: String,
      created_at: Date,
      updated_at: Date
    }

#### Шардирование

*Стратегия:* Ranged Sharding

        sh.shardCollection("somedb.orders", { "user_id": 1, "order_date": -1, "geozone": 1})

*Обоснование:*

- локализация данных пользователя на одном шарде (быстрее создавать/изменять заказ, т.к. данные пользователя физически рядом);
- проще искать заказы пользователя (т.к. данные пользоватля физически на одном сервере);
- заказы из одного региона находятся на одном и том же шарде.

Создание разных географических регионов:

        sh.addShardToZone("shard1rs/shard1-1:27017", "moscow")
        sh.addShardToZone("shard1rs/shard2-1:27017", "peter")
        sh.addShardToZone("shard1rs/shard3-1:27017", "ekb")
        

Привязываем данные из одного региона к одному и тому же шарду:

    sh.updateZoneKeyRange(
      "somedb.orders",
      {geozone: "moscow", user_id: MinKey},
      {geozone: "moscow", user_id: MaxKey}, 
      "moscow"
    )

    sh.updateZoneKeyRange(
      "somedb.orders",
      {geozone: "peter", user_id: MinKey},
      {geozone: "peter", user_id: MaxKey},
      "peter"
    )
    sh.updateZoneKeyRange(
      "somedb.orders",
      {geozone: "ekb", user_id: MinKey}, 
      {geozone: "ekb", user_id: MaxKey},
      "ekb"
    )

### Products

    {
      _id: ObjectId,
      name: String,
      category: String,
      price: Number,
      
      stock: { // остатки по геозонам
        "moscow": Number,
        "peter": Number,
        "ekb": Number
      },
      
      attributes: { // дополнительные атрибуты
        color: String,
        size: String,
        brand: String
      },
      
      created_at: Date,
      updated_at: Date
    }


#### Шардирование

*Стратегия:* Hashed Sharding

    sh.shardCollection("somedb.products", { "category": "hashed" })

*Обоснование:*

- Поиск быстрее т.к. товары одной категории на одном и том же шарде;
- Эффективная фильтрация по категориям и ценам.


### Carts

    {
      _id: ObjectId,
      user_id: ObjectId,
      session_id: String,
      items: [
        {
          product_id: ObjectId,
          quantity: Number,
          added_at: Date
        }
      ],
      status: String, // "active" | "ordered" | "abandoned"
      created_at: Date,
      updated_at: Date,
      expires_at: Date
    }


Автоматическая очистка корзин старше 30 дней:

    db.carts.createIndex(
      { "updated_at": 1 }, 
      { 
        expireAfterSeconds: 2592000, // 30 дней
        partialFilterExpression: { "status": { $in: ["abandoned", "ordered"] } }
      }
    )

#### Шардирование

*Стратегия:* Hashed Sharding

    sh.shardCollection("somedb.orders", { "user_id": 1, "order_date": -1 })
    sh.shardCollection("somedb.carts", { "session_id": "hashed" })

*Обоснование*:

- Равномерное распределение корзин по шардам;
- Быстрый доступ к активным корзинам по session_id/user_id.

