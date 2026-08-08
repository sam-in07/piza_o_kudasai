This file adds **custom validation rules** to Gin's form validation system.

In your pizza tracker, this is what makes sure that a customer can't submit something like:

```text
pizza = "Hamburger"
size  = "Huge"
```

when your application only allows specific pizza types and sizes.

The overall flow is:

```text
Customer submits form
        ↓
ShouldBind(&form)
        ↓
Gin validator
        ↓
required?
min/max?
valid_pizza_type?
valid_pizza_size?
        ↓
All valid?
   ┌────┴────┐
  YES       NO
   │         │
   ▼         ▼
Create     Error
order
```

---

# 1. Imports

```go
import (
    "pizza-tracker-go/internal/models"
    "slices"

    "github.com/gin-gonic/gin/binding"
    "github.com/go-playground/validator/v10"
)
```

You have four important dependencies.

### `models`

```go
"pizza-tracker-go/internal/models"
```

This contains your allowed pizza values.

From your previous code, you have:

```go
models.PizzaTypes
models.PizzaSizes
```

For example, conceptually:

```go
models.PizzaTypes = []string{
    "Margherita",
    "Pepperoni",
    "Hawaiian",
}
```

and:

```go
models.PizzaSizes = []string{
    "Small",
    "Medium",
    "Large",
}
```

---

### `slices`

```go
"slices"
```

This is Go's standard `slices` package.

You're using:

```go
slices.Contains(...)
```

to check whether a value exists inside a slice.

For example:

```go
slices.Contains(
    []string{"Small", "Medium", "Large"},
    "Medium",
)
```

returns:

```text
true
```

while:

```go
slices.Contains(
    []string{"Small", "Medium", "Large"},
    "Huge",
)
```

returns:

```text
false
```

---

### Gin binding

```go
"github.com/gin-gonic/gin/binding"
```

Gin uses this package to handle request/form binding and validation.

This is what connects your struct tags like:

```go
binding:"required,min=2,max=100"
```

to the validator library.

---

### Validator

```go
"github.com/go-playground/validator/v10"
```

This is the actual validation library used by Gin.

Your previous `OrderReuqest` had:

```go
Name string `form:"name" binding:"required,min=2,max=100"`
```

and:

```go
Sizes []string `form:"size" binding:"required,min=1,dive,valid_pizza_size"`
```

The validator understands those tags.

---

# 2. `RegisterCustomValidators()`

```go
func RegisterCustomValidators() {
```

This function registers your custom validation rules.

You call it from `main.go`:

```go
RegisterCustomValidators()
```

The important thing is **when** you call it.

Your startup sequence is approximately:

```text
main()
  ↓
loadConfig()
  ↓
InitDB()
  ↓
RegisterCustomValidators()
  ↓
NewHandler()
  ↓
setupRoutes()
  ↓
server starts
```

So by the time a customer submits an order, your custom validators have already been registered.

---

# 3. Getting Gin's validator

```go
if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
```

This looks complicated, but it is basically doing two things.

### First:

```go
binding.Validator.Engine()
```

asks Gin:

> Give me the underlying validation engine.

Gin normally uses `go-playground/validator`.

---

### Second:

```go
.(*validator.Validate)
```

is a **type assertion**.

You're saying:

> I expect this validation engine to be a `*validator.Validate`.

The result gives you two variables:

```go
v
ok
```

For example:

```text
v  → validator engine
ok → true/false
```

If the assertion succeeds:

```text
ok = true
```

then you register your validators.

---

# 4. Register `valid_pizza_type`

```go
v.RegisterValidation(
    "valid_pizza_type",
    createSliceValidator(models.PizzaTypes),
)
```

This creates a custom validation tag called:

```text
valid_pizza_type
```

Now your struct can say:

```go
PizzaTypes []string `binding:"...,valid_pizza_type"`
```

and the validator knows what `valid_pizza_type` means.

---

# 5. Register `valid_pizza_size`

```go
v.RegisterValidation(
    "valid_pizza_size",
    createSliceValidator(models.PizzaSizes),
)
```

Same idea.

You're creating:

```text
valid_pizza_size
```

which checks whether the submitted size is one of your allowed sizes.

---

# 6. Why `createSliceValidator()`?

This is a clever part of your code.

Instead of writing:

```go
func validatePizzaType(...) bool {
    ...
}

func validatePizzaSize(...) bool {
    ...
}
```

you create **one generic validator factory**:

```go
func createSliceValidator(
    allowedValues []string,
) validator.Func
```

It receives a list of allowed values.

For pizza types:

```go
createSliceValidator(models.PizzaTypes)
```

For sizes:

```go
createSliceValidator(models.PizzaSizes)
```

So:

```text
                  createSliceValidator()
                         │
              ┌──────────┴──────────┐
              │                     │
      models.PizzaTypes      models.PizzaSizes
              │                     │
              ▼                     ▼
       pizza type validator    size validator
```

This avoids duplicated code.

---

# 7. The return value

```go
return func(fl validator.FieldLevel) bool {
```

This function returns another function.

That is a **function closure**.

The returned function has to match the type expected by:

```go
validator.Func
```

Its job is simply:

```text
value valid → true
value invalid → false
```

---

# 8. `FieldLevel`

```go
fl validator.FieldLevel
```

`fl` gives the validator information about the field currently being validated.

For example, suppose:

```go
PizzaTypes = []string{"Pepperoni"}
```

The validator is checking:

```text
"Pepperoni"
```

`fl` lets you access that value.

---

# 9. `fl.Field()`

```go
fl.Field()
```

gets the reflected value of the field being validated.

Then:

```go
fl.Field().String()
```

turns it into a string.

So conceptually:

```text
form field
   ↓
fl.Field()
   ↓
.String()
   ↓
"Pepperoni"
```

---

# 10. The actual validation

The most important line is:

```go
return slices.Contains(
    allowedValues,
    fl.Field().String(),
)
```

Suppose:

```go
allowedValues = []string{
    "Margherita",
    "Pepperoni",
    "Hawaiian",
}
```

and the user submits:

```text
Pepperoni
```

Then:

```go
slices.Contains(
    allowedValues,
    "Pepperoni",
)
```

returns:

```text
true
```

Validation succeeds.

But if they submit:

```text
Pineapple Supreme
```

then:

```go
slices.Contains(
    allowedValues,
    "Pineapple Supreme",
)
```

returns:

```text
false
```

Validation fails.

---

# 11. Connecting this to your `OrderReuqest`

This is where your previous code becomes much clearer.

You had:

```go
type OrderReuqest struct {
    Name         string   `form:"name" binding:"required,min=2,max=100"`
    Phone        string   `form:"phone" binding:"required,min=10,max=20"`
    Address      string   `form:"address" binding:"required,min=5,max=200"`
    Sizes        []string `form:"size" binding:"required,min=1,dive,valid_pizza_size"`
    PizzaTypes   []string `form:"pizza" binding:"required,min=1,dive,valid_pizza_type"`
    Instructions []string `form:"instructions" binding:"max=200"`
}
```

The interesting fields are:

```go
Sizes []string `... dive,valid_pizza_size`
```

and:

```go
PizzaTypes []string `... dive,valid_pizza_type`
```

---

# 12. What does `dive` mean?

This is particularly important.

You have:

```text
[]string
```

which means a **slice containing multiple strings**.

For example:

```text
Sizes:
[
    "Large",
    "Medium",
    "Small"
]
```

The validator needs to validate each element individually.

That's what:

```text
dive
```

means.

It tells the validator:

> Go inside this collection and validate each element.

So:

```go
binding:"required,min=1,dive,valid_pizza_size"
```

can be understood as:

```text
required
   ↓
slice must exist

min=1
   ↓
must contain at least one item

dive
   ↓
go through each item

valid_pizza_size
   ↓
each item must be an allowed pizza size
```

---

# 13. Example

Suppose:

```go
models.PizzaSizes = []string{
    "Small",
    "Medium",
    "Large",
}
```

The customer submits:

```text
size=Large
size=Medium
size=Small
```

Validator sees:

```text
Sizes
├── "Large"  → valid_pizza_size → true
├── "Medium" → valid_pizza_size → true
└── "Small"  → valid_pizza_size → true
```

Everything passes.

But:

```text
size=Large
size=Huge
size=Small
```

becomes:

```text
Sizes
├── "Large" → true
├── "Huge"  → false ❌
└── "Small" → true
```

So the entire validation fails.

---

# 14. Pizza types work the same way

Suppose:

```go
models.PizzaTypes = []string{
    "Margherita",
    "Pepperoni",
    "Hawaiian",
}
```

Then:

```text
pizza=Pepperoni
pizza=Hawaiian
```

becomes:

```text
PizzaTypes
├── Pepperoni → valid_pizza_type → true
└── Hawaiian  → valid_pizza_type → true
```

But:

```text
pizza=Pepperoni
pizza=Burger
```

becomes:

```text
PizzaTypes
├── Pepperoni → true
└── Burger    → false ❌
```

---

# 15. Why not just trust the HTML form?

You might have something like:

```html
<select name="size">
    <option value="Small">Small</option>
    <option value="Medium">Medium</option>
    <option value="Large">Large</option>
</select>
```

You might think:

> The user can only select those options.

But that's not enough.

A user can manually send an HTTP request with:

```text
size=Huge
```

The browser UI isn't a security boundary.

That's why server-side validation is important:

```text
Browser validation
       ↓
nice user experience

Server validation
       ↓
actual enforcement
```

Your custom validators provide that server-side enforcement.

---

# 16. Why the validator uses the same model lists

This is another good design decision.

You aren't hard-coding:

```go
if value == "Small" ||
   value == "Medium" ||
   value == "Large" {
```

Instead, you use:

```go
models.PizzaSizes
```

That means your allowed values have one central source.

For example:

```go
models.PizzaSizes = []string{
    "Small",
    "Medium",
    "Large",
    "Extra Large",
}
```

The validator automatically accepts the new size.

You don't need to change the validator.

---

# 17. Your complete validation pipeline

Now let's connect this to:

```go
func (h *Handler) HandleNewOrderPost(c *gin.Context)
```

You had:

```go
var form OrderReuqest

if err := c.ShouldBind(&form); err != nil {
    c.JSON(http.StatusBadRequest, gin.H{
        "error": err.Error(),
    })
    return
}
```

Here's what happens:

```text
POST /new-order
       │
       ▼
ShouldBind(&form)
       │
       ├── Read form fields
       │
       ├── Put values into OrderReuqest
       │
       └── Run validation
                │
                ▼
       ┌─────────────────────┐
       │ validator           │
       │                     │
       │ required            │
       │ min/max             │
       │ dive                │
       │ valid_pizza_size    │
       │ valid_pizza_type    │
       └──────────┬──────────┘
                  │
             ┌────┴────┐
             │         │
          valid      invalid
             │         │
             ▼         ▼
       CreateOrder   HTTP 400
```

So your `RegisterCustomValidators()` function is effectively extending Gin's built-in validation system.

---

# 18. One thing to notice about your architecture

Your code has a nice separation:

```text
models
  │
  ├── PizzaTypes
  ├── PizzaSizes
  └── Order data
       │
       ▼
validation
       │
       ▼
HTTP handler
       │
       ▼
OrderModel
       │
       ▼
database
```

The model defines what values are allowed.

The validator enforces those values.

The handler decides what to do with valid data.

The database model stores it.

---

# 19. One small naming issue

You called your type:

```go
type OrderReuqest struct {
```

There is a typo:

```text
Reuqest
```

should normally be:

```go
type OrderRequest struct {
```

Then:

```go
var form OrderRequest
```

This doesn't break the program because Go doesn't care about the English spelling, but `OrderRequest` is much clearer.

---

# 20. In one sentence

This whole file does one main job:

> **It extends Gin's validator so `OrderRequest` can verify that every submitted pizza type and size actually exists in your application's allowed lists.**

The clever part is this reusable function:

```go
func createSliceValidator(allowedValues []string) validator.Func {
    return func(fl validator.FieldLevel) bool {
        return slices.Contains(
            allowedValues,
            fl.Field().String(),
        )
    }
}
```

You give it a list:

```text
PizzaTypes → validator
PizzaSizes → validator
```

and it creates a validator that checks:

```text
"Is the submitted value contained in this list?"
```

That's a clean way to avoid duplicating validation logic.
