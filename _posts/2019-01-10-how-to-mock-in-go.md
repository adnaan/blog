---
ID: 92
post_title: How to mock in Go
author: adnaan
post_excerpt: ""
layout: post
permalink: >
  http://adnaan.badr.in/blog/2019/01/10/how-to-mock-in-go/
published: true
post_date: 2019-01-10 20:06:52
---
We often need to simulate or mimic an object to create a deterministic, fast and network-independent object. Such an object is useful while testing. This practice is also known as mocking.

Let's look at a few approaches to mock in Go.

Since database is one of the components which is often mocked, let's look at a stubbed out example for it.
<h2>Example</h2>
So if you have a type <code>User</code>.
<pre><code class="go">...
type User struct {
    ID string
    Name string
}

</code></pre>
and a type <code>Storage</code> which represents a database.
<pre><code class="go">...
type Storage struct {
    db *sql.DB
}

func (s *Storage) CreateUser(user User) error {
    ...
    s.db.Exec(...)
    ...
    return nil
}

</code></pre>
In the above psuedo-code there are two ways to mock the behaviour of <code>Storage</code> without using a real db connection.

The first way is to mock the <code>db</code> object itself. Driver-level mocking is not trivial but it's possible to find packages out there: <a href="https://github.com/DATA-DOG/go-sqlmock">DATA-DOG/go-sqlmock</a>. We won't talk about it in this post.

The second way is to mock the <code>CreateUser</code> method. Let's see the approaches to mock out the methods.
<h2>How to mock</h2>
<h3>1. Using Interfaces</h3>
To mock out the <code>Storage</code> type, we can declare an interface to have a real and a mock implementation.

So instead of creating a <code>type Storage struct</code>, we create a <code>type Storage interface</code>.
<pre><code class="go">...
type Storage interface {
    CreateUser(user User) error
}

</code></pre>
Implement a real storage for the interface <code>Storage</code>.
<pre><code class="go">...
func NewStorage(db *sql.DB) Storage {
    return &amp;defaultStorage{db : db}
}

type defaultStorage struct {
    db *sql.DB
}

func (d *defaultStorage) CreateUser(user User) error {
    ...
    d.db.Exec(...)
    ...
    return nil
}

</code></pre>
and a mock storage.
<pre><code class="go">...
func NewMockStorage() Storage {
    return &amp;mockStorage{users: make(map[int64]User)}
}

type mockStorage struct {
    users map[int64]User
    lastID int64
}

func (m *mockStorage) CreateUser(user User) error {
    ...
    m.users[lastID] = user
    ...
    return nil
}

</code></pre>
Alternatively, one could split this into multiple packages to keep the method signatures same and imports more sensible.
<pre><code class="bash">

pkg/storage/
    mocksql/
    sql/
    storage.go

</code></pre>
<code>sql.go</code>
<pre><code class="go">...
func New(db *sql.DB) Storage {
    return &amp;storage{db: db}
}

type storage struct{
    db *sql.DB
}

func (s *storage) CreateUser(user User) error {
    ...
    s.db.Exec(...)
    ...
    return nil
}

</code></pre>
<code>mocksql/sql.go</code>
<pre><code class="go">...
func New() Storage {
    return &amp;mockStorage{users: make(map[int64]User)}
}

type mockStorage struct {
    users map[int64]User
    lastID int64
}

func (m *mockStorage) CreateUser(user User) error {
    ...
    m.users[lastID] = user
    ...
    return nil
}

</code></pre>
<code>storage.go</code>
<pre><code class="go">...
type Storage interface {
    CreateUser(user User) error
}

func NewSQL(db *sql.DB) Storage {
    return sql.New(db)
}

func NewMockSQL() Storage {
    return mocksql.New()
}

</code></pre>
To import.

<code>user.go</code>
<pre><code class="go">import "github.com/myuser/mypkg/pkg/storage"

...

storage := storage.NewSQL(db)

</code></pre>
<code>user_test.go</code>
<pre><code class="go">import "github.com/myuser/mypkg/pkg/storage"

...

storage := storage.NewMockSQL()

</code></pre>
One could imagine that if the <code>Storage</code> interface has tens of methods or there are several interfaces like it, it could get quite cumbersome to write out the mock implementations for it. Fortunately there's tooling to help out.
<h4>Generating Mocks</h4>
<h5>1. Use an editor plugin</h5>
In  <code>vscode</code> open the command paletter(cmd+shift+p), move cursor on the target stub and run <code>Go: Generate Interface Stubs</code>. Most of the editors supporting Go have this feature integrated.
<h5>2. Use a cli or package</h5>
<a href="https://godoc.org/github.com/stretchr/testify/mock">testify/mock</a>
<a href="https://godoc.org/github.com/golang/mock">golang/mock</a>
<h3>2. Using Functions</h3>
While using interfaces to mock out behaviour is quite alright, it might look too permanent for some projects. Also, one could rather want an approach where the mocking code is completely contained within a test function. There could be two approaches here:
<h4>1. Custom Function Types</h4>
In this approach you design the <code>Storage</code> object to have custom function types.
<pre><code class="go">...
type CreateUserFunc func(user User) error

type Storage struct {
    CreateUserFunc CreateUserFunc
}

</code></pre>
The real implementation would be:
<pre><code class="go">

...
import "github.com/adnaan/mypkg/sqldb"

...
storage := Storage{
    CreateUserFunc: func(user User) error {
      ...
      db := sqldb.Get()
      db.Exec(...)
      ...
      return nil
    }
}

</code></pre>
and use a mock in test code.
<pre><code class="go">

var db = make(map[int]User)
var lastID = 0
...
mockStorage := Storage{
    CreateUserFunc: func(user User) error {
        ...
        db[lastID] = user
        ...
        return nil
    },
}

</code></pre>
This approach needs a database manager package which provides the database handle.
<h4>2. Interfaces and Custom Mock Structs</h4>
In this approach one still has a <code>Storage</code> interface but also implements a mock struct which holds mocked equivalents of the interface methods.
<pre><code class="go">...
type Storage interface {
    CreateUser(user User) error
}

func New(db *sql.DB) Storage {
    return &amp;storage{db: db}
}

type storage struct {
    db *sql.DB
}

func (s *storage) CreateUser(user User) error {
    ...
    s.db.Exec(...)
    ...
    return nil
}

...

// test code

type StorageMock struct {
    CreateUserFunc func(user User) error
}

func (s *StorageMock) CreateUser(user User) error {
    return mock.CreateUserFunc(user)
}

// test code

...
var db = make(map[int]User)
var lastID = 0

...

mockStorage := &amp;StorageMock{
    CreateUserFunc: func(user User) error {
        ...
        db[lastID] = user
        ...
        return nil
    }
}

</code></pre>
There is tooling available to generate <code>StorageMock</code> via <a href="https://github.com/matryer/moq">moq</a>
<h1>Conclusion</h1>
Depending on a project's complexity, scope and use cases either one of the above mocking approaches could fit. One of the considerations could be other behaviours(apart from mocking) needed for a type. For e.g. the <code>Storage</code> type could have multiple database backends implementations like <code>inmem, postgres, mysql etc.</code>

Mock packages in <a href="https://golanglibs.com/search?q=mock">golanglibs.com</a>.

Feedback is welcome on <a href="https://twitter.com/adnaanx">Twitter</a> too.

Mock well and prosper ✌ .