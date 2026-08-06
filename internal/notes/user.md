This file defines the **User model** and provides methods to **authenticate a user** and **retrieve a user by ID**. Let's go through it section by section.

---

# 1. Package

```go
package models
```

This file belongs to the `models` package.

---

# 2. Imports

```go
import (
    "errors"

    "golang.org/x/crypto/bcrypt"
    "gorm.io/gorm"
)
```

### `errors`

Used to create and compare errors.

Example:

```go
errors.New("invalid credentials")
```

---

### `bcrypt`

Used to securely hash passwords and verify them.

Instead of storing:

```text
123456
```

the database stores something like:

```text
$2a$10$9gGX9m...
```

When the user logs in, bcrypt compares the entered password with the stored hash.

---

### `gorm`

Used to query the database.

---

# 3. User Struct

```go
type User struct {
    ID       uint   `gorm:"primaryKey"`
    Username string `gorm:"uniqueIndex;not null"`
    Password string `gorm:"not null"`
}
```

This struct represents a row in the `users` table.

### ID

```go
ID uint `gorm:"primaryKey"`
```

* Primary key
* Auto-incremented by the database

Example:

| ID | Username |
| -- | -------- |
| 1  | admin    |
| 2  | john     |

---

### Username

```go
Username string `gorm:"uniqueIndex;not null"`
```

* Cannot be empty (`not null`)
* Must be unique (`uniqueIndex`)

The database won't allow:

| Username |
| -------- |
| admin    |
| admin ❌  |

---

### Password

```go
Password string `gorm:"not null"`
```

Stores the **hashed** password, not the plain-text password.

Example:

```text
$2a$10$B8wR...
```

---

# 4. UserModel

```go
type UserModel struct {
    DB *gorm.DB
}
```

This holds the database connection.

It allows methods like:

```go
userModel.AuthenticateUser(...)
userModel.GetUserByID(...)
```

---

# 5. AuthenticateUser()

```go
func (u *UserModel) AuthenticateUser(username, password string) (*User, error)
```

This function checks whether a username and password are valid.

### Step 1: Create an empty user

```go
var user User
```

Initially:

```text
User{}
```

---

### Step 2: Find the user

```go
u.DB.Where("username = ?", username).First(&user)
```

Suppose:

```go
username = "admin"
```

GORM executes a query similar to:

```sql
SELECT * FROM users
WHERE username = 'admin'
LIMIT 1;
```

If found:

```go
user = User{
    ID: 1,
    Username: "admin",
    Password: "$2a$10$..."
}
```

---

### Step 3: Handle "not found"

```go
if errors.Is(err, gorm.ErrRecordNotFound)
```

If no matching user exists:

```text
Username entered: alice

Database:
admin
john
```

Then GORM returns:

```go
gorm.ErrRecordNotFound
```

The function responds with:

```go
return nil, errors.New("invalid credentials")
```

Using a generic message like "invalid credentials" avoids revealing whether the username or password was incorrect.

---

### Step 4: Compare passwords

```go
bcrypt.CompareHashAndPassword(
    []byte(user.Password),
    []byte(password),
)
```

Suppose the database contains:

```text
Hashed Password:
$2a$10$AbCd...
```

User enters:

```text
mypassword
```

`CompareHashAndPassword()` hashes the entered password internally and checks whether it matches the stored hash.

If it doesn't:

```go
return nil, errors.New("invalid credentials")
```

If it does, authentication succeeds.

---

### Step 5: Return the user

```go
return &user, nil
```

The caller receives the authenticated user.

Example:

```go
user, err := userModel.AuthenticateUser("admin", "secret")

if err != nil {
    fmt.Println("Login failed")
} else {
    fmt.Println(user.Username)
}
```

---

# 6. GetUserByID()

```go
func (u *UserModel) GetUserByID(id string) (*User, error)
```

This function retrieves a user by their ID.

### Create an empty user

```go
var user User
```

---

### Query the database

```go
u.DB.First(&user, "id = ?", id)
```

This generates a query similar to:

```sql
SELECT *
FROM users
WHERE id = ?
LIMIT 1;
```

If:

```go
id = "1"
```

the result might be:

| ID | Username |
| -- | -------- |
| 1  | admin    |

---

### Return the result

If successful:

```go
return &user, nil
```

If not:

```go
return nil, err
```

---

# Authentication Flow

```text
User enters

Username: admin
Password: secret
        │
        ▼
AuthenticateUser()
        │
        ▼
Search database
        │
   User exists?
   │          │
 No          Yes
 │            │
 ▼            ▼
Return    Compare password
Error          │
          Match?
        │         │
       No        Yes
       │          │
       ▼          ▼
 Return Error  Return User
```

## Summary

* `User` defines the structure of the `users` table.
* `UserModel` wraps a `*gorm.DB` connection and provides user-related database methods.
* `AuthenticateUser()` first looks up the user by username, then verifies the supplied password against the stored bcrypt hash. It returns the user only if both checks succeed.
* `GetUserByID()` retrieves a user by their database ID.
* Passwords are **never compared in plain text**; bcrypt securely compares the entered password with the stored hashed password. This is a standard and recommended practice for user authentication.
