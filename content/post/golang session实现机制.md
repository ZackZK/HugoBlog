## session使用示例

```go
package main

import (
	"github.com/gin-contrib/sessions"
	"github.com/gin-contrib/sessions/cookie"
	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()  //--> 获取gin默认egine
	store := cookie.NewStore([]byte("secret"))　//定义session存储方式
	r.Use(sessions.Sessions("mysession", store))

	r.GET("/hello", func(c *gin.Context) {
		session := sessions.Default(c)

		if session.Get("hello") != "world" {
			session.Set("hello", "world")
			session.Save()
		}

		c.JSON(200, gin.H{"hello": session.Get("hello")})
	})
	r.Run(":8000")
}
```



**分析：**

```go
sessions.Sessions("mysession", store)

func Sessions(name string, store Store) gin.HandlerFunc {
	return func(c *gin.Context) {
		s := &session{name, c.Request, store, nil, false, c.Writer}
		c.Set(DefaultKey, s)
		defer context.Clear(c.Request)
		c.Next()
	}
}
```

创建一个以name命名，store方式存储的session，并返回一个闭包函数，该函数在每个HTTP请求处理前被调用。

```go
s := &session{name, c.Request, store, nil, false, c.Writer}
```

创建一个session对象,  并使用c.Set以key-value形式存储在gin.Context中。

github.com/gin-contrib/sessions/sessions.go中定义并实现了session结构

```go
type session struct {
	name    string
	request *http.Request
	store   Store
	session *sessions.Session
	written bool
	writer  http.ResponseWriter
}

func (s *session) Get(key interface{}) interface{} {
	return s.Session().Values[key]
}

func (s *session) Set(key interface{}, val interface{}) {
	s.Session().Values[key] = val
	s.written = true
}

func (s *session) Delete(key interface{}) {
	delete(s.Session().Values, key)
	s.written = true
}

func (s *session) Clear() {
	for key := range s.Session().Values {
		s.Delete(key)
	}
}

func (s *session) AddFlash(value interface{}, vars ...string) {
	s.Session().AddFlash(value, vars...)
	s.written = true
}

func (s *session) Flashes(vars ...string) []interface{} {
	s.written = true
	return s.Session().Flashes(vars...)
}

func (s *session) Options(options Options) {
	s.Session().Options = options.ToGorillaOptions()
}

func (s *session) Save() error {
	if s.Written() {
		e := s.Session().Save(s.request, s.writer)
		if e == nil {
			s.written = false
		}
		return e
	}
	return nil
}

func (s *session) Session() *sessions.Session {
	if s.session == nil {
		var err error
		s.session, err = s.store.Get(s.request, s.name)
		if err != nil {
			log.Printf(errorFormat, err)
		}
	}
	return s.session
}

func (s *session) Written() bool {
	return s.written
}
```

session重点是set和get方法，如常用方式:

```go
session := sessions.Default(c)

if session.Get("hello") != "world" {
	session.Set("hello", "world")
	session.Save()
}
```

sessions.Default(c)从gin.Context中获取session(session就是从闭包函数中设置的)

```go
// shortcut to get session
func Default(c *gin.Context) Session {
	return c.MustGet(DefaultKey).(Session)
}

func (s *session) Session() *sessions.Session {
	if s.session == nil {
		var err error
		s.session, err = s.store.Get(s.request, s.name)
		if err != nil {
			log.Printf(errorFormat, err)
		}
	}
	return s.session
}
```

session.Set()函数实现

```go
func (s *session) Set(key interface{}, val interface{}) {
	s.Session().Values[key] = val
	s.written = true
}
```

首先s.Session()返回顶层存储的session　key-value信息



session.Get()函数实现:

```go
func (s *session) Get(key interface{}) interface{} {
	return s.Session().Values[key]
}

s.session, err = s.store.Get(s.request, s.name)
```

以cookie　store为例子,　看怎么实现:

```go
store := cookie.NewStore([]byte("secret"))　//定义session存储方式

//home/zhangke/workspace/src/sfauth/src/github.com/gin-contrib/sessions/cookie/cookie.go
type store struct {
	*gsessions.CookieStore
}

func (c *store) Options(options sessions.Options) {
	c.CookieStore.Options = options.ToGorillaOptions()
}

func NewStore(keyPairs ...[]byte) Store {
	return &store{gsessions.NewCookieStore(keyPairs...)}
}

//home/zhangke/workspace/src/sfauth/src/github.com/gorilla/sessions/store.go
func NewCookieStore(keyPairs ...[]byte) *CookieStore {
	cs := &CookieStore{
		Codecs: securecookie.CodecsFromPairs(keyPairs...),
		Options: &Options{
			Path:   "/",
			MaxAge: 86400 * 30,
		},
	}

	cs.MaxAge(cs.Options.MaxAge)
	return cs
}

//home/zhangke/workspace/src/sfauth/src/github.com/gorilla/securecookie/securecookie.go
func CodecsFromPairs(keyPairs ...[]byte) []Codec {
	codecs := make([]Codec, len(keyPairs)/2+len(keyPairs)%2)
	for i := 0; i < len(keyPairs); i += 2 {
		var blockKey []byte
		if i+1 < len(keyPairs) {
			blockKey = keyPairs[i+1]
		}
		codecs[i/2] = New(keyPairs[i], blockKey)
	}
	return codecs
}

```

store实例主要存储了session内容加密方式，有效期等其他配置信息。



store.Get函数返回具体:

```go
//home/zhangke/workspace/src/sfauth/src/github.com/gorilla/sessions/store.go

// Get returns a session for the given name after adding it to the registry.
//
// It returns a new session if the sessions doesn't exist. Access IsNew on
// the session to check if it is an existing session or a new one.
//
// It returns a new session and an error if the session exists but could
// not be decoded.
func (s *CookieStore) Get(r *http.Request, name string) (*Session, error) {
	return GetRegistry(r).Get(s, name)
}



//home/zhangke/workspace/src/sfauth/src/github.com/gorilla/sessions/sessions.go

// contextKey is the type used to store the registry in the context.
type contextKey int

// registryKey is the key used to store the registry in the context.
const registryKey contextKey = 0

// GetRegistry returns a registry instance for the current request.
func GetRegistry(r *http.Request) *Registry {
	var ctx = r.Context()
	registry := ctx.Value(registryKey)
	if registry != nil {
		return registry.(*Registry)
	}
	newRegistry := &Registry{
		request:  r,
		sessions: make(map[string]sessionInfo),
	}
	*r = *r.WithContext(context.WithValue(ctx, registryKey, newRegistry))
	return newRegistry
}

// Registry stores sessions used during a request.
type Registry struct {
	request  *http.Request
	sessions map[string]sessionInfo
}

// Get registers and returns a session for the given name and session store.
//
// It returns a new session if there are no sessions registered for the name.
func (s *Registry) Get(store Store, name string) (session *Session, err error) {
	if !isCookieNameValid(name) {
		return nil, fmt.Errorf("sessions: invalid character in cookie name: %s", name)
	}
	if info, ok := s.sessions[name]; ok {
		session, err = info.s, info.e
	} else {
		session, err = store.New(s.request, name)
		session.name = name
		s.sessions[name] = sessionInfo{s: session, e: err}
	}
	session.store = store
	return
}
```

GetRegistry返回当前请求中registry实例，如果之前没有设置过，则新分配一个。通过

```go
	*r = *r.WithContext(context.WithValue(ctx, registryKey, newRegistry))
```

设置。



Registry.Get函数返回当前请求对应的session，如果没有，则通过 store.New(s.request, name)新分配一个. 这里是主要的逻辑。以CookieStore为例:

```go
///home/zhangke/workspace/src/sfauth/src/github.com/gorilla/sessions/store.go
// New returns a session for the given name without adding it to the registry.
//
// The difference between New() and Get() is that calling New() twice will
// decode the session data twice, while Get() registers and reuses the same
// decoded session after the first call.
func (s *CookieStore) New(r *http.Request, name string) (*Session, error) {
	session := NewSession(s, name)
	opts := *s.Options
	session.Options = &opts
	session.IsNew = true
	var err error
    //-----------------重点 r.Cookie(name)取到当前请求中对应name的Cookie，如果为空，表示是新的cookie，否则使用之前的加密方式解出cookie内容，存放到session.Values中
	if c, errCookie := r.Cookie(name); errCookie == nil {
		err = securecookie.DecodeMulti(name, c.Value, &session.Values,
			s.Codecs...)
		if err == nil {
			session.IsNew = false
		}
	}
	return session, err
}

//home/zhangke/workspace/src/sfauth/src/github.com/gorilla/sessions/sessions.go
// NewSession is called by session stores to create a new session instance.
func NewSession(store Store, name string) *Session {
	return &Session{
		Values:  make(map[interface{}]interface{}),
		store:   store,
		name:    name,
		Options: new(Options),
	}
}

// Session stores the values and optional configuration for a session.
type Session struct {
	// The ID of the session, generated by stores. It should not be used for
	// user data.
	ID string
	// Values contains the user-data for the session.
	Values  map[interface{}]interface{}
	Options *Options
	IsNew   bool
	store   Store
	name    string
}

```

所以

github.com/gin-contrib/sessions/sessions.go

func (s *session) Get(key interface{}) interface{} {
	return s.Session().Values[key]
}

中从store中获取到对应的Session对象，然后使用Values属性获取对应的session参数值。



Set函数类似，同时设置writen标志。

func (s *session) Set(key interface{}, val interface{}) {
	s.Session().Values[key] = val
	s.written = true
}



现在看session怎么Save, 思路Save就是将对应Session的key value值存入对应store的存储介质中。

```go
func (s *session) Save() error {
	if s.Written() {
		e := s.Session().Save(s.request, s.writer)
		if e == nil {
			s.written = false
		}
		return e
	}
	return nil
}

///home/zhangke/workspace/src/sfauth/src/github.com/gorilla/sessions/sessions.go
// Save is a convenience method to save this session. It is the same as calling
// store.Save(request, response, session). You should call Save before writing to
// the response or returning from the handler.
func (s *Session) Save(r *http.Request, w http.ResponseWriter) error {
    //调用session对应store的Save方法
	return s.store.Save(r, w, s)
}

//以CookieStore为例:
// Save adds a single session to the response.
func (s *CookieStore) Save(r *http.Request, w http.ResponseWriter,
	session *Session) error {
    //使用CookieStore的加密方式加密
	encoded, err := securecookie.EncodeMulti(session.Name(), session.Values,
		s.Codecs...)
	if err != nil {
		return err
	}
    //设置到头部
	http.SetCookie(w, NewCookie(session.Name(), encoded, session.Options))
	return nil
}

```

从上面可以看到, CookieStore是将所有数据经过加密放在传给客户端的cookie中，而RedisStore

```go
///home/zhangke/workspace/src/sfauth/src/github.com/boj/redistore/redistore.go
// Save adds a single session to the response.
func (s *RediStore) Save(r *http.Request, w http.ResponseWriter, session *sessions.Session) error {
	// Marked for deletion.
	if session.Options.MaxAge <= 0 {
		if err := s.delete(session); err != nil {
			return err
		}
		http.SetCookie(w, sessions.NewCookie(session.Name(), "", session.Options))
	} else {
		// Build an alphanumeric key for the redis store.
		if session.ID == "" {
			session.ID = strings.TrimRight(base32.StdEncoding.EncodeToString(securecookie.GenerateRandomKey(32)), "=")
		}
		if err := s.save(session); err != nil {
			return err
		}
		encoded, err := securecookie.EncodeMulti(session.Name(), session.ID, s.Codecs...)
		if err != nil {
			return err
		}
		http.SetCookie(w, sessions.NewCookie(session.Name(), encoded, session.Options))
	}
	return nil
}

// save stores the session in redis.
func (s *RediStore) save(session *sessions.Session) error {
	b, err := s.serializer.Serialize(session)
	if err != nil {
		return err
	}
	if s.maxLength != 0 && len(b) > s.maxLength {
		return errors.New("SessionStore: the value to store is too big")
	}
	conn := s.Pool.Get()
	defer conn.Close()
	if err = conn.Err(); err != nil {
		return err
	}
	age := session.Options.MaxAge
	if age == 0 {
		age = s.DefaultMaxAge
	}
	_, err = conn.Do("SETEX", s.keyPrefix+session.ID, age, b)
	return err
}
```

redis只是将session-id放在cookie中，将序列化的数据内容以sessionid为key存入redis中。



