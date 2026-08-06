This Go code defines the **data model** and **database operations** for a pizza ordering system using **GORM** (an ORM for Go). Here's a breakdown.

---

# 1. Package and Imports

```go
package models
```

This file belongs to the `models` package, which usually contains database models and related logic.

```go
import (
    "time"

    "github.com/teris-io/shortid"
    "gorm.io/gorm"
)
```

Imports:

* `time` → for timestamps (`CreatedAt`)
* `shortid` → generates unique short IDs
* `gorm` → Go ORM used to communicate with the database

---

# 2. Global Variables

## Order Statuses

```go
OrderStatuses = []string{
    "Order placed",
    "Preparing",
    "Baking",
    "Quality Check",
    "Ready",
}
```

A slice containing all valid order statuses.

Example:

```
Order placed
      ↓
Preparing
      ↓
Baking
      ↓
Quality Check
      ↓
Ready
```

Instead of typing strings repeatedly, the application can use:

```go
OrderStatuses[0]
```

which is

```
Order placed
```

---

## Pizza Types

```go
PizzaTypes = []string{
    "Margherita",
    "Pepperoni",
    ...
}
```

Stores all available pizzas.

Example:

```
Margherita
Pepperoni
Vegetarian
Hawaiian
```

Useful for validating user input.

---

## Pizza Sizes

```go
PizzaSizes = []string{
    "Small",
    "Medium",
    "Large",
    "X-Large",
}
```

Possible sizes.

---

# 3. OrderModel

```go
type OrderModel struct {
    DB *gorm.DB
}
```

This struct holds a database connection.

Example:

```go
orderModel := OrderModel{
    DB: db,
}
```

Then you can call:

```go
orderModel.CreateOrder(...)
```

instead of passing `db` everywhere.

---

# 4. Order Struct

```go
type Order struct {
```

Represents one pizza order.

Fields:

```go
ID string
```

Primary key.

Example:

```
ID = "Vkd93Nac"
```

---

```go
Status string
```

Current status.

Example:

```
Preparing
```

---

```go
CustomerName string
```

Example:

```
John Doe
```

---

```go
Phone string
```

Example:

```
01712345678
```

---

```go
Address string
```

Delivery address.

---

```go
Items []OrderItem
```

One order can have many pizzas.

Example:

```
Order
 ├── Margherita
 ├── Pepperoni
 └── Hawaiian
```

This is a **one-to-many relationship**.

The GORM tag

```go
gorm:"foreignKey:OrderID"
```

tells GORM:

> Connect `Order` with `OrderItem` using the `OrderID` field.

---

```go
CreatedAt time.Time
```

Stores creation time automatically.

Example:

```
2026-08-06 15:40:02
```

---

# 5. OrderItem Struct

```go
type OrderItem struct {
```

Represents **one pizza** inside an order.

Example:

```
Order #123

Pizza 1
Pizza 2
Pizza 3
```

Each pizza is an `OrderItem`.

Fields:

---

```go
ID string
```

Primary key.

---

```go
OrderID string
```

Foreign key.

Suppose

```
Order ID = abc123
```

Then every pizza item stores

```
OrderID = abc123
```

to show which order it belongs to.

---

```go
Size string
```

Example:

```
Large
```

---

```go
Pizza string
```

Example:

```
Pepperoni
```

---

```go
Instructions string
```

Example:

```
Extra cheese
No onions
Less spicy
```

---

# 6. BeforeCreate Hook (Order)

```go
func (o *Order) BeforeCreate(tx *gorm.DB) error {
```

GORM automatically calls this before inserting a new order.

```go
if o.ID == "" {
    o.ID = shortid.MustGenerate()
}
```

If the ID wasn't set manually, generate one automatically.

Example:

Before:

```
ID = ""
```

After:

```
ID = "Jd82kaP"
```

`MustGenerate()` returns a unique short ID and panics if generation fails.

---

# 7. BeforeCreate Hook (OrderItem)

Exactly the same idea:

```go
func (oi *OrderItem) BeforeCreate(tx *gorm.DB) error
```

Each pizza item also gets its own unique ID.

---

# 8. CreateOrder()

```go
func (o *OrderModel) CreateOrder(order *Order) error {
    return o.DB.Create(order).Error
}
```

Creates a new order in the database.

Example:

```go
err := orderModel.CreateOrder(&order)
```

Equivalent SQL:

```sql
INSERT INTO orders (...)
VALUES (...);
```

If `Items` are populated, GORM can also insert them depending on how associations are configured.

---

# 9. GetOrder()

```go
func (o *OrderModel) GetOrder(id string)
```

Gets one order by ID.

```go
Preload("Items")
```

This is important.

Without:

```go
Order
```

Only order data is returned.

With:

```go
Order
    +
All pizzas
```

Example SQL:

```sql
SELECT * FROM orders WHERE id='abc';
SELECT * FROM order_items WHERE order_id='abc';
```

Then GORM combines the results into:

```go
Order{
    Items: []OrderItem{
        ...
    }
}
```

---

# 10. GetAllOrders()

```go
func (o *OrderModel) GetAllOrders()
```

Returns every order.

```go
Order("created_at desc")
```

Sorts by newest first.

Equivalent SQL:

```sql
SELECT *
FROM orders
ORDER BY created_at DESC;
```

Again,

```go
Preload("Items")
```

loads every pizza belonging to every order.

---

# 11. UpdateOrderStatus()

```go
func (o *OrderModel) UpdateOrderStatus(id string, status string)
```

Updates only the status field.

```go
Where("id = ?", id)
```

finds the correct order.

```go
Update("status", status)
```

changes only that column.

Equivalent SQL:

```sql
UPDATE orders
SET status = 'Preparing'
WHERE id = 'abc123';
```

---

# 12. DeleteOrder()

```go
func (o *OrderModel) DeleteOrder(id string)
```

Deletes an order.

```go
Select("Items")
```

tells GORM to also delete the associated `OrderItem` records when deleting the order (assuming the associations are configured appropriately).

Equivalent SQL conceptually:

```sql
DELETE FROM order_items
WHERE order_id='abc123';

DELETE FROM orders
WHERE id='abc123';
```

---

# Relationship Diagram

```text
                Order
      +----------------------+
      | ID                   |
      | CustomerName         |
      | Phone                |
      | Address              |
      | Status               |
      | CreatedAt            |
      +----------------------+
                 |
                 | One-to-Many
                 |
         +-------+--------+
         |                |
         v                v

      OrderItem      OrderItem
   +-------------+ +-------------+
   | ID          | | ID          |
   | OrderID ----|-| OrderID ----|
   | Pizza       | | Pizza       |
   | Size        | | Size        |
   | Instructions| | Instructions|
   +-------------+ +-------------+
```

## Summary

* `Order` represents a customer's pizza order.
* `OrderItem` represents individual pizzas within an order.
* `OrderModel` wraps a `*gorm.DB` connection and provides methods to create, retrieve, update, and delete orders.
* `BeforeCreate` hooks automatically generate short unique IDs for orders and order items.
* `Preload("Items")` eagerly loads related pizza items when fetching orders.
* `UpdateOrderStatus` updates only the `status` column.
* `DeleteOrder` removes an order and is intended to delete its associated items as well, depending on the configured association behavior.
