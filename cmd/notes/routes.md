This file is the **routing/URL map** for your entire pizza tracker.

If `main.go` is responsible for **building the application**, then `setupRoutes()` is responsible for telling Gin:

> "When a request comes to this URL with this HTTP method, call this handler."

The overall flow is:

```text
Browser
   │
   │ HTTP request
   ▼
Gin Router
   │
   ├── /                    → new order page
   ├── /new-order           → create order
   ├── /customer/:id        → customer order tracking
   ├── /notifications       → customer notifications
   ├── /login               → login
   ├── /logout              → logout
   │
   └── /admin
          │
          ▼
     AuthMiddleware
          │
          ├── not logged in → /login
          │
          └── logged in
                 │
                 ├── /admin
                 ├── /admin/order/:id/update
                 ├── /admin/order/:id/delete
                 └── /admin/notifications
```

Let's go through it.

---

# 1. Function definition

```go
func setupRoutes(
    router *gin.Engine,
    h *Handler,
    store sessions.Store,
) {
```

This function receives three things.

### `router`

```go
router *gin.Engine
```

This is your Gin web server/router.

You created it in `main.go`:

```go
router := gin.Default()
```

You're passing it here so this function can register routes.

---

### `h`

```go
h *Handler
```

This is the `Handler` you created earlier:

```go
h := NewHandler(dbModel)
```

It gives the routes access to all your handlers:

```text
h
├── HandleLoginPost
├── HandleLogout
├── HandleNewOrderPost
├── ServeAdminDashboard
├── HandleOrderPut
├── HandleOrderDelete
└── ...
```

---

### `store`

```go
store sessions.Store
```

This is the session storage system you created in `main.go`:

```go
sessionStore := setupSessionStore(...)
```

So `setupRoutes()` receives the session store and attaches it to Gin.

---

# 2. Enable sessions

```go
router.Use(
    sessions.Sessions("pizza-tracker", store),
)
```

This adds **session middleware** to the router.

It means every request can access the application's session.

The session is named:

```text
pizza-tracker
```

So your request pipeline becomes:

```text
HTTP Request
     ↓
Session Middleware
     ↓
Route Handler
```

This connects directly to your earlier login code:

```go
SetSessionValue(c, "userID", ...)
SetSessionValue(c, "username", ...)
```

and:

```go
GetSessionString(c, "userID")
```

The session middleware makes that possible.

---

# 3. Home page

```go
router.GET("/", h.ServeNewOrderForm)
```

This says:

> When someone sends a `GET` request to `/`, call `h.ServeNewOrderForm`.

So:

```text
GET /
 ↓
ServeNewOrderForm()
 ↓
order.tmpl
```

Your earlier handler was:

```go
func (h *Handler) ServeNewOrderForm(c *gin.Context) {
    c.HTML(http.StatusOK, "order.tmpl", OrderFormData{
        PizzaTypes: models.PizzaTypes,
        PizzaSizes: models.PizzaSizes,
    })
}
```

So visiting:

```text
http://localhost:8080/
```

shows the pizza order form.

---

# 4. Create a new order

```go
router.POST("/new-order", h.HandleNewOrderPost)
```

This handles a `POST` request.

So your form probably submits to:

```text
POST /new-order
```

which calls:

```go
h.HandleNewOrderPost
```

The flow is:

```text
Customer fills form
       ↓
POST /new-order
       ↓
HandleNewOrderPost()
       ↓
Validate form
       ↓
Create Order
       ↓
Notify admins
       ↓
Redirect to /customer/:id
```

This connects directly to the code you showed earlier.

---

# 5. Customer order page

```go
router.GET("/customer/:id", h.serveCustomer)
```

The `:id` is a **route parameter**.

For example:

```text
/customer/123
```

matches:

```text
/customer/:id
```

Inside your handler:

```go
orderID := c.Param("id")
```

will give:

```text
"123"
```

So:

```text
GET /customer/123
        ↓
serveCustomer()
        ↓
c.Param("id")
        ↓
"123"
        ↓
GetOrder("123")
        ↓
customer.tmpl
```

This is how the customer gets their individual order tracking page.

---

# 6. Customer notifications

```go
router.GET("/notifications", h.notificationHandler)
```

This route is probably the endpoint that connects the browser to your `NotificationManager`.

You haven't shown `notificationHandler()` yet, but based on the code you've already shown, it's probably responsible for something like:

```text
Browser
   ↓
GET /notifications
   ↓
notificationHandler
   ↓
NotificationManager
   ↓
chan string
   ↓
real-time updates
```

The exact mechanism depends on whether you're using Server-Sent Events (SSE), streaming, or something else.

---

# 7. Login page

```go
router.GET("/login", h.HandleLoginGet)
```

A browser visiting:

```text
GET /login
```

gets:

```go
h.HandleLoginGet
```

Your earlier handler:

```go
func (h *Handler) HandleLoginGet(c *gin.Context) {
    c.HTML(http.StatusOK, "login.tmpl", LoginData{})
}
```

So:

```text
GET /login
    ↓
HandleLoginGet
    ↓
login.tmpl
```

---

# 8. Login submission

```go
router.POST("/login", h.HandleLoginPost)
```

When the login form is submitted:

```text
POST /login
```

Gin calls:

```go
h.HandleLoginPost
```

That handler:

1. Validates username/password
2. Authenticates the user
3. Stores `userID` in the session
4. Stores `username` in the session
5. Redirects to `/admin`

So:

```text
POST /login
     ↓
HandleLoginPost()
     ↓
AuthenticateUser()
     ↓
SetSessionValue("userID")
     ↓
SetSessionValue("username")
     ↓
303 /admin
```

---

# 9. Logout

```go
router.POST("/logout", h.HandleLogout)
```

This handles:

```text
POST /logout
```

which calls:

```go
h.HandleLogout
```

Your previous code:

```go
ClearSession(c)
```

then redirects to:

```text
/login
```

So:

```text
POST /logout
     ↓
HandleLogout
     ↓
ClearSession
     ↓
/login
```

---

# 10. The interesting part: `router.Group`

Now we get to:

```go
admin := router.Group("/admin")
```

This creates a **route group**.

Instead of writing `/admin` repeatedly, you can group related routes together.

For example:

```go
admin.GET("", ...)
admin.POST("/order/:id/update", ...)
```

become:

```text
/admin
/admin/order/:id/update
```

because the group prefix is:

```text
/admin
```

---

# 11. Apply authentication middleware to the group

This is one of the most important lines:

```go
admin.Use(h.AuthMiddleware())
```

It means:

> Every route inside this `admin` group must pass `AuthMiddleware`.

Remember your middleware:

```go
func (h *Handler) AuthMiddleware() gin.HandlerFunc
```

It checks:

```text
Does session have userID?
        ↓
Does that user exist?
```

So the admin routes are protected.

---

# 12. Why the `{ }` block?

You have:

```go
{
    admin.GET("", h.ServeAdminDashboard)
    admin.POST("/order/:id/update", h.HandleOrderPut)
    admin.POST("/order/:id/delete", h.HandleOrderDelete)
    admin.GET("/notifications", h.adminNotificationHandler)
}
```

The braces create a normal Go block.

They help visually group all the routes belonging to the admin group.

They don't create a new HTTP feature by themselves.

---

# 13. Admin dashboard

```go
admin.GET("", h.ServeAdminDashboard)
```

Because the group prefix is:

```text
/admin
```

and the route is:

```text
""
```

the final route is:

```text
GET /admin
```

The request goes through:

```text
AuthMiddleware
       ↓
ServeAdminDashboard
```

So:

```text
GET /admin
     ↓
Is user authenticated?
     │
     ├── NO → /login
     │
     └── YES
          ↓
    ServeAdminDashboard()
          ↓
       admin.tmpl
```

---

# 14. Update an order

```go
admin.POST(
    "/order/:id/update",
    h.HandleOrderPut,
)
```

The group prefix `/admin` is added.

Therefore the complete route is:

```text
POST /admin/order/:id/update
```

For example:

```text
POST /admin/order/123/update
```

Inside:

```go
orderId := c.Param("id")
```

gives:

```text
"123"
```

Then:

```go
newStatus := c.PostForm("status")
```

gets the new status from the form.

Then:

```go
h.orders.UpdateOrderStatus(orderId, newStatus)
```

updates the database.

---

# 15. Delete an order

```go
admin.POST(
    "/order/:id/delete",
    h.HandleOrderDelete,
)
```

The complete URL is:

```text
POST /admin/order/:id/delete
```

For example:

```text
POST /admin/order/123/delete
```

Then:

```go
orderID := c.Param("id")
```

gets:

```text
123
```

and:

```go
h.orders.DeleteOrder(orderID)
```

deletes it.

Again, authentication middleware runs first.

---

# 16. Admin notifications

```go
admin.GET(
    "/notifications",
    h.adminNotificationHandler,
)
```

The final route is:

```text
GET /admin/notifications
```

And because it belongs to the `admin` group, it automatically receives:

```go
h.AuthMiddleware()
```

So:

```text
GET /admin/notifications
          ↓
AuthMiddleware
          ↓
     authenticated?
       │       │
      NO      YES
       │       │
    /login     ▼
       │   adminNotificationHandler
       │
       └── STOP
```

This is an important benefit of route groups.

You don't have to write:

```go
admin.GET(
    "/notifications",
    h.AuthMiddleware(),
    h.adminNotificationHandler,
)
```

because the middleware was already applied to the group.

---

# 17. Static files

Finally:

```go
router.Static(
    "/static",
    "./templates/static",
)
```

This tells Gin:

> Serve files from `./templates/static` when the browser requests `/static/...`.

For example, if your project contains:

```text
templates/
└── static/
    ├── style.css
    └── app.js
```

then:

```text
/static/style.css
```

maps to:

```text
./templates/static/style.css
```

and:

```text
/static/app.js
```

maps to:

```text
./templates/static/app.js
```

Your HTML can then have something like:

```html
<link rel="stylesheet" href="/static/style.css">
```

---

# 18. Your complete route table

Based on everything you've shown, your routes currently look like this:

| Method | URL                       | Handler                    | Protected? |
| ------ | ------------------------- | -------------------------- | ---------- |
| GET    | `/`                       | `ServeNewOrderForm`        | No         |
| POST   | `/new-order`              | `HandleNewOrderPost`       | No         |
| GET    | `/customer/:id`           | `serveCustomer`            | No         |
| GET    | `/notifications`          | `notificationHandler`      | No         |
| GET    | `/login`                  | `HandleLoginGet`           | No         |
| POST   | `/login`                  | `HandleLoginPost`          | No         |
| POST   | `/logout`                 | `HandleLogout`             | No         |
| GET    | `/admin`                  | `ServeAdminDashboard`      | **Yes**    |
| POST   | `/admin/order/:id/update` | `HandleOrderPut`           | **Yes**    |
| POST   | `/admin/order/:id/delete` | `HandleOrderDelete`        | **Yes**    |
| GET    | `/admin/notifications`    | `adminNotificationHandler` | **Yes**    |
| GET    | `/static/*filepath`       | Static files               | No         |

---

# 19. How everything you've shown connects

Now we can put almost your entire application together:

```text
                         main.go
                            │
                            ▼
                    models.InitDB()
                            │
                            ▼
                       NewHandler()
                            │
                            ▼
                    ┌───────────────┐
                    │    Handler    │
                    │               │
                    │ orders        │
                    │ users         │
                    │ notifications │
                    └───────┬───────┘
                            │
                            ▼
                     setupRoutes()
                            │
             ┌──────────────┼──────────────┐
             │              │              │
             ▼              ▼              ▼
        Public routes   Login routes   Admin group
             │              │              │
             │              │        AuthMiddleware
             │              │              │
             ▼              ▼              ▼
         Customers         Login        Admin
             │              │              │
             ▼              ▼              ▼
        Create order   Session user    Dashboard
             │                             │
             ▼                             ▼
     NotificationManager             Order updates
             │
             ▼
       Real-time events
```

And the **session** sits across the application:

```text
Browser
   │
   ▼
Session Middleware
   │
   ├── userID
   └── username
          │
          ▼
    AuthMiddleware
          │
          ▼
    Protected /admin routes
```

---

## One important distinction

There are **three different concepts** working together here:

### Router

Answers:

> **Which function should handle this URL?**

```go
router.GET("/login", h.HandleLoginGet)
```

### Middleware

Answers:

> **Should this request be allowed to reach the handler?**

```go
admin.Use(h.AuthMiddleware())
```

### Handler

Answers:

> **What should we actually do with the request?**

```go
h.ServeAdminDashboard
h.HandleNewOrderPost
h.HandleOrderPut
```

So for an admin request:

```text
GET /admin/order/123/update
          │
          ▼
       Router
          │
          ▼
   AuthMiddleware
          │
     ┌────┴────┐
     │         │
   denied    allowed
     │         │
     ▼         ▼
  /login   Handler
               │
               ▼
        Database / response
```

**That's the architecture you've been building across all the files you've shown.**
