# Snippetbox
 A follow along project in the [Let's go](https://lets-go.alexedwards.net/) book by Alex Edwards

## Content
* [Project Structure](#project-structure)
* [Logging](#logging)
* [Dependency Injection](#dependency-injection)
* [Errors Handling](#errors-handling)
* [Database](#database)
* [Middleware](#middleware)
* [Routing](#routing)
* [Process Forms](#process-forms)
* [Session Manager](#session-manager)
* [TLS certificate](#tls-certificate)
* [Authentication](#authentication)
* [Request Context](#request-context)
* [File embedding and generics](#file-embedding-and-generics)
* [Testing](#testing)


### Project Structure
### Logging
<p>Use log.New() to create new custom loggers with information message "INFO" output to standard out(stdout) and prefix error message "ERROR" output to standard error(stderr). log.Ldate, log.Ltime, log.Lshortfile are flags to indicate what additional information to include(date, time, relevant filename and line number)</p>

```go
 infoLog := log.New(os.Stdout, "INFO\t", log.Ldate|log.Ltime)
 errorLog := log.New(os.Stderr, "ERROR\t", log.Ldate|log.Ltime|log.Lshortfile)
```

<p>A benefit of this method is that the application and logging are decoupled. Application itself isn’t concerned with the routing or storage of the logs, and that can make it easier to manage the logs differently depending on the environment.</p>
<p>During development, it’s easy to view the log output because the standard streams are displayed in the terminal.</p>
<p>In staging or production environments, can redirect the streams to a final destination for viewing and archival. This destination could be on-disk files, or a logging service such as Splunk.</p>
<p> By default, if Go’s HTTP server encounters an error it will log it using the standard logger. For consistency it’d be better to use new errorLog logger instead.</p>

```go
 srv := &http.Server{
  Addr: *addr,
  ErrorLog: errorLog,
  Handler: mux,
 }
 err := srv..ListenAndServe()
```

<p>Avoid using the Panic() and Fatal() variations outside of main() function — it’s good practice to return errors instead, and only panic or exit directly from main().</p>

### Dependency Injection
<p>For applications where all your handlers are in the same package, a neat way to inject dependencies is to put them into a custom application struct, and then define your handler functions as methods against application.</p>

```go
 type application struct {
  errorLog *log.Logger
  infoLog *log.Logger
 }
```

<p>And then update handlers function so that they become methods. Eg:</p>

```go
 func (app *application) home(w http.ResponseWriter, r *http.Request){...}
```

<p>Initialize a new instance of our application struct, containing the dependencies in main()</p>

```go
 app := &application{
  errorLog: errorLog,
  infoLog: infoLog,
 }
```

#### Alternative approach
<p>Create a config package exporting an Application struct and have handler functions close over this to form a closure.</p>

```go
 func main() {
  app := &config.Application{
   ErrorLog: log.New(os.Stderr, "ERROR\t", log.Ldate|log.Ltime|log.Lshortfile)
  } 
  mux.Handle("/", examplePackage.ExampleHandler(app))
 }
 func ExampleHandler(app *config.Application) http.HandlerFunc {
  return func(w http.ResponseWriter, r *http.Request) {
   ...
   ts, err := template.ParseFiles(files...)
   if err != nil {
    app.ErrorLog.Print(err.Error())
    http.Error(w, "Internal Server Error", 500)
    return
   } 
   ...
  }
 }
```

### Errors Handling

Centralized error handling with helpers.go

```go
func (app *application) serverError(w http.ResponseWriter, err error) {
	trace := fmt.Sprintf("%s\n%s", err.Error(), debug.Stack())
	app.errorLog.Output(2, trace)

	http.Error(w, http.StatusText(http.StatusInternalServerError), http.StatusInternalServerError)
}
```

<p> debug.Stack() function to get a stack trace for the current goroutine and append it to the log message. Being able to see the execution path
of the application via the stack trace can be helpful when you’re trying to debug errors </p>

```go
func (app *application) clientError(w http.ResponseWriter, status int) {
	http.Error(w, http.StatusText(status), status)
}
```

<p> http.StatusText() function to automatically generate a human-friendly text representation of a given HTTP status code. </p>
<p>Example:</p>

```go
func (app *application) notFound(w http.ResponseWriter) {
	app.clientError(w, http.StatusNotFound)
}
```

```go
func (app *application) newTemplateData(r *http.Request) *templateData {
	return &templateData{
		CurrentYear:     time.Now().Year(),
		Flash:           app.sessionManager.PopString(r.Context(), "flash"),
		IsAuthenticated: app.isAuthenticated(r),
		CSRFToken:       nosurf.Token(r),
	}
}
```

<p>Returns a pointer to a templateData with struct initialized with the current year, flash message using [session manager](#session-manager), a boolean value indicating whether the user is <del> [authenticated](#authentication) </del> and a <del> [CSRF token](#tls-certificate) </del></p>

```go
func (app *application) render(w http.ResponseWriter, status int, page string, data *templateData) {
	ts, ok := app.templateCache[page]
	if !ok {
		err := fmt.Errorf("the template %s does not exist", page)
		app.serverError(w, err)
		return
	}

	buf := new(bytes.Buffer)

	err := ts.ExecuteTemplate(buf, "base", data)
	if err != nil {
		app.serverError(w, err)
		return
	}

	w.WriteHeader(status)

	buf.WriteTo(w)
}
```

<p>Checks if the requested template exists in the templateCache. If it doesn't exist, it returns an error. Otherwise, it executes the template by passing it the data and writes the result to the http.ResponseWriter</p>

```go
func (app *application) decodePostForm(r *http.Request, dst any) error {
	err := r.ParseForm()
	if err != nil {
		return err
	}

	err = app.formDecoder.Decode(dst, r.PostForm)
	if err != nil {

		var invalidDecoderError *form.InvalidDecoderError

		if errors.As(err, &invalidDecoderError) {
			panic(err)
		}

		return err
	}

	return nil
}
```

<p>Calls r.ParseForm() on the current <del> [request](#request-context) </del>, Calls app.formDecoder.Decode() to unpack the HTML form data to a target destination, Checks for a form.InvalidDecoderError error and triggers a panic if we ever see it.</p>

```go
func (app *application) isAuthenticated(r *http.Request) bool {
	isAuthenticated, ok := r.Context().Value(isAuthenticatedContextKey).(bool)
	if !ok {
		return false
	}

	return isAuthenticated
}
```

<p>Check whether or not the request is coming from an authenticated (logged in) user </p>

### Database

```go
db, err := sql.Open("mysql", "web:pass@/snippetbox?parseTime=true")
if err != nil {
...
}
```

<p>The first parameter to sql.Open() is the driver name and the second parameter is the data source name (sometimes also called a connection string or DSN) which describes how to connect to your database.</p>
<p>sql.Open() function initializes a new sql.DB object, which is essentially a pool of database connections with user: web, password: pass, database name(models): snippetbox, parseTime=true part of the DSN above is a driver-specific parameter which instructs our driver to convert SQL TIME and DATE fields to Go time.Time objects.</p>
<p>This isn’t a database connection — it’s a pool of many connections. Go manages the connections in this pool as needed, automatically opening and closing connections to the database via the driver. The connection pool is safe for concurrent access.</p>
<p>It’s normal to initialize the connection pool in main() function and then pass the pool to handlers. </p>
<p>Remember to use</p>

```go
defer db.Close()
```

<p>so that the connection pool is closed before the main() function exits.</p>
<p>Should use openDB() to wraps sql.Open() and returns a sql.DB connection pool for a given DSN</p>

```go
func openDB(dsn string) (*sql.DB, error) {
db, err := sql.Open("mysql", dsn)
if err != nil {
return nil, err
} i
f err = db.Ping(); err != nil {
return nil, err
} r
eturn db, nil
}
```

#### Models

##### Snippet Model

```go
type SnippetModelInterface interface {
	Insert(title string, content string, expires int) (int, error)
	Get(id int) (*Snippet, error)
	Latest() ([]*Snippet, error)
}
type Snippet struct {
	ID      int
	Title   string
	Content string
	Created time.Time
	Expires time.Time
}

type SnippetModel struct {
	DB *sql.DB
}
```

<p>SnippetModelInterface is use latter in <del> [testing](#testing) </del></p>
<p>To use this model in handlers we need to establish a new SnippetModel struct in main() then inject it as a dependency via the application struct like below</p>

```go
type application struct {
errorLog *log.Logger
infoLog *log.Logger
snippets *models.SnippetModel
}
```
##### User Model

```go
type UserModelInterface interface {
	Insert(name, email, password string) error
	Authenticate(email, password string) (int, error)
	Exists(id int) (bool, error)
}
type User struct {
	ID             int
	Name           string
	Email          string
	HashedPassword []byte
	Created        time.Time
}

type UserModel struct {
	DB *sql.DB
}
```

<p>UserModelInterface is use latter in <del> [testing](#testing) </del>.</p>
<p>User Model is use for <del> [authentication](#authentication) </del> purpose.</p>

##### Model Methods

<p>Go provides three different methods for executing database queries</p>
* DB.Query() is used for SELECT queries which return multiple rows
* DB.QueryRow() is used for SELECT queries which return a single row
* DB.Exec() is used for statements which don’t return rows (like INSERT and DELETE)

###### Snippet Model Methods

```go
func (m *SnippetModel) Insert(title string, content string, expires int) (int, error) {
	stmt := `INSERT INTO snippets (title, content, created, expires)
    VALUES(?, ?, UTC_TIMESTAMP(), DATE_ADD(UTC_TIMESTAMP(), INTERVAL ? DAY))`

	result, err := m.DB.Exec(stmt, title, content, expires)
	if err != nil {
		return 0, err
	}

	id, err := result.LastInsertId()
	if err != nil {
		return 0, err
	}

	return int(id), nil
}
```
<p>stmt is the SQL statement we want to execute, Use the Exec() method on the embedded connection pool to execute the statement then use the LastInsertId() method on the result to get the ID of our newly inserted record in the snippets table, The ID returned has the type int64, so we convert it to an int type</p>
<p>In the code above we constructed our SQL statement using placeholder parameters, where ? acted as a placeholder for the data we want to insert. </p>
<p>This help avoid SQL injection attacks from any untrusted user-provided input.</p>

```go
func (m *SnippetModel) Get(id int) (*Snippet, error) {
	stmt := `SELECT id, title, content, created, expires FROM snippets
    WHERE expires > UTC_TIMESTAMP() AND id = ?`

	row := m.DB.QueryRow(stmt, id)

	s := &Snippet{}

	err := row.Scan(&s.ID, &s.Title, &s.Content, &s.Created, &s.Expires)
	if err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return nil, ErrNoRecord
		} else {
			return nil, err
		}
	}

	return s, nil
}
```
<p>QueryRow() method returns a pointer to a sql.Row object which holds the result from the database</p>
<p>Use row.Scan() to copy the values from each field in sql.Row to the corresponding field in the Snippet struct. Notice that the arguments to row.Scan are *pointers* to the place you want to copy the data into, and the number of arguments must be exactly the same as the number of columns returned by your statement.</p>
<p>In errors.go</p>

```go
var ErrNoRecord = errors.New("models: no matching record found")
```

<p>Returning the ErrNoRecord error from our SnippetModel.Get() method, instead of sql.ErrNoRows directly</p>
<p>The reason is to help encapsulate the model completely, so that our application isn’t concerned with the underlying datastore or reliant on datastore-specific errors for its behavior</p>

```go
func (m *SnippetModel) Latest() ([]*Snippet, error) {
	stmt := `SELECT id, title, content, created, expires FROM snippets
    ORDER BY id DESC LIMIT 10`

	rows, err := m.DB.Query(stmt)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	snippets := []*Snippet{}

	for rows.Next() {
		s := &Snippet{}

		err = rows.Scan(&s.ID, &s.Title, &s.Content, &s.Created, &s.Expires)
		if err != nil {
			return nil, err
		}

		snippets = append(snippets, s)
	}

	if err = rows.Err(); err != nil {
		return nil, err
	}

	return snippets, nil
}
```

<p>Query() method returns a sql.Rows resultset containing the result </p>
<p>defer rows.Close() to ensure the sql.Rows resultset is always properly closed before the Latest() method returns.</p>
<p>Use rows.Next to iterate through the rows in the resultset. This prepares the first (and then each subsequent) row to be acted on by the rows.Scan() method. If iteration over all the rows completes then the resultset automatically closes itself and frees-up the underlying database connection.</p>

###### User Model Methods

```go
func (m *UserModel) Insert(name, email, password string) error {
	hashedPassword, err := bcrypt.GenerateFromPassword([]byte(password), 12)
	if err != nil {
		return err
	}

	stmt := `INSERT INTO users (name, email, hashed_password, created)
    VALUES(?, ?, ?, UTC_TIMESTAMP())`
 // Use the Exec() method to insert the user details and hashed password
 // into the users table.
	_, err = m.DB.Exec(stmt, name, email, string(hashedPassword))
	if err != nil {
  // If this returns an error, we use the errors.As() function to check
  // whether the error has the type *mysql.MySQLError. If it does, the
  // error will be assigned to the mySQLError variable. We can then check
  // whether or not the error relates to our users_uc_email key by
  // checking if the error code equals 1062 and the contents of the error
  // message string. If it does, we return an ErrDuplicateEmail error.
		var mySQLError *mysql.MySQLError
		if errors.As(err, &mySQLError) {
			if mySQLError.Number == 1062 && strings.Contains(mySQLError.Message, "users_uc_email") {
				return ErrDuplicateEmail
			}
		}
		return err
	}

	return nil
}
```

<p>It’s good practice — well, essential, really — to store a one-way hash of the password, derived with a computationally expensive key-derivation function such as Argon2, scrypt or bcrypt.</p>
<p>A plus-point of the bcrypt implementation specifically is that it includes helper functions specifically designed for hashing and checking passwords.</p>
<p>bcrypt.GenerateFromPassword() function which lets us create a hash of a given plain-text password like so : hash, err := bcrypt.GenerateFromPassword([]byte("my plain text password"), 12) with '12' indicates the cost, which means that that 4096 (2^12) bcrypt iterations will be used to generate the password hash. Cost can be an integer between 4 and 31.</p>
<p>On the flip side, we can check that a plain-text password matches a particular hash using the bcrypt.CompareHashAndPassword() function like so:</p>

```go
hash := []byte("$2a$12$NuTjWXm3KKntReFwyBVHyuf/to.HEwTy.eS206TNfkGfr6GzGJSWG")
err := bcrypt.CompareHashAndPassword(hash, []byte("my plain text password"))
```

```go
func (m *UserModel) Authenticate(email, password string) (int, error) {
	var id int
	var hashedPassword []byte

	stmt := "SELECT id, hashed_password FROM users WHERE email = ?"

	err := m.DB.QueryRow(stmt, email).Scan(&id, &hashedPassword)
	if err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return 0, ErrInvalidCredentials
		} else {
			return 0, err
		}
	}

	err = bcrypt.CompareHashAndPassword(hashedPassword, []byte(password))
	if err != nil {
		if errors.Is(err, bcrypt.ErrMismatchedHashAndPassword) {
			return 0, ErrInvalidCredentials
		} else {
			return 0, err
		}
	}

	return id, nil
}
```

<p>This is part of <del> [authentication](#authentication) </del>.</p>
<p>Quick summary: first we retrieve the id and hashed password associated with the given email, scan databse with given email then check whether the hashed password and plain-text password provided match, if they don't, we return the ErrInvalidCredentials error.</p>

```go
func (m *UserModel) Exists(id int) (bool, error) {
	var exists bool

	stmt := "SELECT EXISTS(SELECT true FROM users WHERE id = ?)"

	err := m.DB.QueryRow(stmt, id).Scan(&exists)
	return exists, err
}
```

<p>Check if user existed by id</p>


### Middleware

<p>When building a web application there’s probably some shared functionality that you want to use for many (or even all) HTTP requests. </p>
<p>A common way of organizing this shared functionality is to set it up as middleware. This is essentially some self-contained code which independently acts on a request before or after normal application handlers.</p>

#### How middleware work

<p>Currently, in our application, when our server receives a new HTTP request it calls the servemux’s ServeHTTP() method. This looks up the relevant handler based on the request URL path, and in turn calls that handler’s ServeHTTP() method.</p>
<p>The basic idea of middleware is to insert another handler into this chain. The middleware handler executes some logic, like logging a request, and then calls the ServeHTTP() method of the next handler in the chain.</p>


#### Middleware pattern

<p>The standard pattern for creating middleware looks like this:</p>

```go
func myMiddleware(next http.Handler) http.Handler {
fn := func(w http.ResponseWriter, r *http.Request) {
// TODO: Execute our middleware logic here...
next.ServeHTTP(w, r)
}
return http.HandlerFunc(fn)
}
```

* The myMiddleware() function is essentially a wrapper around the next handler.
* It establishes a function fn which closes over the next handler to form a closure. When fn is run it executes our middleware logic and then transfers control to the next handler by calling it’s ServeHTTP() method.
* fn will always have access to the next variable.
* We then convert this closure to a http.Handler and return it using the http.HandlerFunc() adapter.

<p>Simplifying the middleware:</p>

```go
func myMiddleware(next http.Handler) http.Handler {
return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
// TODO: Execute our middleware logic here...
next.ServeHTTP(w, r)
})
}
```

<p>In any middleware handler, code which comes before next.ServeHTTP() will be executed on the way down the chain, and any code after next.ServeHTTP() — or in a deferred function — will be executed on the way back up.</p>

```go
func myMiddleware(next http.Handler) http.Handler {
return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
// Any code here will execute on the way down the chain.
next.ServeHTTP(w, r)
// Any code here will execute on the way back up the chain.
})
}
```
<p>If you call return in your middleware function before you call next.ServeHTTP(), then the chain will stop being executed and control will flow back upstream.</p>

```go
func myMiddleware(next http.Handler) http.Handler {
return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
// If the user isn't authorized send a 403 Forbidden status and
// return to stop executing the chain.
if !isAuthorized(r) {
w.WriteHeader(http.StatusForbidden)
return
} // Otherwise, call the next handler in the chain.
next.ServeHTTP(w, r)
})
}
```

#### Positioning the middleware

<p>If you position your middleware before the servemux in the chain then it will act on every request that your application receives.</p>
  myMiddleware → servemux → application handler
<p>A good example of where this would be useful is middleware to log requests — as that’s typically something you would want to do for all requests.</p>
<p>If you position the middleware after the servemux in the chain — by wrapping a specific application handler. This would cause your middleware to only be executed for a specific route.</p>
  servemux → myMiddleware → application handler
<p>An example of this would be something like authorization middleware, which you may only want to run on specific routes.</p>

#### Set up middleware

##### Security headers

```go
func secureHeaders(next http.Handler) http.Handler {
return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
// Note: This is split across multiple lines for readability. You don't
// need to do this in your own code.
w.Header().Set("Content-Security-Policy",
"default-src 'self'; style-src 'self' fonts.googleapis.com; font-src fonts.gstatic.com")
w.Header().Set("Referrer-Policy", "origin-when-cross-origin")
w.Header().Set("X-Content-Type-Options", "nosniff")
w.Header().Set("X-Frame-Options", "deny")
w.Header().Set("X-XSS-Protection", "0")
next.ServeHTTP(w, r)
})
}
```

<p>Because we want this middleware to act on every request that is received, we’ll need the secureHeaders middleware function to wrap our servemux so our routes should return secureHeaders(mux)</p>
<p>Flow of control</p>
  secureHeaders → servemux → application handler → servemux → secureHeaders

##### Request logging

```go
func (app *application) logRequest(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		app.infoLog.Printf("%s - %s %s %s", r.RemoteAddr, r.Proto, r.Method, r.URL.RequestURI())

		next.ServeHTTP(w, r)
	})
}
```

<p>We’re implementing the middleware as a method on application</p>
<p>Flow of control</p>
  logRequest ↔ secureHeaders ↔ servemux ↔ application handler
<p>Our routes should return app.logRequest(secureHeaders(mux))</p>

##### Panic recovery

<p>Following a panic our server will log a stack trace to the server error log, unwind the stack for the affected goroutine (calling any deferred functions along the way) and close the underlying HTTP connection. But it won’t terminate the application, so importantly, any panic in your handlers won’t bring down your server.</p>

```go
func (app *application) recoverPanic(next http.Handler) http.Handler {
return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
// Create a deferred function (which will always be run in the event
// of a panic as Go unwinds the stack).
defer func() {
// Use the builtin recover function to check if there has been a
// panic or not. If there has...
if err := recover(); err != nil {
// Set a "Connection: close" header on the response.
w.Header().Set("Connection", "close")
// Call the app.serverError helper method to return a 500
// Internal Server response.
app.serverError(w, fmt.Errorf("%s", err))
}
}()
next.ServeHTTP(w, r)
})
}
```

<p>Setting the Connection: Close header on the response acts as a trigger to make Go’s HTTP server automatically close the current connection after a response has been sent. It also informs the user that the connection will be closed.</p>
<p>This should be the first thing in our chain to be executed (so that it covers panics in all subsequent middleware and handlers) so our routes should return app.recoverPanic(app.logRequest(secureHeaders(mux)))</p>
<p>It’s important to realise that our middleware will only recover panics that happen in the same goroutine that executed the recoverPanic() middleware.</p>
<p>So, if you are spinning up additional goroutines from within your web application and there is any chance of a panic, you must make sure that you recover any panics from within those too:</p>

```go
func myHandler(w http.ResponseWriter, r *http.Request) {
...
// Spin up a new goroutine to do some background processing.
go func() {
defer func() {
if err := recover(); err != nil {
log.Print(fmt.Errorf("%s\n%s", err, debug.Stack()))
}
}()
doSomeBackgroundProcessing()
}()
w.Write([]byte("OK"))
}
```

#### Manage middleware/handler chains

<p>Using the justinas/alice package makes it easy to create composable, reusable, middleware chains </p>
<p>It allows to rewrite a handler chain like this:</p>
  return myMiddleware1(myMiddleware2(myMiddleware3(myHandler)))
<p>to this:</p>
  return alice.New(myMiddleware1, myMiddleware2, myMiddleware3).Then(myHandler)
<p>But the real power lies in the fact that you can use it to create middleware chains that can be assigned to variables, appended to, and reused. For example:</p>

```go
myChain := alice.New(myMiddlewareOne, myMiddlewareTwo)
myOtherChain := myChain.Append(myMiddleware3)
return myOtherChain.Then(myHandler)
```

<p>Update our routes:</p>

```go
// Create a middleware chain containing our 'standard' middleware
// which will be used for every request our application receives.
standard := alice.New(app.recoverPanic, app.logRequest, secureHeaders)
// Return the 'standard' middleware chain followed by the servemux.
return standard.Then(mux)
```


### Routing
### Process Forms
### Session Manager
### TLS certificate
### Authentication
### Request Context
### File embedding and Generics
### Testing
