# Snippetbox
 A follow along project in the ["Let's go"](https://lets-go.alexedwards.net/) book by Alex Edwards

## Content
<ul>
    <li><a href="#project-structure">Project Structure</a></li>
    <li><a href="logging">Logging</a></li>
    <li><a href="#dependency-injection">Dependency Injection</a></li>
    <li><a href="#errors-handling">Errors Handling</a></li>
    <li><a href="#database">Database</a></li>
    <li><a href="#middleware">Middleware</a></li>
    <li><a href="#routing">Routing</a></li>
    <li><a href="#process-forms">Process Forms</a></li>
    <li><a href="#session-manager">Session Manager</a></li>
    <li><a href="#tls-certificate">TLS certificate</a></li>
    <li><a href ="#authentication">Authentication</a></li>
    <li><a href ="#request-context">Request Context</a></li>
    <li><a href ="#file-embedding-and-generics">File embedding and generics</a></li>
    <li><a href ="#testing">Testing</a></li>
</ul>

### Project Structure
### Logging
<p>Use log.New() to create new custom loggers with information message "INFO" output to standard out(stdout) and prefix error message "ERROR" output to standard error(stderr). log.Ldate, log.Ltime, log.Lshortfile are flags to indicate what additional information to include(date, time, relevant filename and line number)</p>
``` go
 infoLog := log.New(os.Stdout, "INFO\t", log.Ldate|log.Ltime)
 errorLog := log.New(os.Stderr, "ERROR\t", log.Ldate|log.Ltime|log.Lshortfile)
```
<p>A benefit of this method is that the application and logging are decoupled. Application itself isn’t concerned with the routing or storage of the logs, and that can make it easier to manage the logs differently depending on the environment.</p>
<p>During development, it’s easy to view the log output because the standard streams are displayed in the terminal.</p>
<p>In staging or production environments, can redirect the streams to a final destination for viewing and archival. This destination could be on-disk files, or a logging service such as Splunk.</p>
<p> By default, if Go’s HTTP server encounters an error it will log it using the standard logger. For consistency it’d be better to use new errorLog logger instead.</p>
``` go
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
``` go
 type application struct {
  errorLog *log.Logger
  infoLog *log.Logger
 }
```
<p>And then update handlers function so that they become methods. Eg:</p>
``` go
 func (app *application) home(w http.ResponseWriter, r *http.Request){...}
```
<p>Initialize a new instance of our application struct, containing the dependencies in main()</p>
``` go
 app := &application{
  errorLog: errorLog,
  infoLog: infoLog,
 }
```
#### Alternative approach
<p>create a config package exporting an Application struct and have handler functions close over this to form a closure.</p>
``` go
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
### Database
### Middleware
### Routin
### Process Forms
### Session Manager
### TLS certificate
### Authentication
### Request Context
### File embedding and Generics
### Testing
