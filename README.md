# Session

Session middleware for [Fiber](https://github.com/gofiber/fiber).

## Table of Contents

- [Session](#session)
	- [Table of Contents](#table-of-contents)
	- [Signatures](#signatures)
		- [Examples](#examples)
		- [Default Configuration](#default-configuration)
		- [Custom Storage/Database](#custom-storagedatabase)
	- [Config](#config)
	- [Default Config](#default-config)

## Signatures

```go
func New(config ...Config) *Store
func (s *Store) RegisterType(i interface{})
func (s *Store) Get(c *fiber.Ctx) (*Session, error)
func (s *Store) Reset() error

func (s *Session) Get(key string) interface{}
func (s *Session) Set(key string, val interface{})
func (s *Session) Delete(key string)
func (s *Session) Destroy() error
func (s *Session) Regenerate() error
func (s *Session) Save() error
func (s *Session) Fresh() bool
func (s *Session) ID() string
func (s *Session) Keys() []string
func (s *Session) SetExpiry(time.Duration) 
```

**⚠ _Storing `interface{}` values are limited to built-ins Go types_**

### Examples
Import the middleware package that is part of the Fiber web framework
```go
import (
  "github.com/gofiber/fiber/v2"
	redisstore "github.com/hyuricane/redisstore-fiber"
	session "github.com/hyuricane/session-fiber"
)
```

Then create a Fiber app with `app := fiber.New()`.

### Default Configuration

```go
// This stores all of your app's sessions
// Default middleware config
store := session.New()
sessionSecret := "session-secret"
redisclient := redis.NewClient(&redis.Options{
	Addr:     "localhost:6379",
	Password: "",
	DB:       0,
})

sessionMiddleware := session.New(session.Config{
	KeyLookup:    "cookie:connect_sid",
	SignKey: func(key string) (signed string) {
		return url.QueryEscape("s:" + cookie.Sign(key, sessionSecret))
	},
	UnsignKey: func(signed string) (key string, ok bool) {
		if len(signed) == 0 {
			return
		}
		unsigned := utils.CopyString(signed)
		if unsigned, err := url.QueryUnescape(unsigned); err == nil {
			if len(unsigned) >= 2 && unsigned[0:2] == "s:" {
				unsigned = unsigned[2:]
				if unsigned, err := cookie.Unsign(unsigned, sessionSecret); err == nil {
					return unsigned, true
				}
			}
		}
		return
	},
	Expiration: time.Hour * 24 * 7,
	Storage: redisstore.New(redisstore.Config{
		Client: redisclient,
	}),
})
// This panic will be catch by the middleware
app.Get("/", func(c *fiber.Ctx) error {
	// Get session from storage
	sess, err := store.Get(c)
	if err != nil {
		panic(err)
	}

	// Get value
	name := sess.Get("name")

	// Set key/value
	sess.Set("name", "john")

	// Get all Keys
	keys := sess.Keys()

	// Delete key
	sess.Delete("name")

	// Destroy session
	if err := sess.Destroy(); err != nil {
		panic(err)
	}

	// Sets a specific expiration for this session
	sess.SetExpiry(time.Second * 2)

	// Save session
	if err := sess.Save(); err != nil {
		panic(err)
	}

	return c.SendString(fmt.Sprintf("Welcome %v", name))
})
```

### Custom Storage/Database

You can use any storage from our [storage](https://github.com/gofiber/storage/) package.

```go
storage := sqlite3.New() // From github.com/gofiber/storage/sqlite3
store := session.New(session.Config{
	Storage: storage,
})
```

To use the the store, see the above example.

## Config

```go
// Config defines the config for middleware.
type Config struct {
	// Allowed session duration
	// Optional. Default value 24 * time.Hour
	Expiration time.Duration

	// Storage interface to store the session data
	// Optional. Default value memory.New()
	Storage fiber.Storage

	// KeyLookup is a string in the form of "<source>:<name>" that is used
	// to extract session id from the request.
	// Possible values: "header:<name>", "query:<name>" or "cookie:<name>"
	// Optional. Default value "cookie:session_id".
	KeyLookup string

	// Domain of the cookie.
	// Optional. Default value "".
	CookieDomain string

	// Path of the cookie.
	// Optional. Default value "".
	CookiePath string

	// Indicates if cookie is secure.
	// Optional. Default value false.
	CookieSecure bool

	// Indicates if cookie is HTTP only.
	// Optional. Default value false.
	CookieHTTPOnly bool

	// Sets the cookie SameSite attribute.
	// Optional. Default value "Lax".
	CookieSameSite string

	// KeyGenerator generates the session key.
	// Optional. Default value utils.UUID
	KeyGenerator func() string

	// Deprecated, please use KeyLookup
	CookieName string

	// Source defines where to obtain the session id
	source      Source

	// The session name
	sessionName string
}
```

## Default Config

```go
var ConfigDefault = Config{
	Expiration:   24 * time.Hour,
	KeyLookup:    "cookie:session_id",
	KeyGenerator: utils.UUID,
}
```
