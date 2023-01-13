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

## DB transaction lock & How to handle deadlock in Golang
* Test driven development (or TDD):
We write tests first to make our current code breaks, then we gradually improve the code until the tests pass.

1. at store_test.go, 加上
```go
// check accounts
		fromAccount := result.FromAccount
		require.NotEmpty(t, fromAccount)
		require.Equal(t, account1.ID, fromAccount.ID)

		toAccount := result.ToAccount
		require.NotEmpty(t, toAccount)
		require.Equal(t, account2.ID, toAccount.ID)

		// check accounts' balance
		fmt.Println(">> tx:", fromAccount.Balance, toAccount.Balance)
		diff1 := account1.Balance - fromAccount.Balance
		diff2 := toAccount.Balance - account2.Balance
		require.Equal(t, diff1, diff2)
		require.True(t, diff1 > 0)
		require.True(t, diff1%amount == 0) // 1 * amount, 2 * amount, 3* amount, n* amount

		k := int(diff1 / amount)
		require.True(t, k >= 1 && k <= n)
		require.NotContains(t, existed, k)
		existed[k] = true

	}

	// check the final updated balances
	updatedAccount1, err := testQueries.GetAccount(context.Background(), account1.ID)
	require.NoError(t, err)

	updatedAccount2, err := testQueries.GetAccount(context.Background(), account2.ID)
	require.NoError(t, err)

	fmt.Println(">> after:", updatedAccount1.Balance, updatedAccount2.Balance)
	require.Equal(t, account1.Balance-int64(n)*amount, updatedAccount1.Balance)
	require.Equal(t, account2.Balance+int64(n)*amount, updatedAccount2.Balance)
}

```
會發現require.NotEmpty(t, fromAccount)，會出錯，因為我們還沒有在store.go上面實現balance

2. 在store.go中實現balance功能：
```go=
account1, err := q.GetAccount(ctx, arg.FromAccountID)
if err != nil {
	return err
}

result.FromAccount, err = q.UpdateAccount(ctx, UpdateAccountParams{
	ID:      arg.FromAccountID,
	Balance: account1.Balance - arg.Amount,
})
if err != nil {
	return err
}

account2, err := q.GetAccount(ctx, arg.ToAccountID)
if err != nil {
	return err
}

result.ToAccount, err = q.UpdateAccount(ctx, UpdateAccountParams{
	ID:      arg.ToAccountID,
	Balance: account2.Balance + arg.Amount,
})
if err != nil {
	return err
}
```
但測試是失敗的，兩者的tx會不一樣：

![](https://i.imgur.com/FCTxCel.png)
出錯的點在第94行
![](https://i.imgur.com/m7pc8A9.png)
原因在於account.sql中
```sql
-- name: GetAccount :one
SELECT * FROM accounts
WHERE id = $1 LIMIT 1;
```
這個select並沒有在一個迴圈執行後block起來被人禁用直到下個迴圈，因此兩個concurrent transactions可能得到一樣的值，這證明了為什麼上個迴圈得到146，這個迴圈給出136，而下個迴圈一樣是136

修正方法為，在account.sql, 加上，並make sqlc
```sql
-- name: GetAccountForUpdate :one
SELECT * FROM accounts
WHERE id = $1 LIMIT 1
FOR UPDATE;

```
並在store.go中修改q.GetAccount 為 q.GetAccountForUpdate
```go 
account1, err := q.GetAccount(ctx, arg.FromAccountID)
if err != nil {
	return err
}

result.FromAccount, err = q.UpdateAccount(ctx, UpdateAccountParams{
	ID:      arg.FromAccountID,
	Balance: account1.Balance - arg.Amount,
})
if err != nil {
	return err
}

account2, err := q.GetAccount(ctx, arg.ToAccountID)
if err != nil {
	return err
}
```
但依然得到錯誤，此次錯誤為deadlock：
![](https://i.imgur.com/O71ivng.png)
加上一些標籤後，可以看到deadlock的時機
![](https://i.imgur.com/YCiuFjN.png)

在經過ㄧ些測試後，我們發現deadlock的發生與foreign key有關(影片中有說明)，所以在account.sql中，我們更新NO KEY keyword 在GetAccountForUpdate 中，確保更新過程不會碰到key，並再次make sqlc：
```sql 
-- name: GetAccountForUpdate :one
SELECT * FROM accounts
WHERE id = $1 LIMIT 1
FOR NO KEY UPDATE;
```
這樣處理後就解決問題了
![](https://i.imgur.com/eByjxJR.png)
