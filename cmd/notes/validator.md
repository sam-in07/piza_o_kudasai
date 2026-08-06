This file registers **custom validation rules** for your Gin application. These rules ensure that users can only submit **valid pizza types** and **valid pizza sizes**.

Let's go through it step by step.

---

# 1. Package

```go
package cmd
```

This file belongs to the `cmd` package.

---

# 2. Imports

```go
import (
    "pizza-tracker-go/internal/models"
    "slices"

    "github.com/gin-gonic/gin/binding"
    "github.com/go-playground/validator"
)
```

### `models`

Imports:

```go
models.PizzaTypes
models.PizzaSizes
```

which are defined as:

```go
var PizzaTypes = []string{
    "Margherita",
    "Pepperoni",
    ...
}
```

and

```go
var PizzaSizes = []string{
    "Small",
    "Medium",
    "Large",
    "X-Large",
}
```

---

### `slices`

This is a Go standard library package (Go 1.21+).

It provides useful slice operations.

You're using:

```go
slices.Contains(...)
```

Example:

```go
sizes := []string{"Small", "Medium", "Large"}

slices.Contains(sizes, "Medium")
```

returns

```text
true
```

while

```go
slices.Contains(sizes, "Huge")
```

returns

```text
false
```

---

### `binding`

```go
"github.com/gin-gonic/gin/binding"
```

Gin uses this package to bind incoming JSON or form data to Go structs and validate them.

Example:

```go
c.ShouldBindJSON(&order)
```

During binding, Gin automatically runs the registered validators.

---

### `validator`

```go
"github.com/go-playground/validator"
```

This is the validation library used internally by Gin.

It supports tags like:

```go
binding:"required"
binding:"email"
binding:"min=3"
```

and also lets you create custom validators.

---

# 3. RegisterCustomValidators()

```go
func RegisterCustomValidators() {
```

This function registers your custom validation rules.

---

## Get Gin's validator

```go
if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
```

### What is happening?

Gin already has a validator.

This line retrieves it and converts (type asserts) it to:

```go
*validator.Validate
```

If successful:

```go
v
```

can register new validation rules.

---

## Register pizza type validator

```go
v.RegisterValidation(
    "valid_pizza_type",
    createSliceValidator(models.PizzaTypes),
)
```

This creates a validation rule called:

```text
valid_pizza_type
```

Later you can use it in a struct:

```go
Pizza string `binding:"required,valid_pizza_type"`
```

If the client sends:

```json
{
    "pizza": "Pepperoni"
}
```

✅ Passes.

If they send:

```json
{
    "pizza": "Chocolate Pizza"
}
```

❌ Validation fails because `"Chocolate Pizza"` is not in `models.PizzaTypes`.

---

## Register pizza size validator

```go
v.RegisterValidation(
    "valid_pizza_size",
    createSliceValidator(models.PizzaSizes),
)
```

Now you can write:

```go
Size string `binding:"required,valid_pizza_size"`
```

Valid:

```text
Small
Medium
Large
X-Large
```

Invalid:

```text
Tiny
Huge
XXXL
```

---

# 4. createSliceValidator()

```go
func createSliceValidator(allowedValues []string) validator.Func
```

This is a helper function.

Instead of writing two almost identical validators, you create one reusable function.

---

## Parameter

```go
allowedValues []string
```

Example:

```go
[]string{
    "Small",
    "Medium",
    "Large",
}
```

---

## Returns

```go
validator.Func
```

A validation function that Gin can call.

---

## The returned function

```go
return func(fl validator.FieldLevel) bool {
```

Whenever Gin validates a field, it passes information about that field through `fl`.

Suppose:

```go
Size = "Medium"
```

then

```go
fl.Field().String()
```

returns:

```text
Medium
```

---

## Validation

```go
return slices.Contains(
    allowedValues,
    fl.Field().String(),
)
```

If

```go
allowedValues = []string{
    "Small",
    "Medium",
    "Large",
}
```

and the input is:

```text
Medium
```

then

```go
slices.Contains(...)
```

returns

```text
true
```

Validation succeeds.

If the input is:

```text
Huge
```

then

```text
false
```

Validation fails.

---

# Example Usage

Suppose you have:

```go
type OrderItemRequest struct {
    Pizza string `json:"pizza" binding:"required,valid_pizza_type"`
    Size  string `json:"size" binding:"required,valid_pizza_size"`
}
```

Incoming request:

```json
{
    "pizza": "Pepperoni",
    "size": "Large"
}
```

Validation flow:

```text
ShouldBindJSON()
        │
        ▼
required?
        │
        ▼
valid_pizza_type?
        │
        ▼
Pepperoni exists?
        │
       Yes
        │
        ▼
valid_pizza_size?
        │
        ▼
Large exists?
        │
       Yes
        │
        ▼
Request accepted
```

If the request is:

```json
{
    "pizza": "Ice Cream",
    "size": "Huge"
}
```

then:

* `valid_pizza_type` → ❌ fails
* `valid_pizza_size` → ❌ fails

and Gin returns a validation error.

---

## Why use `createSliceValidator`?

Without it, you'd write two nearly identical functions:

```go
func validatePizzaType(...) { ... }
func validatePizzaSize(...) { ... }
```

Instead, you write one generic function that works for **any slice of allowed strings**. If later you want to validate order statuses, you can reuse it:

```go
v.RegisterValidation(
    "valid_order_status",
    createSliceValidator(models.OrderStatuses),
)
```

This keeps your code shorter, easier to maintain, and reusable.
