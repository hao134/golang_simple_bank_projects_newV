# Note
## Install & use Docker + Postgres + TablePlus to create DB schema
* easy read :https://hackmd.io/@byETi8KzTUyys88QkasF2w/Sk_1uaXqo
1. install postgres in docker, 15-alpine is the postgres version
:::info
docker pull < image>:<tag>
:::
```
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

```
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

```sql
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
```sql
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
```yaml
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
```go
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

## How to avoid deadlock in DB transaction? Queries order matters!
* see deadlock example

type this on terminal
```
docker exec -it postgres15 psql -U root -d simple_bank
```
在兩個終端視窗執行這些指令：
![](https://i.imgur.com/rKu14l3.png)
![](https://i.imgur.com/fxDGtPV.png)

1. 對第一個視窗進行第一個指令的update:
```sql
UPDATE accounts SET balance = balance - 10 WHERE id = 1 RETURNING *;
```
馬上跑出結果：
![](https://i.imgur.com/RXZRdCm.png)

2. 同樣的對第二個視窗進行第一個指令：

```sql 
UPDATE accounts SET balance = balance - 10 WHERE id = 2 RETURNING *
```
![](https://i.imgur.com/8dX1ThD.png)

3. 此時對第一個視窗進行第二個指令的update會發現發生block
```sql 
UPDATE accounts SET balance = balance + 10 WHERE id = 2 RETURNING *;
```
![](https://i.imgur.com/pdY0ilm.png)

4. 此時若是對第二個視窗進行第二個指令更新，會得到deadlock
![](https://i.imgur.com/k77mp1e.png)


* 把上面的過程寫在test中就會發生deadlock
```go
func TestTransferTxDeadlock(t *testing.T) {
	store := NewStore(testDB)

	account1 := createRandomAccount(t)
	account2 := createRandomAccount(t)
	fmt.Println(">> before:", account1.Balance, account2.Balance)

	// run a concurrent transfer transactions
	n := 10
	amount := int64(10)
	errs := make(chan error)

	for i := 0; i < n; i++ {
		fromAccountID := account1.ID
		toAccountID := account2.ID

		if i%2 == 1 {
			fromAccountID = account2.ID
			toAccountID = account1.ID
		}
		go func() {
			ctx := context.Background()
			_, err := store.TransferTx(ctx, TransferTxParams{
				FromAccountID: fromAccountID,
				ToAccountID:   toAccountID,
				Amount:        amount,
			})

			errs <- err
		}()
	}

	// check results
	for i := 0; i < n; i++ {
		err := <-errs
		require.NoError(t, err)
	}

	// check the final updated balances
	updatedAccount1, err := testQueries.GetAccount(context.Background(), account1.ID)
	require.NoError(t, err)

	updatedAccount2, err := testQueries.GetAccount(context.Background(), account2.ID)
	require.NoError(t, err)

	fmt.Println(">> after:", updatedAccount1.Balance, updatedAccount2.Balance)
	require.Equal(t, account1.Balance, updatedAccount1.Balance)
	require.Equal(t, account2.Balance, updatedAccount2.Balance)
}
```
run test 後不意外的會發生deadlock:
![](https://i.imgur.com/yVfJ6uG.png)

* 如何避免？
把次序替換，確保id=1的操作都優先於id=2的操作
the best defense against deadlocks is to avoid them by making sure that our application always acquire locks in a consistent order.
![](https://i.imgur.com/WAEOvIf.png)

例如，在store.go中將這段程式碼改成如此，確保update的次序id都是由小而大的(consistent order)：
```go 
if arg.FromAccountID < arg.ToAccountID {
	result.FromAccount, err = q.AddAccountBalance(ctx, AddAccountBalanceParams{
		ID:     arg.FromAccountID,
		Amount: -arg.Amount,
	})
	if err != nil {
		return err
	}

	result.ToAccount, err = q.AddAccountBalance(ctx, AddAccountBalanceParams{
		ID:     arg.ToAccountID,
		Amount: arg.Amount,
	})
	if err != nil {
		return err
	}
} else {
	result.ToAccount, err = q.AddAccountBalance(ctx, AddAccountBalanceParams{
		ID:     arg.ToAccountID,
		Amount: arg.Amount,
	})
	if err != nil {
		return err
	}

	result.FromAccount, err = q.AddAccountBalance(ctx, AddAccountBalanceParams{
		ID:     arg.FromAccountID,
		Amount: -arg.Amount,
	})
	if err != nil {
		return err
	}
}
```
## Understand isolation levels & read phenomena in MySQL & PostgreSQL via examples

### Read Phenomena

- DIRTY READ
  A transaction **reads** data written by other concurrent **uncommitted** transaction
- NON-REPEATABLE READ
  A transaction **reads** the **same row twice** and sees different value because it has been **modified** by other **committed** transaction
- PHANTOM READ
  A transaction **re-executes** a query to **find rows** that satisfy a condition and sees a **different set** of rows, due to changes by other **committed** transaction
- SERIALIZATION ANOMALY
  The result of a **group** of concurrent **committed transactions** is **impossible to achieve** if we try to run them **sequentially** in any order without overlapping.

### 4 Standard Isolation Levels(American National Standards Institure-ANSI)

- Low Level

1. READ UNCOMMITTED: Can see data written by uncommitted transaction
2. READ COMMITTED: Only see data written by committed transaction
3. REPEATABLE READ: Same read query always reutrns same result.
4. SERIALIZABLE: Can achieve same result if execute transactions serially in some order instead of concurrently.

### See 4 isolation Levels in MySQL & postgreSQL

- MySQL

1. READ UNCOMMITTED: 當左邊尚未commit, 右邊就能讀取，會有dirty read 的問題
   ![](https://i.imgur.com/Cl5G5Wc.jpg)
2. READ COMMITTED: 此時左邊更新了，但右邊仍得到相同的結果(90)，代表read committed 可以預防 dirty read
   ![](https://i.imgur.com/XzywHf0.jpg)
3. REPEATABLE READ: 同樣的query得到的結果都相同，能預防phantom read（即相同的query，得到不同的結果)，看右邊底下，會有 serializable anomaly 的問題 (即一次-10的update卻顯示-20 (80 -> 60))
   ![](https://i.imgur.com/uILFuyH.jpg)
4. SERIALIZABLE
   MySQL 是靠鎖住來預防 serializable anomaly 的，因此右邊要 commit;左邊才會執行 update
   ![](https://i.imgur.com/EjaF8RE.jpg)

- postgreSQL
  因為 postgre 預設 read uncommitted 和 read committed 是一樣的，因此只有三個 isolation level.

1. READ UNCOMMITTED & READ COMMITTED:
   當左邊已經更新了，但右邊並不曉得，因此，會導致右邊相同的查詢有不同的值出現的情況，這就是 PHANTOM READ
   ![](https://i.imgur.com/OgZO7BE.jpg)
2. REPEATABLE READ
   同樣的情況，在 REAPEATABLE READ 中會提醒錯誤發生(could not serialize access due to concurrent update)
   ![](https://i.imgur.com/1Yseacc.jpg)
   這是在 REAPEATABLE READ 下發生的 SERIALIZATION ANOMALY(看右下角)
   ![](https://i.imgur.com/zEvZHQ1.jpg)

3. SERIALIZABLE
   右下角提醒了(CHECK)(HINT: The transaction might succeed if retried.)來避免 SERIALIZATION ANOMALY
   ![](https://i.imgur.com/aLkG26B.jpg)

### Isolation level in MySQL:
![](https://i.imgur.com/mkHuLzm.jpg)

### Isolation level in PostgreSQL:
![](https://i.imgur.com/4l3bcev.jpg)

### Summary
level 4 可以避免 全部
level 3 可以避免 SERIALIZATION ANOMALY 以外的問題
...

mysql default mode is REPEATABLE COMMITTED
postgreSql mode is READ COMMITTED

mysql 利用 Loking mechanism
postgresql 利用 dependencies detection


## Simple-bank - Setup Github Actions for Golang + Postgres to run automated tests

### Workflow
![](https://i.imgur.com/t9GqWsC.jpg)

### Runner
![](https://i.imgur.com/Z0WTbdK.jpg)


### Job
![](https://i.imgur.com/TG5FT5L.jpg)

### step & action
![](https://i.imgur.com/31ayQLm.jpg)

### Summary
![](https://i.imgur.com/tm3ix8W.jpg)


1. write cml.yml

這是github提供的golang模板
```yaml
name: ci-test

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:

  test:
    name: Test
    runs-on: ubuntu-latest
    steps:

    - name: Set up Go 1.x
      uses: actions/setup-go@v2
      with:
        go-version: ^1.19
      id: go

    - name: Check out code into the Go module directory
      uses: actions/checkout@v2
```
加上 make test:
```yaml
- name: Test
  run: make test
```

直接提交後，因為沒有連接資料庫發現這個錯誤
![](https://i.imgur.com/YO8P88E.png)

2. 因此在google搜尋：github action postgres，並點開第一個連結：https://docs.github.com/en/actions/using-containerized-services/creating-postgresql-service-containers

並在 ci.yml 加上：
```yaml
# Service containers to run with `container-job`
services:
  # Label used to access the service container
  postgres:
    # Docker Hub image
    image: postgres:15
    # Provide the password for postgres
    env:
      POSTGRES_USER: root
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: simple_bank
    ports:
      # Map tcp port 5432 on service container to the host
      - 5432:5432
    # Set health checks to wait until postgres has started
    options: >-
      --health-cmd pg_isready
      --health-interval 10s
      --health-timeout 5s
      --health-retries 5
```
還有run migrate:
```yaml
- name: Run migrations
  run: make migrateup
```

直接提交後，因為還沒有安裝migration，因此會報錯
![](https://i.imgur.com/qSVzJol.png)

3. 安裝 golang-migrate
```yaml
- name: Install golang-migrate
  run: |
    curl -L https://github.com/golang-migrate/migrate/releases/download/v4.14.1/migrate.linux-amd64.tar.gz | tar xvz
    sudo mv migrate.linux-amd64 /usr/bin/migrate
    which migrate
```
因為移動東西到/usr/bin要有權限，因此加上sudo
* 欲知詳細的邏輯演進可以去看 github commit 順序