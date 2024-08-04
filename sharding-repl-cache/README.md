# Схемы

https://disk.yandex.ru/i/rC09R4U166oXeg

# Запуск приложения

```shell
docker compose up -d
```

# Инициализация

Запуск configSrv

```shell
docker compose exec -T configSrv mongosh --port 27000 <<EOF
rs.initiate(
  {
    _id : "config_server",
    configsvr: true,
    members: [
      { _id : 0, host : "configSrv:27000" }
    ]
  }
);
EOF
```

Инициализация shard1, shard2

```shell
docker compose exec -T shard1 mongosh --port 27001 <<EOF
rs.initiate({
  _id: "shard1",
  members: [
    {_id: 0, host: "shard1:27001"},
    {_id: 1, host: "shard1_mongodb1:27002"},
    {_id: 2, host: "shard1_mongodb2:27003"}
  ]
});
EOF
```

```shell
docker compose exec -T shard2 mongosh --port 27004 <<EOF
rs.initiate({
  _id: "shard2",
  members: [
    {_id: 0, host: "shard2:27004"},
    {_id: 1, host: "shard2_mongodb1:27005"},
    {_id: 2, host: "shard2_mongodb2:27006"}
  ]
});
EOF
```

Добавляем шарды

```shell
docker compose exec -T mongos_router mongosh --port 27016 <<EOF
sh.addShard("shard1/shard1:27001,shard1_mongodb1:27002,shard1_mongodb2:27003");
sh.addShard("shard2/shard2:27004,shard2_mongodb1:27005,shard2_mongodb2:27006");
EOF
```

Наполнение данными

```shell
docker compose exec -T mongos_router mongosh --port 27016 <<EOF
sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )
use somedb
for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i})
db.helloDoc.countDocuments()
EOF
```

## Проверка shard1

```shell
docker compose exec -T shard1 mongosh --port 27001 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

## Проверка shard2

```shell
docker compose exec -T shard2 mongosh --port 27004 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

Может быть такое, что количества файлов недостаточно для того чтобы шардирование сработало, стандартный размер чанка 64MB в mongo. Можно попробовать установить другой размер.

```shell
docker compose exec -T mongos_router mongosh --port 27016 <<EOF
use config
db.settings.updateOne(
  { _id: "chunksize" },
  { $set: { value: 1 } },
  { upsert: true }
)
EOF
```

## Проверка реплики shard2_mongodb1

```shell
docker compose exec -T shard2_mongodb1 mongosh --port 27005 <<EOF
use somedb
db.helloDoc.find().limit(5)
EOF
```
