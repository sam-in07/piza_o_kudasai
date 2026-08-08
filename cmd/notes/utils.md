This file contains several **application utility/setup functions**. It handles three major things:

```text id="f4qk3n"
Configuration
    │
    ├── PORT
    ├── DATABASE_URL
    └── SESSION_SECRET_KEY

Templates
    │
    ├── login.tmpl
    ├── order.tmpl
    ├── admin.tmpl
    └── customer.tmpl

Sessions
    │
    ├── create/store values
    ├── read values
    └── clear values
```

This file is basically the glue that makes your configuration, HTML templates, and login sessions work.

---

# 1. Imports

```go
import (
    "encoding/json"
    "html/template"
    "os"

    "github.com/gin-contrib/sessions"
    gormsessions "github.com/gin-contrib/sessions/gorm"
    "github.com/gin-gonic/gin"
    "gorm.io/gorm"
)
```

You have standard-library packages:

### `encoding/json`

Used to convert Go data into JSON:

```go
json.Marshal(v)
```

### `html/template`

Used to load and execute your `.tmpl` HTML templates.

### `os`

Used to read environment variables:

```go
os.Getenv(...)
```

And then you have your external libraries:

```text id="f4y5jd"
gin
 │
 └── web framework

gin-contrib/sessions
 │
 └── session management

gin-contrib/sessions/gorm
 │
 └── stores sessions using GORM/database

gorm
 │
 └── database abstraction
```

---

# 2. `Config`

```go
type Config struct {
    Port             string
    DBPath           string
    SessionSecretKey string
}
```

This is a simple structure that holds your application's configuration.

You can think of it as:

```text id="uh6d5s"
Config
├── Port
├── DBPath
└── SessionSecretKey
```

For example:

```text id="w4s7xn"
Port             = "8080"
DBPath           = "./data/orders.db"
SessionSecretKey = "some-secret"
```

---

# 3. `loadConfig()`

```go
func loadConfig() Config {
    return Config{
        Port:             getEnv("PORT", "8080"),
        DBPath:           getEnv("DATABASE_URL", "./data/orders.db"),
        SessionSecretKey: getEnv("SESSION_SECRET_KEY", "pizza-order-secret-key"),
    }
}
```

This creates your configuration.

Notice that it doesn't directly use:

```go
os.Getenv()
```

Instead, it calls your helper:

```go
getEnv(...)
```

---

# 4. `getEnv()`

```go
func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }

    return defaultValue
}
```

This function means:

> Get a value from an environment variable. If it doesn't exist, use the default.

For example:

```go
getEnv("PORT", "8080")
```

means:

```text id="8ajl5h"
Does PORT exist?
      │
   ┌──┴──┐
  YES    NO
   │      │
   ▼      ▼
return   "8080"
value
```

---

# 5. Why environment variables?

Environment variables let you change configuration without changing your Go source code.

For example, locally:

```text id="l8n6gp"
PORT=8080
```

But in production:

```text id="t5n0vr"
PORT=80
```

Same Go program, different configuration.

Similarly:

```text id="kqk9r5"
DATABASE_URL=./data/orders.db
```

could be different in another environment.

---

# 6. Important security issue with your default secret

You currently have:

```go id="g1n3p8"
getEnv(
    "SESSION_SECRET_KEY",
    "pizza-order-secret-key",
)
```

This means if the environment variable isn't set, your application uses:

```text id="6zq2cp"
pizza-order-secret-key
```

That's okay for a learning/local project, but **not a good production secret**.

For production, you should provide a strong random secret through the environment:

```text id="d4t7sn"
SESSION_SECRET_KEY=<strong-random-secret>
```

rather than relying on a predictable hard-coded default.

---

# 7. `loadTemplates()`

Now we get into your HTML templates.

```go
func loadTemplates(router *gin.Engine) error {
```

This function loads your `.tmpl` files into Gin.

You previously had handlers like:

```go
c.HTML(
    http.StatusOK,
    "login.tmpl",
    LoginData{},
)
```

For that to work, Gin needs to know where:

```text id="s7d8kf"
templates/login.tmpl
```

is.

That's what this function does.

---

# 8. `template.FuncMap`

```go
functions := template.FuncMap{
```

This lets you define **custom functions that your templates can call**.

You have two:

```text id="v5yq8s"
add
json
```

---

# 9. The `add` template function

```go
"add": func(a, b int) int {
    return a + b
},
```

This allows your template to do something like:

```html
{{ add 1 2 }}
```

and get:

```text
3
```

Why would you want this?

A common use is displaying an index:

```html
{{ add $index 1 }}
```

because Go template indexes start at `0`.

For example:

```text id="o8g0l9"
Go index:

0
1
2
3

Display:

1
2
3
4
```

---

# 10. The `json` template function

```go
"json": func(v interface{}) template.JS {
    b, _ := json.Marshal(v)
    return template.JS(b)
},
```

This converts Go data into JSON so JavaScript in your HTML can use it.

For example, suppose Go gives your template:

```go
[]string{"small", "medium", "large"}
```

The function can turn it into:

```json
["small","medium","large"]
```

Then your template could potentially do something like:

```html
<script>
    const sizes = {{ json .Statuses }};
</script>
```

and JavaScript receives an actual array.

---

# 11. What's `interface{}`?

```go
v interface{}
```

means:

> Accept basically any Go value.

It could be:

```text id="3fgy0p"
string
int
slice
struct
map
...
```

That's useful for a generic JSON helper.

---

# 12. `json.Marshal`

```go
b, _ := json.Marshal(v)
```

`json.Marshal` converts a Go value into JSON bytes.

For example:

```go
[]string{"Margherita", "Pepperoni"}
```

becomes roughly:

```json
["Margherita","Pepperoni"]
```

The result is `[]byte`, which is why the variable is named:

```go
b
```

for bytes.

---

# 13. Why `template.JS`?

```go
return template.JS(b)
```

This tells Go's template system:

> This value is intended to be JavaScript content.

This is powerful but should be handled carefully: **only pass data you trust or have safely serialized**, because marking something as `template.JS` bypasses some of `html/template`'s normal escaping protections.

---

# 14. Parse the templates

```go
tmpl, err := template.New("").
    Funcs(functions).
    ParseGlob("templates/*.tmpl")
```

This does several things.

### First:

```go
template.New("")
```

creates a template collection.

### Then:

```go
.Funcs(functions)
```

registers your custom functions:

```text id="8jy3rj"
add()
json()
```

### Then:

```go
.ParseGlob("templates/*.tmpl")
```

loads all `.tmpl` files from:

```text id="b7n9qk"
templates/
```

For example:

```text id="e4x7g2"
templates/
├── login.tmpl
├── order.tmpl
├── admin.tmpl
└── customer.tmpl
```

---

# 15. Check template errors

```go
if err != nil {
    return err
}
```

If one of the templates has a syntax error, `ParseGlob` returns an error.

The function returns that error to `main.go`.

Remember your `main.go`:

```go
if err := loadTemplates(router); err != nil {
    slog.Error("Failed to load templates", "error", err)
    os.Exit(1)
}
```

So the application refuses to start if the templates can't be loaded.

That's a sensible startup check.

---

# 16. Give templates to Gin

```go
router.SetHTMLTemplate(tmpl)
```

This tells Gin:

> Use these templates when `c.HTML()` is called.

Now your previous code works:

```go
c.HTML(
    http.StatusOK,
    "order.tmpl",
    OrderFormData{...},
)
```

The complete flow is:

```text id="s6x8fp"
main()
  ↓
loadTemplates(router)
  ↓
ParseGlob("templates/*.tmpl")
  ↓
router.SetHTMLTemplate(tmpl)
  ↓
Request arrives
  ↓
c.HTML(..., "order.tmpl", ...)
  ↓
Gin renders order.tmpl
```

---

# 17. `setupSessionStore`

Now we get to authentication/session management.

```go
func setupSessionStore(
    db *gorm.DB,
    secretKey []byte,
) sessions.Store {
```

This creates the application's session store.

It receives:

```text id="1mtyz4"
db
secretKey
```

---

# 18. GORM session store

```go
store := gormsessions.NewStore(
    db,
    true,
    secretKey,
)
```

You're using the GORM-backed session store.

Conceptually:

```text id="k8p7zm"
Browser
   │
   │ session cookie
   ▼
Gin Sessions
   │
   ▼
GORM Session Store
   │
   ▼
Database
```

This is why your application can persist session information.

---

# 19. Session options

```go
store.Options(sessions.Options{
    Path:     "/",
    MaxAge:   86400,
    HttpOnly: true,
    Secure:   true,
    SameSite: 3,
})
```

These options control the session cookie.

Let's examine each.

---

## `Path: "/"`

```go
Path: "/",
```

The cookie applies to the entire site.

So it can be used for:

```text id="1xg0te"
/login
/admin
/customer/123
/notifications
```

---

# 20. `MaxAge`

```go
MaxAge: 86400,
```

86400 seconds = **24 hours**.

So you're asking the browser to keep the session cookie for approximately one day.

```text
86400 seconds
     ↓
24 hours
     ↓
1 day
```

---

# 21. `HttpOnly`

```go
HttpOnly: true,
```

This is a useful security setting.

It tells the browser:

> JavaScript should not be able to directly access this cookie.

So code like:

```js
document.cookie
```

won't expose an `HttpOnly` session cookie.

This helps reduce certain cookie-theft risks from client-side scripts.

---

# 22. `Secure`

```go
Secure: true,
```

This tells the browser to send the cookie only over HTTPS.

That's good for production security.

However, there's an important development issue:

If you're testing with plain:

```text
http://localhost:8080
```

a `Secure` cookie may not behave the way you expect in some setups because `Secure` is intended for HTTPS.

Since your `main.go` currently logs:

```text
http://localhost:...
```

you should be aware of this when debugging login sessions locally.

A common setup is:

```text
Development → HTTP
Production   → HTTPS
```

with the cookie configuration adjusted accordingly.

---

# 23. `SameSite: 3`

```go
SameSite: 3,
```

This is configuring the cookie's SameSite policy using a numeric constant.

This is harder to read than using the named constant from the package.

A clearer style would generally be something like:

```go
SameSite: http.SameSiteStrictMode,
```

or another appropriate named value depending on the desired behavior.

The exact policy should be chosen based on how your application handles cross-site requests.

---

# 24. Return the store

```go
return store
```

Now `main.go` receives it:

```go
sessionStore := setupSessionStore(
    dbModel.DB,
    []byte(cfg.SessionSecretKey),
)
```

and passes it into:

```go
setupRoutes(router, h, sessionStore)
```

Then `setupRoutes()` attaches it:

```go
router.Use(
    sessions.Sessions("pizza-tracker", store),
)
```

So the complete connection is:

```text id="qztr0r"
Config
 │
 └── SessionSecretKey
          │
          ▼
setupSessionStore()
          │
          ▼
    session store
          │
          ▼
    setupRoutes()
          │
          ▼
sessions.Sessions()
          │
          ▼
     Gin requests
```

---

# 25. `SetSessionValue`

Now you have:

```go
func SetSessionValue(
    c *gin.Context,
    key string,
    value interface{},
) error {
```

This is your helper for putting data into the user's session.

Inside:

```go
session := sessions.Default(c)
```

gets the current session associated with the request.

Then:

```go
session.Set(key, value)
```

stores a value.

Then:

```go
return session.Save()
```

saves it.

---

# 26. Connecting this to your login code

Earlier you had:

```go
SetSessionValue(
    c,
    "userID",
    fmt.Sprintf("%v", user.ID),
)
```

and:

```go
SetSessionValue(
    c,
    "username",
    user.Username,
)
```

So after login, the session might conceptually contain:

```text id="y0q8op"
Session
├── userID  → "123"
└── username → "john"
```

Then your middleware can read it.

---

# 27. `GetSessionString`

```go
func GetSessionString(
    c *gin.Context,
    key string,
) string {
```

This retrieves a string from the session.

First:

```go
session := sessions.Default(c)
```

gets the current session.

Then:

```go
val := session.Get(key)
```

retrieves the value.

---

# 28. If the value doesn't exist

```go
if val == nil {
    return ""
}
```

So instead of returning `nil`, this helper returns an empty string.

That makes code like your middleware simple:

```go
userID := GetSessionString(c, "userID")

if userID == "" {
    // not logged in
}
```

---

# 29. Type assertion

```go
str, _ := val.(string)
return str
```

This says:

> Try to treat `val` as a string.

If successful:

```text id="w8k5dx"
val = "123"
 ↓
str = "123"
```

If the stored value isn't actually a string, the assertion fails and `str` becomes the zero value for a string:

```text id="4c8qz9"
""
```

You're ignoring the success/failure boolean with `_`.

A stricter version could explicitly check the type, but for your current usage the helper is keeping things simple.

---

# 30. `ClearSession`

```go
func ClearSession(c *gin.Context) error {
```

This is used during logout and invalid authentication.

Inside:

```go
session := sessions.Default(c)
```

gets the current session.

Then:

```go
session.Clear()
```

removes the session values.

And:

```go
return session.Save()
```

persists the cleared session.

---

# 31. Connecting everything to your login flow

Now all the code you've shown fits together nicely.

### Login

```text id="0xry1p"
POST /login
     ↓
HandleLoginPost()
     ↓
AuthenticateUser()
     ↓
SetSessionValue("userID", "123")
     ↓
SetSessionValue("username", "john")
     ↓
Save session
     ↓
Redirect /admin
```

### Access admin

```text id="z89qwx"
GET /admin
     ↓
Session middleware
     ↓
AuthMiddleware()
     ↓
GetSessionString("userID")
     ↓
"123"
     ↓
GetUserByID("123")
     ↓
User exists
     ↓
c.Next()
     ↓
ServeAdminDashboard()
```

### Logout

```text id="1z7g3f"
POST /logout
     ↓
HandleLogout()
     ↓
ClearSession()
     ↓
session.Clear()
     ↓
session.Save()
     ↓
Redirect /login
```

---

# 32. Your application's complete architecture so far

You've now shown enough code to see the overall architecture quite clearly:

```text id="0r9q2d"
                         main.go
                            │
              ┌─────────────┼─────────────┐
              │             │             │
              ▼             ▼             ▼
           Config         Logger       Database
              │                           │
              │                           ▼
              │                        DBModel
              │                           │
              │                           ▼
              │                      NewHandler()
              │                           │
              │                           ▼
              │                       Handler
              │                           │
              └──────────┐       ┌────────┘
                         │       │
                         ▼       ▼
                    Session    Routes
                      Store      │
                         │       │
                         └───┬───┘
                             ▼
                         Gin Router
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
          ▼                  ▼                  ▼
       Customer            Login              Admin
       Routes              Routes             Routes
          │                  │                  │
          │                  ▼                  ▼
          │              Session          AuthMiddleware
          │                                     │
          │                                     ▼
          │                              Admin handlers
          │
          ▼
      OrderModel
          │
          ▼
       Database

              NotificationManager
                       │
                       ▼
                  Real-time
                  notifications
```

### So each file has a clear responsibility:

| Component             | Responsibility                           |
| --------------------- | ---------------------------------------- |
| `main.go`             | Starts and wires the application         |
| `Handler`             | Holds dependencies used by HTTP handlers |
| `setupRoutes()`       | Maps URLs → handlers                     |
| `AuthMiddleware()`    | Protects authenticated routes            |
| `NotificationManager` | Manages real-time notification channels  |
| `loadConfig()`        | Loads application configuration          |
| `loadTemplates()`     | Loads `.tmpl` files into Gin             |
| `setupSessionStore()` | Creates database-backed sessions         |
| `SetSessionValue()`   | Writes session values                    |
| `GetSessionString()`  | Reads session values                     |
| `ClearSession()`      | Removes session values                   |

The key architectural idea is that **`main.go` constructs everything, `setupRoutes()` connects HTTP requests to handlers, middleware controls access, handlers implement application behavior, models talk to the database, and the notification manager handles real-time events.**
