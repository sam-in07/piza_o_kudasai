No problem! Yes, **PostgreSQL is fully supported by GORM**. You only need to change the driver and connection code. Most of your models (`Order`, `OrderItem`, `User`) can remain the same.

### 1. Install the PostgreSQL driver

```bash
go get gorm.io/driver/postgres
```

---

### 2. Change the import

Instead of:

```go
import "gorm.io/driver/sqlite"
```

use:

```go
import "gorm.io/driver/postgres"
```

---

### 3. Update `InitDB`

Replace:

```go
db, err := gorm.Open(sqlite.Open(dataSourceName), &gorm.Config{})
```

with something like:

```go
func InitDB(dsn string) (*DBModel, error) {
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        return nil, fmt.Errorf("failed to connect database: %v", err)
    }

    err = db.AutoMigrate(&Order{}, &OrderItem{}, &User{})
    if err != nil {
        return nil, fmt.Errorf("failed to migrate database: %v", err)
    }

    dbModel := &DBModel{
        DB:    db,
        Order: OrderModel{DB: db},
        User:  UserModel{DB: db},
    }

    return dbModel, nil
}
```

---

### 4. PostgreSQL connection string (DSN)

A typical DSN looks like:

```text
host=localhost user=postgres password=yourpassword dbname=pizza port=5432 sslmode=disable
```

Then call:

```go
dbModel, err := models.InitDB(
    "host=localhost user=postgres password=1234 dbname=pizza port=5432 sslmode=disable",
)
```

---

### 5. Your models don't need major changes

Your existing model:

```go
type Order struct {
    ID           string      `gorm:"primaryKey;size:14"`
    Status       string
    CustomerName string
    Phone        string
    Address      string
    Items        []OrderItem `gorm:"foreignKey:OrderID"`
    CreatedAt    time.Time
}
```

will still work with PostgreSQL. GORM will create equivalent PostgreSQL tables and columns.

---

### 6. What changes?

Mostly just the database connection:

| SQLite                      | PostgreSQL                         |
| --------------------------- | ---------------------------------- |
| `gorm.io/driver/sqlite`     | `gorm.io/driver/postgres`          |
| `sqlite.Open("pizza.db")`   | `postgres.Open(dsn)`               |
| Stores data in a `.db` file | Stores data in a PostgreSQL server |
| No server needed            | PostgreSQL server must be running  |

Everything else—`Create()`, `Find()`, `First()`, `Preload()`, `Update()`, `Delete()`, and `AutoMigrate()`—works the same because GORM provides a consistent API across supported SQL databases.

So if you move from SQLite to PostgreSQL, you'll mainly change the driver and connection string while keeping most of your application code unchanged.
