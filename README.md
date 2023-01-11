# Note
## Install & use Docker + Postgres + TablePlus to create DB schema
* easy read :https://hackmd.io/@byETi8KzTUyys88QkasF2w/Sk_1uaXqo
1. install postgres in docker, 15-alpine is the postgres version
:::info
docker pull < image>:<tag>
:::
```bash=
docker pull postgres:15-alpine
```
![](https://i.imgur.com/4aVLMFv.png)
    
2. start a container
:::info
docker run --name <container_name> 
-e <environment_variable>
-p <host_ports:container_ports>
        -d
< image>:<tag>
:::

* a container is 1 instance of the application contained in the image, which is started by the docker run command. We can start multiple containers from 1 single images
    
```
docker run --name postgres15 -p 5432:5432 -e POSTGRES_USER=root -e POSTGRES_PASSWORD=secret -d postgres:15-alpine
```

![](https://i.imgur.com/td42dle.png)
    
3. Run command in container
:::info
docker exec -it <container_name_or_id >
    <command> [args]
:::

```
docker exec -it postgres15 psql -U root
```
![](https://i.imgur.com/jyDEitJ.png)

4. View container logs
:::info
docker logs
<container_name_or_id>
:::
see the information logs in the container you created
```
docker logs postgres15
```

5. TablePlus
![](https://i.imgur.com/gCqga9x.png)
press "Create a new connection..."
    
![](https://i.imgur.com/Pba3rpc.png)
* password = secret, test passed
![](https://i.imgur.com/gD9QoGP.png)
    
* open simple_bank.sql we created at dbdiagram in tablePlus and press [command + enter]
![](https://i.imgur.com/O8BC87B.png)
* it creates the tables
![](https://i.imgur.com/kYI1aD3.png)

## How to write & run database migration in Golang
###### tags: `golang`

1. install migrate

```bash=
brew install golang-migrate
```
![](https://i.imgur.com/NO2vbFZ.png)

2. init and create schema
```
migrate create -ext sql -dir db/migration -seq init_schema
```

![](https://i.imgur.com/oAILhiQ.png)
* The up-script is run to make a forward change to the schema
* The down-script is run if we want to revert the change by the up-script


3. put the sql code into 
* up schema

```
CREATE TABLE "accounts" (
  "id" bigserial PRIMARY KEY,
  "owner" varchar NOT NULL,
  "balance" bigint NOT NULL,
  "currency" varchar NOT NULL,
  "created_at" timestamptz NOT NULL DEFAULT (now())
);

CREATE TABLE "entries" (
  "id" bigserial PRIMARY KEY,
  "account_id" bigint NOT NULL,
  "amount" bigint NOT NULL,
  "created_at" timestamptz NOT NULL DEFAULT (now())
);

CREATE TABLE "transfers" (
  "id" bigserial PRIMARY KEY,
  "from_account_id" bigint NOT NULL,
  "to_account_id" bigint NOT NULL,
  "amount" bigint NOT NULL,
  "created_at" timestamptz NOT NULL DEFAULT (now())
);

ALTER TABLE "entries" ADD FOREIGN KEY ("account_id") REFERENCES "accounts" ("id");

ALTER TABLE "transfers" ADD FOREIGN KEY ("from_account_id") REFERENCES "accounts" ("id");

ALTER TABLE "transfers" ADD FOREIGN KEY ("to_account_id") REFERENCES "accounts" ("id");

CREATE INDEX ON "accounts" ("owner");

CREATE INDEX ON "entries" ("account_id");

CREATE INDEX ON "transfers" ("from_account_id");

CREATE INDEX ON "transfers" ("to_account_id");

CREATE INDEX ON "transfers" ("from_account_id", "to_account_id");

COMMENT ON COLUMN "entries"."amount" IS 'can be negative or positive';

COMMENT ON COLUMN "transfers"."amount" IS 'must be positive';
```

* down schema:
```sql=
DROP TABLE IF EXISTS entries;
DROP TABLE IF EXISTS transfers;
DROP TABLE IF EXISTS accounts;
```

4. make migrate
**confirm the docker is running**
![](https://i.imgur.com/jxZfvkz.png)

* in Makefile 
```
postgres:
	docker run --name postgres15 -p 5432:5432 -e POSTGRES_USER=root -e POSTGRES_PASSWORD=secret -d postgres:15-alpine

createdb:
	docker exec -it postgres15 createdb --username=root --owner=root simple_bank

dropdb:
	docker exec -it postgres15 dropdb simple_bank
    
migrateup:
	migrate -path db/migration -database "postgresql://root:secret@localhost:5432/simple_bank?sslmode=disable" -verbose up

migratedown:
	migrate -path db/migration -database "postgresql://root:secret@localhost:5432/simple_bank?sslmode=disable" -verbose down

.PHONY: postgres createdb dorpdb migrateup migratedown
```
打上
```
make createdb
```
就可以創資料庫
![](https://i.imgur.com/J9O4v17.png)
stop and remove the container
![](https://i.imgur.com/EM93Gra.png)

type: make postgres 就可以再次創立container

* use make migrateup/make migratedown to create/delete the tables in database simple_bank
![](https://i.imgur.com/NcbkHMP.png)

## Generate CRUD Golang code from SQL | Compare db/sql, gorm, sqlx & sqlc
###### tags: `golang`

### What is CRUD
![](https://i.imgur.com/UAVStpZ.jpg)

### Usage
![](https://i.imgur.com/WPtltEP.jpg)

1. use sqlc:
* install sqlc
```
brew install sqlc
```
![](https://i.imgur.com/tHO2W2K.png)


* run "sqlc init" at project file
![](https://i.imgur.com/mIKfKGn.png)
* create sqlc, query folders in db folder and write below in the sqlc.yaml
```
version: "1"
packages:
  - name: "db"
    path: "./db/sqlc"
    queries: "./db/query/"
    schema: "./db/migration/"
    engine: "postgresql"
    emit_json_tags: true
    emit_prepared_queries: false
    emit_interface: false
    emit_exact_table_names: false
```
* Add "sqlc: sqlc generate" in makefile
* After create account.sql in query folder and run "make sqlc", you will see the generated files in the sqlc folder:
![](https://i.imgur.com/gaeYzRJ.png)


2. init go
```
go mod init simple_bank
go mod tidy
```

## Write unit tests for database CRUD with random data in Golang

* install golang lib pq
```
go get github.com/lib/pq
```
at main_test.go, add this line in import:
```
_ "github.com/lib/pq"
```

* install testify

```
go get github.com/stretchr/testify
```

1. create main_test.go
```
package db

import (
	"database/sql"
	"log"
	"os"
	"testing"

	_ "github.com/lib/pq"
)

const (
	dbDriver = "postgres"
	dbSource = "postgresql://root:secret@localhost:5432/simple_bank?sslmode=disable"
)

var testQueries *Queries

func TestMain(m *testing.M) {
	conn, err := sql.Open(dbDriver, dbSource)
	if err != nil {
		log.Fatal("cannot connect to db", err)
	}

	// New defined in the db.go
	testQueries = New(conn)
	os.Exit(m.Run())
}
```

2. test account.go -> account_test.go
3. create util folder -> util/random.go

## A clean way to implement database transaction in Golang

### What is a db transaction?
![](https://i.imgur.com/xjYOQ6q.png)
#### example
![](https://i.imgur.com/uxiBINU.jpg)

### Why do we need db transaction
![](https://i.imgur.com/svJrPoF.png)

### ACID PROPERTY
![](https://i.imgur.com/wVtaldt.png)



### 創立store.go和store_test.go來處理上面example的操作

#### 因為在store_test.go有用到Goroutine 與 Channel，這個連結是介紹：
https://peterhpchen.github.io/2020/03/08/goroutine-and-channel.html

### 可以看到transfer db中從114帳戶轉到115帳戶的同時，114帳戶在entries中 -10, 而115帳戶在entries中 +10
* transfer
![](https://i.imgur.com/tyYL96A.png)
* entries
![](https://i.imgur.com/64k9NEK.png)
