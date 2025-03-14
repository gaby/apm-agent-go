---
mapped_pages:
  - https://www.elastic.co/guide/en/apm/agent/go/current/builtin-modules.html
---

# Built-in instrumentation modules [builtin-modules]

For each server instrumentation module, a transaction is reported for each handled request. The transaction will be stored in the request context, which can be obtained through that framework’s API. The request context can be used for reporting [custom spans](/reference/custom-instrumentation.md#custom-instrumentation-spans).

* [module/apmhttp](#builtin-modules-apmhttp)
* [module/apmfasthttp](#builtin-modules-apmfasthttp)
* [module/apmecho](#builtin-modules-apmecho)
* [module/apmgin](#builtin-modules-apmgin)
* [module/apmfiber](#builtin-modules-apmfiber)
* [module/apmbeego](#builtin-modules-apmbeego)
* [module/apmgorilla](#builtin-modules-apmgorilla)
* [module/apmgrpc](#builtin-modules-apmgrpc)
* [module/apmhttprouter](#builtin-modules-apmhttprouter)
* [module/apmnegroni](#builtin-modules-apmnegroni)
* [module/apmlambda](#builtin-modules-apmlambda)
* [module/apmsql](#builtin-modules-apmsql)
* [module/apmgopg](#builtin-modules-apmgopg)
* [module/apmgorm](#builtin-modules-apmgorm)
* [module/apmgocql](#builtin-modules-apmgocql)
* [module/apmredigo](#builtin-modules-apmredigo)
* [module/apmgoredis](#builtin-modules-apmgoredis)
* [module/apmgoredisv8](#builtin-modules-apmgoredisv8)
* [module/apmrestful](#builtin-modules-apmrestful)
* [module/apmchi](#builtin-modules-apmchi)
* [module/apmlogrus](#builtin-modules-apmlogrus)
* [module/apmzap](#builtin-modules-apmzap)
* [module/apmzerolog](#builtin-modules-apmzerolog)
* [module/apmelasticsearch](#builtin-modules-apmelasticsearch)
* [module/apmmongo](#builtin-modules-apmmongo)
* [module/apmawssdkgo](#builtin-modules-apmawssdkgo)
* [module/apmazure](#builtin-modules-apmazure)
* [module/apmpgx](#builtin-modules-apmpgx)

## module/apmhttp [builtin-modules-apmhttp]

Package apmhttp provides a low-level `net/http` middleware handler. Other web middleware should typically be based off this.

For each request, a transaction is stored in the request context, which can be obtained via [http.Request.Context](https://golang.org/pkg/net/http/#Request.Context) in your handler.

```go
import (
	"go.elastic.co/apm/module/apmhttp/v2"
)

func main() {
	var myHandler http.Handler = ...
	tracedHandler := apmhttp.Wrap(myHandler)
}
```

The apmhttp handler will recover panics and send them to Elastic APM.

Package apmhttp also provides functions for instrumenting an `http.Client` or `http.RoundTripper` such that outgoing requests are traced as spans, if the request context includes a transaction. When performing the request, the enclosing context should be propagated by using [http.Request.WithContext](https://golang.org/pkg/net/http/#Request.WithContext), or a helper, such as those provided by [https://golang.org/x/net/context/ctxhttp](https://golang.org/x/net/context/ctxhttp).

Client spans are not ended until the response body is fully consumed or closed. If you fail to do either, the span will not be sent. Always close the response body to ensure HTTP connections can be reused; see [`func (*Client) Do`](https://golang.org/pkg/net/http/#Client.Do).

```go
import (
	"net/http"

	"golang.org/x/net/context/ctxhttp"

	"go.elastic.co/apm/v2"
	"go.elastic.co/apm/module/apmhttp/v2"
)

var tracingClient = apmhttp.WrapClient(http.DefaultClient)

func serverHandler(w http.ResponseWriter, req *http.Request) {
	// Propagate the transaction context contained in req.Context().
	resp, err := ctxhttp.Get(req.Context(), tracingClient, "http://backend.local/foo")
	if err != nil {
		apm.CaptureError(req.Context(), err).Send()
		http.Error(w, "failed to query backend", 500)
		return
	}
	body, err := ioutil.ReadAll(resp.Body)
	...
}

func main() {
	http.ListenAndServe(":8080", apmhttp.Wrap(http.HandlerFunc(serverHandler)))
}
```


## module/apmfasthttp [builtin-modules-apmfasthttp]

Package apmfasthttp provides a low-level [valyala/fasthttp](https://github.com/valyala/fasthttp) middleware request handler. Other fasthttp-based web middleware should typically be based off this.

For each request, a transaction is stored in the request context, which can be obtained via [fasthttp.RequestCtx](https://pkg.go.dev/github.com/valyala/fasthttp#RequestCtx) in your handler using `apm.TransactionFromContext`.

```go
import (
	"github.com/valyala/fasthttp"

	"go.elastic.co/apm/module/apmfasthttp/v2"
)

func main() {
	var myHandler fasthttp.RequestHandler = func(ctx *fasthttp.RequestCtx) {
		apmCtx := apm.TransactionFromContext(ctx)
		// ...
	}
	tracedHandler := apmfasthttp.Wrap(myHandler)
}
```

The apmfasthttp handler will recover panics and send them to Elastic APM.


## module/apmecho [builtin-modules-apmecho]

Packages apmecho and apmechov4 provide middleware for the [Echo](https://github.com/labstack/echo) web framework, versions 3.x and 4.x respectively.

If you are using Echo 4.x (`github.com/labstack/echo/v4`), use `module/apmechov4`. For the older Echo 3.x versions (`github.com/labstack/echo`), use `module/apmecho`.

For each request, a transaction is stored in the request context, which can be obtained via [echo.Context](https://godoc.org/github.com/labstack/echo#Context)`.Request().Context()` in your handler.

```go
import (
	echo "github.com/labstack/echo/v4"

	"go.elastic.co/apm/module/apmechov4/v2"
)

func main() {
	e := echo.New()
	e.Use(apmechov4.Middleware())
	...
}
```

The middleware will recover panics and send them to Elastic APM, so you do not need to install the echo/middleware.Recover middleware.


## module/apmgin [builtin-modules-apmgin]

Package apmgin provides middleware for the [Gin](https://gin-gonic.com/) web framework.

For each request, a transaction is stored in the request context, which can be obtained via [gin.Context](https://godoc.org/github.com/gin-gonic/gin#Context)`.Request.Context()` in your handler.

```go
import (
	"go.elastic.co/apm/module/apmgin/v2"
)

func main() {
	engine := gin.New()
	engine.Use(apmgin.Middleware(engine))
	...
}
```

The apmgin middleware will recover panics and send them to Elastic APM, so you do not need to install the gin.Recovery middleware.


## module/apmfiber [builtin-modules-apmfiber]

Package apmfiber provides middleware for the [Fiber](https://gofiber.io/) web framework, versions 2.x (v2.18.0 and greater).

For each request, a transaction is stored in the request context, which can be obtained via [fiber.Ctx](https://pkg.go.dev/github.com/gofiber/fiber/v2#Ctx)`.Context()` in your handler.

```go
import (
	"go.elastic.co/apm/module/apmfiber/v2"
)

func main() {
	app := fiber.New()
	app.Use(apmfiber.Middleware())
	...
}
```

The apmfiber middleware will recover panics and send them to Elastic APM, so you do not need to install the fiber [recover](https://docs.gofiber.io/api/middleware/recover) middleware. You can disable default behaviour by using `WithPanicPropagation` option.


## module/apmbeego [builtin-modules-apmbeego]

Package apmbeego provides middleware for the [Beego](https://beego.me/) web framework.

For each request, a transaction is stored in the request context, which can be obtained via [beego/context.Context](https://godoc.org/github.com/astaxie/beego/context#Context)`.Request.Context()` in your controller.

```go
import (
	"github.com/astaxie/beego"

	"go.elastic.co/apm/v2"
	"go.elastic.co/apm/module/apmbeego/v2"
)

type thingController struct{beego.Controller}

func (c *thingController) Get() {
	span, _ := apm.StartSpan(c.Ctx.Request.Context(), "thingController.Get", "controller")
	span.End()
	...
}

func main() {
	beego.Router("/", &thingController{})
	beego.Router("/thing/:id:int", &thingController{}, "get:Get")
	beego.RunWithMiddleWares("localhost:8080", apmbeego.Middleware())
}
```


## module/apmgorilla [builtin-modules-apmgorilla]

Package apmgorilla provides middleware for the [Gorilla Mux](http://www.gorillatoolkit.org/pkg/mux) router.

For each request, a transaction is stored in the request context, which can be obtained via [http.Request](https://golang.org/pkg/net/http/#Request)`.Context()` in your handler.

```go
import (
	"github.com/gorilla/mux"

	"go.elastic.co/apm/module/apmgorilla/v2"
)

func main() {
	router := mux.NewRouter()
	apmgorilla.Instrument(router)
	...
}
```

The apmgorilla middleware will recover panics and send them to Elastic APM, so you do not need to install any other recovery middleware.


## module/apmgrpc [builtin-modules-apmgrpc]

Package apmgrpc provides server and client interceptors for [gRPC-Go](https://github.com/grpc/grpc-go). Server interceptors report transactions for each incoming request, while client interceptors report spans for each outgoing request. For each RPC served, a transaction is stored in the context passed into the method.

```go
import (
	"go.elastic.co/apm/module/apmgrpc/v2"
)

func main() {
	server := grpc.NewServer(
		grpc.UnaryInterceptor(apmgrpc.NewUnaryServerInterceptor()),
		grpc.StreamInterceptor(apmgrpc.NewStreamServerInterceptor()),
	)
	...
	conn, err := grpc.Dial(addr,
		grpc.WithUnaryInterceptor(apmgrpc.NewUnaryClientInterceptor()),
		grpc.WithStreamInterceptor(apmgrpc.NewStreamClientInterceptor()),
	)
	...
}
```

The server interceptor can optionally be made to recover panics, in the same way as [grpc_recovery](https://github.com/grpc-ecosystem/go-grpc-middleware/tree/master/recovery). The apmgrpc server interceptor will always send panics it observes as errors to the Elastic APM server. If you want to recover panics but also want to continue using grpc_recovery, then you should ensure that it comes before the apmgrpc interceptor in the interceptor chain, or panics will not be captured by apmgrpc.

```go
server := grpc.NewServer(grpc.UnaryInterceptor(
	apmgrpc.NewUnaryServerInterceptor(apmgrpc.WithRecovery()),
))
...
```

Stream interceptors emit transactions and spans that represent the entire stream, and not individual messages. For client streams, spans will be ended when the request fails; when any of `grpc.ClientStream.RecvMsg`, `grpc.ClientStream.SendMsg`, or `grpc.ClientStream.Header` return with an error; or when `grpc.ClientStream.RecvMsg` returns for a non-streaming server method.


## module/apmhttprouter [builtin-modules-apmhttprouter]

Package apmhttprouter provides a low-level middleware handler for [httprouter](https://github.com/julienschmidt/httprouter).

For each request, a transaction is stored in the request context, which can be obtained via [http.Request](https://golang.org/pkg/net/http/#Request)`.Context()` in your handler.

```go
import (
	"github.com/julienschmidt/httprouter"

	"go.elastic.co/apm/module/apmhttprouter/v2"
)

func main() {
	router := httprouter.New()

	const route = "/my/route"
	router.GET(route, apmhttprouter.Wrap(h, route))
	...
}
```

[httprouter does not provide a means of obtaining the matched route](https://github.com/julienschmidt/httprouter/pull/139), hence the route must be passed into the wrapper.

Alternatively, use the `apmhttprouter.Router` type, which wraps `httprouter.Router`, providing the same API and instrumenting added routes. To use this wrapper type, rewrite your use of `httprouter.New` to `apmhttprouter.New`; the returned type is `*apmhttprouter.Router`, and not `*httprouter.Router`.

```go
import "go.elastic.co/apm/module/apmhttprouter/v2"

func main() {
	router := apmhttprouter.New()

	router.GET(route, h)
	...
}
```


## module/apmnegroni [builtin-modules-apmnegroni]

Package apmnegroni provides middleware for the [negroni](https://github.com/urfave/negroni/) library.

For each request, a transaction is stored in the request context, which can be obtained via [http.Request.Context](https://golang.org/pkg/net/http/#Request.Context) in your handler.

```go
import (
	"net/http"

	"go.elastic.co/apm/module/apmnegroni/v2"
)

func serverHandler(w http.ResponseWriter, req *http.Request) {
	...
}

func main() {
	n := negroni.New()
	n.Use(apmnegroni.Middleware())
	n.UseHandler(serverHandler)
	http.ListenAndServe(":8080", n)
}
```

The apmnegroni handler will recover panics and send them to Elastic APM.


## module/apmlambda [builtin-modules-apmlambda]

Package apmlambda intercepts requests to your AWS Lambda function invocations.

::::{warning}
This functionality is in technical preview and may be changed or removed in a future release. Elastic will work to fix any issues, but features in technical preview are not subject to the support SLA of official GA features.
::::


Importing the package is enough to report the function invocations.

```go
import (
	_ "go.elastic.co/apm/module/apmlambda/v2"
)
```

We currently do not expose the transactions via context; when we do, it will be necessary to make a small change to your code to call apmlambda.Start instead of lambda.Start.


## module/apmsql [builtin-modules-apmsql]

Package apmsql provides a means of wrapping `database/sql` drivers so that queries and other executions are reported as spans within the current transaction.

To trace SQL queries, register drivers using apmsql.Register and obtain connections with apmsql.Open. The parameters are exactly the same as if you were to call sql.Register and sql.Open respectively.

As a convenience, we also provide packages which will automatically register popular drivers with apmsql.Register:

* module/apmsql/pq (github.com/lib/pq)
* module/apmsql/pgxv4 (github.com/jackc/pgx/v4/stdlib)
* module/apmsql/mysql (github.com/go-sql-driver/mysql)
* module/apmsql/sqlite3 (github.com/mattn/go-sqlite3)
* module/apmsql/sqlserver (github.com/microsoft/mssqldb)

```go
import (
	"go.elastic.co/apm/module/apmsql/v2"
	_ "go.elastic.co/apm/module/apmsql/v2/pq"
	_ "go.elastic.co/apm/module/apmsql/v2/sqlite3"
)

func main() {
	db, err := apmsql.Open("postgres", "postgres://...")
	db, err := apmsql.Open("sqlite3", ":memory:")
}
```

Spans will be created for queries and other statement executions if the context methods are used, and the context includes a transaction.


## module/apmgopg [builtin-modules-apmgopg]

Package apmgopg provides a means of instrumenting [go-pg](http://github.com/go-pg/pg) database operations.

To trace `go-pg` statements, call `apmgopg.Instrument` with the database instance you plan on using and provide a context that contains an apm transaction.

```go
import (
	"github.com/go-pg/pg"

	"go.elastic.co/apm/module/apmgopg/v2"
)

func main() {
	db := pg.Connect(&pg.Options{})
	apmgopg.Instrument(db)

	db.WithContext(ctx).Model(...)
}
```

Spans will be created for queries and other statement executions if the context methods are used, and the context includes a transaction.


## module/apmgopgv10 [builtin-modules-apmgopgv10]

Package apmgopgv10 provides a means of instrumenting v10 of [go-pg](http://github.com/go-pg/pg) database operations.

To trace `go-pg` statements, call `apmgopgv10.Instrument` with the database instance you plan on using and provide a context that contains an apm transaction.

```go
import (
	"github.com/go-pg/pg/v10"

	"go.elastic.co/apm/module/apmgopgv10/v2"
)

func main() {
	db := pg.Connect(&pg.Options{})
	apmgopg.Instrument(db)

	db.WithContext(ctx).Model(...)
}
```


## module/apmgorm [builtin-modules-apmgorm]

Package apmgorm provides a means of instrumenting [GORM](http://gorm.io) database operations.

To trace `GORM` operations, import the appropriate `apmgorm/dialects` package (instead of the `gorm/dialects` package), and use `apmgorm.Open` (instead of `gorm.Open`). The parameters are exactly the same.

Once you have a `*gorm.DB` from `apmgorm.Open`, you can call `apmgorm.WithContext` to propagate a context containing a transaction to the operations:

```go
import (
	"go.elastic.co/apm/module/apmgorm/v2"
	_ "go.elastic.co/apm/module/apmgorm/v2/dialects/postgres"
)

func main() {
	db, err := apmgorm.Open("postgres", "")
	...
	db = apmgorm.WithContext(ctx, db)
	db.Find(...) // creates a "SELECT FROM <foo>" span
}
```


## module/apmgormv2 [_moduleapmgormv2]

Package apmgormv2 provides a means of instrumenting [GORM](http://gorm.io) database operations.

To trace `GORM` operations, import the appropriate `apmgormv2/driver` package (instead of the `gorm.io/driver` package), use these dialects to `gorm.Open` instead of gorm drivers.

Once you have a `*gorm.DB`, you can call `db.WithContext` to propagate a context containing a transaction to the operations:

```go
import (
	"gorm.io/gorm"
	postgres "go.elastic.co/apm/module/apmgormv2/v2/driver/postgres"
)

func main() {
	db, err := gorm.Open(postgres.Open("dsn"), &gorm.Config{})
	...
	db = db.WithContext(ctx)
	db.Find(...) // creates a "SELECT FROM <foo>" span
}
```


## module/apmgocql [builtin-modules-apmgocql]

Package apmgocql provides a means of instrumenting [gocql](https://github.com/gocql/gocql) so that queries are reported as spans within the current transaction.

To report `gocql` queries, construct an `apmgocql.Observer` and assign it to the `QueryObserver` and `BatchObserver` fields of `gocql.ClusterConfig`, or install it into a specific `gocql.Query` or `gocql.Batch` via their `Observer` methods.

Spans will be created for queries as long as they have context associated, and the context includes a transaction.

```go
import (
	"github.com/gocql/gocql"

	"go.elastic.co/apm/module/apmgocql/v2"
)

func main() {
	observer := apmgocql.NewObserver()
	config := gocql.NewCluster("cassandra_host")
	config.QueryObserver = observer
	config.BatchObserver = observer

	session, err := config.CreateSession()
	...
	err = session.Query("SELECT * FROM foo").WithContext(ctx).Exec()
	...
}
```


## module/apmredigo [builtin-modules-apmredigo]

Package apmredigo provides a means of instrumenting [Redigo](https://github.com/gomodule/redigo) so that Redis commands are reported as spans within the current transaction.

To report Redis commands, use the top-level `Do` or `DoWithTimeout` functions. These functions have the same signature as the `redis.Conn` equivalents apart from an initial `context.Context` parameter. If the context passed in contains a sampled transaction, a span will be reported for the Redis command.

Another top-level function, `Wrap`, is provided to wrap a `redis.Conn` such that its `Do` and `DoWithTimeout` methods call the above mentioned functions. Initially, the wrapped connection will be associated with the background context; its `WithContext` method may be used to obtain a shallow copy with another context. For example, in an HTTP middleware you might bind a connection to the request context, which would associate spans with the request’s APM transaction.

```go
import (
	"net/http"

	"github.com/gomodule/redigo/redis"

	"go.elastic.co/apm/module/apmredigo/v2"
)

var redisPool *redis.Pool // initialized at program startup

func handleRequest(w http.ResponseWriter, req *http.Request) {
	// Wrap and bind redis.Conn to request context. If the HTTP
	// server is instrumented with Elastic APM (e.g. with apmhttp),
	// Redis commands will be reported as spans within the request's
	// transaction.
	conn := apmredigo.Wrap(redisPool.Get()).WithContext(req.Context())
	defer conn.Close()
	...
}
```


## module/apmgoredis [builtin-modules-apmgoredis]

Package apmgoredis provides a means of instrumenting [go-redis/redis](https://github.com/go-redis/redis) so that Redis commands are reported as spans within the current transaction.

To report Redis commands, use the top-level `Wrap` function to wrap a `redis.Client`, `redis.ClusterClient`, or `redis.Ring`. Initially, the wrapped client will be associated with the background context; its `WithContext` method may be used to obtain a shallow copy with another context. For example, in an HTTP middleware you might bind a client to the request context, which would associate spans with the request’s APM transaction.

```go
import (
	"net/http"

	"github.com/go-redis/redis"

	"go.elastic.co/apm/module/apmgoredis/v2"
)

var redisClient *redis.Client // initialized at program startup

func handleRequest(w http.ResponseWriter, req *http.Request) {
	// Wrap and bind redisClient to the request context. If the HTTP
	// server is instrumented with Elastic APM (e.g. with apmhttp),
	// Redis commands will be reported as spans within the request's
	// transaction.
	client := apmgoredis.Wrap(redisClient).WithContext(req.Context())
	...
}
```


## module/apmgoredisv8 [builtin-modules-apmgoredisv8]

Package apmgoredisv8 provides a means of instrumenting [go-redis/redis](https://github.com/go-redis/redis) for v8 so that Redis commands are reported as spans within the current transaction.

To report Redis commands, you can call `AddHook(apmgoredis.NewHook())` from instance of `redis.Client`, `redis.ClusterClient`, or `redis.Ring`.

```go
import (
	"github.com/go-redis/redis/v8"

	apmgoredis "go.elastic.co/apm/module/apmgoredisv8/v2"
)

func main() {
	redisClient := redis.NewClient(&redis.Options{})
	// Add apm hook to redisClient.
	// Redis commands will be reported as spans within the current transaction.
	redisClient.AddHook(apmgoredis.NewHook())

	redisClient.Get(ctx, "key")
}
```


## module/apmrestful [builtin-modules-apmrestful]

Package apmrestful provides a [go-restful](https://github.com/emicklei/go-restful) filter for tracing requests, and capturing panics.

For each request, a transaction is stored in the request context, which can be obtained via [http.Request](https://golang.org/pkg/net/http/#Request)`.Context()` in your handler.

```go
import (
	"github.com/emicklei/go-restful"

	"go.elastic.co/apm/module/apmrestful/v2"
)

func init() {
	// Trace all requests to web services registered with the default container.
	restful.Filter(apmrestful.Filter())
}

func main() {
	var ws restful.WebService
	ws.Path("/things").Consumes(restful.MIME_JSON, restful.MIME_XML).Produces(restful.MIME_JSON, restful.MIME_XML)
	ws.Route(ws.GET("/{id:[0-1]+}").To(func(req *restful.Request, resp *restful.Response) {
		// req.Request.Context() should be propagated to downstream operations such as database queries.
	}))
	...
}
```


## module/apmchi [builtin-modules-apmchi]

Package apmchi provides middleware for [chi](https://github.com/go-chi/chi) routers, for tracing requests and capturing panics.

For each request, a transaction is stored in the request context, which can be obtained via [http.Request](https://golang.org/pkg/net/http/#Request)`.Context()` in your handler.

```go
import (
	"github.com/go-chi/chi"

	"go.elastic.co/apm/module/apmchi/v2"
)

func main() {
	r := chi.NewRouter()
	r.Use(apmchi.Middleware())
	r.Get("/route/{pattern}", routeHandler)
	...
}
```


## module/apmlogrus [builtin-modules-apmlogrus]

Package apmlogrus provides a [logrus](https://github.com/sirupsen/logrus) Hook implementation for sending error messages to Elastic APM, as well as a function for adding trace context fields to log records.

```go
import (
	"github.com/sirupsen/logrus"

	"go.elastic.co/apm/module/apmlogrus/v2"
)

func init() {
	// apmlogrus.Hook will send "error", "panic", and "fatal" level log messages to Elastic APM.
	logrus.AddHook(&apmlogrus.Hook{})
}

func handleRequest(w http.ResponseWriter, req *http.Request) {
	// apmlogrus.TraceContext extracts the transaction and span (if any) from the given context,
	// and returns logrus.Fields containing the trace, transaction, and span IDs.
	traceContextFields := apmlogrus.TraceContext(req.Context())
	logrus.WithFields(traceContextFields).Debug("handling request")

	// Output:
	// {"level":"debug","msg":"handling request","time":"1970-01-01T00:00:00Z","trace.id":"67829ae467e896fb2b87ec2de50f6c0e","transaction.id":"67829ae467e896fb"}
}
```


## module/apmzap [builtin-modules-apmzap]

Package apmzap provides a [go.uber.org/zap/zapcore.Core](https://godoc.org/go.uber.org/zap/zapcore#Core) implementation for sending error messages to Elastic APM, as well as a function for adding trace context fields to log records.

```go
import (
	"go.uber.org/zap"

	"go.elastic.co/apm/module/apmzap/v2"
)

// apmzap.Core.WrapCore will wrap the core created by zap.NewExample
// such that logs are also sent to the apmzap.Core.
//
// apmzap.Core will send "error", "panic", and "fatal" level log
// messages to Elastic APM.
var logger = zap.NewExample(zap.WrapCore((&apmzap.Core{}).WrapCore))

func handleRequest(w http.ResponseWriter, req *http.Request) {
	// apmzap.TraceContext extracts the transaction and span (if any)
	// from the given context, and returns zap.Fields containing the
	// trace, transaction, and span IDs.
	traceContextFields := apmzap.TraceContext(req.Context())
	logger.With(traceContextFields...).Debug("handling request")

	// Output:
	// {"level":"debug","msg":"handling request","trace.id":"67829ae467e896fb2b87ec2de50f6c0e","transaction.id":"67829ae467e896fb"}
}
```


## module/apmslog [builtin-modules-apmslog]

Package apmslog provides a [slog](https://pkg.go.dev/log/slog) Handler implementation for sending error messages to Elastic APM, as well as automatically attaching trace context fields to log records while using the context aware logging methods.

```go
import (
	"context"
	"log/slog"

	"go.elastic.co/apm/module/apmslog/v2"
)

func ExampleHandler() {
	// Report slog "ERROR" level messages to Elastic APM using
	// apm.DefaultTracer() while utilizing some specific slog handler
	// to format logging messages
	apmHandler = apmslog.NewApmHandler(
		apmslog.WithHandler(
			slog.NewJSONHandler(os.Stderr, &slog.HandlerOptions{
				Level: slog.LevelInfo,
			}),
		),
	)
	logger = slog.New(apmHandler)

	// while using slog context aware methods, any existing trace,
	// transaction, or span ID are added from the given context
	tx := apm.DefaultTracer().StartTransaction("name", "type")
	defer tx.End()

	ctx := apm.ContextWithTransaction(context.Background(), tx)
	span, ctx := apm.StartSpan(ctx, "name", "type")
	defer span.End()

	// log msg will have a trace, transaction, and a span attached
	logger.InfoContext(ctx, "I should have a trace, transaction, and span id attached!")

	// the log msg will be reported to apm
	logger.ErrorContext(ctx, "I want this to be reported, but have no error to attach")

	// the log msg with its error will be reported to apm
	logger.ErrorContext(ctx, "I will report this error to apm", "error", errors.New("new error"))

	// BOTH errors with the log msg will be reported to apm. [ error, err ] slog attribute keys are by default reported
	logger.ErrorContext(ctx, "I will report this error to apm", "error", errors.New("new error"), "err", errors.New("new err"))
}
```


## module/apmzerolog [builtin-modules-apmzerolog]

Package apmzerolog provides an implementation of [Zerolog](https://github.com/rs/zerolog)'s `LevelWriter` interface for sending error records to Elastic APM, as well as functions for adding trace context and detailed error stack traces to log records.

```go
import (
	"net/http"

	"github.com/rs/zerolog"

	"go.elastic.co/apm/module/apmzerolog/v2"
)

// apmzerolog.Writer will send log records with the level error or greater to Elastic APM.
var logger = zerolog.New(zerolog.MultiLevelWriter(os.Stdout, &apmzerolog.Writer{}))

func init() {
	// apmzerolog.MarshalErrorStack will extract stack traces from
	// errors produced by github.com/pkg/errors. The main difference
	// with github.com/rs/zerolog/pkgerrors.MarshalStack is that
	// the apmzerolog implementation records fully-qualified function
	// names, enabling errors reported to Elastic APM to be attributed
	// to the correct package.
	zerolog.ErrorStackMarshaler = apmzerolog.MarshalErrorStack
}

func traceLoggingMiddleware(h http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
		ctx := req.Context()
		logger := zerolog.Ctx(ctx).Hook(apmzerolog.TraceContextHook(ctx))
		req = req.WithContext(logger.WithContext(ctx))
		h.ServeHTTP(w, req)
	})
}
```


## module/apmelasticsearch [builtin-modules-apmelasticsearch]

Package apmelasticsearch provides a means of instrumenting the HTTP transport of Elasticsearch clients, such as [go-elasticsearch](https://github.com/elastic/go-elasticsearch) and [olivere/elastic](https://github.com/olivere/elastic), so that Elasticsearch requests are reported as spans within the current transaction.

To create spans for an Elasticsearch request, wrap the client’s HTTP transport using the `WrapRoundTripper` function, and then associate the request with a context containing a transaction.

```go
import (
	"net/http"

	"github.com/olivere/elastic"

	"go.elastic.co/apm/module/apmelasticsearch/v2"
)

var client, _ = elastic.NewClient(elastic.SetHttpClient(&http.Client{
	Transport: apmelasticsearch.WrapRoundTripper(http.DefaultTransport),
}))

func handleRequest(w http.ResponseWriter, req *http.Request) {
	result, err := client.Search("index").Query(elastic.NewMatchAllQuery()).Do(req.Context())
	...
}
```


## module/apmmongo [builtin-modules-apmmongo]

Package apmmongo provides a means of instrumenting the [MongoDB Go Driver](https://github.com/mongodb/mongo-go-driver), so that MongoDB commands are reported as spans within the current transaction.

To create spans for MongoDB commands, pass in a `CommandMonitor` created with `apmmongo.CommandMonitor` as an option when constructing a client, and then when executing commands, pass in a context containing a transaction.

```go
import (
	"context"
	"net/http"

	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"

	"go.elastic.co/apm/module/apmmongo/v2"
)

var client, _ = mongo.Connect(
	context.Background(),
	options.Client().SetMonitor(apmmongo.CommandMonitor()),
)

func handleRequest(w http.ResponseWriter, req *http.Request) {
	collection := client.Database("db").Collection("coll")
	cur, err := collection.Find(req.Context(), bson.D{})
	...
}
```


## module/apmawssdkgo [builtin-modules-apmawssdkgo]

Package apmawssdkgo provides a means of instrumenting the [AWS SDK Go](https://github.com/aws/aws-sdk-go) session object, so that AWS requests are reported as spans within the current transaction.

To create spans for AWS requests, you should wrap the `session.Session` created with `session.NewSession` when constructing a client. When executing commands, pass in a context containing a transaction.

The following services are supported:

* S3
* DynamoDB
* SQS
* SNS

Passing a `session.Session` wrapped with `apmawssdkgo.WrapSession` to these services from the AWS SDK will report spans within the current transaction.

```go
import (
	"context"
	"net/http"

	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/service/s3/s3manager"

	"go.elastic.co/apm/module/apmawssdkgo/v2"
)

func main() {
  session := session.Must(session.NewSession())
  session = apmawssdkgo.WrapSession(session)

  uploader := s3manager.NewUploader(session)
  s := &server{uploader}
  ...
}

func (s *server) handleRequest(w http.ResponseWriter, req *http.Request) {
  ctx := req.Context()
  out, err := s.uploader.UploadWithContext(ctx, &s3manager.UploadInput{
    Bucket: aws.String("your-bucket"),
    Key:    aws.String("your-key"),
    Body:   bytes.NewBuffer([]byte("your-body")),
  })
  ...
}
```


## module/apmazure [builtin-modules-apmazure]

Package apmazure provides a means of instrumenting the [Azure Pipeline Go](https://github.com/Azure/azure-pipeline-go) pipeline object, so that Azure requests are reported as spans within the current transaction.

To create spans for Azure requests, you should create a new pipeline using the relevant service’s `NewPipeline` function, like `azblob.NewPipeline`, then wrap the `pipeline.Pipeline` with `apmazure.WrapPipeline`. The returned `Pipeline` can be used as normal.

The following services are supported:

* Blob Storage
* Queue Storage
* File Storage

```go
import (
  "github.com/Azure/azure-storage-blob-go/azblob"

  "go.elastic.co/apm/module/apmazure/v2"
)

func main() {
  p := azblob.NewPipeline(azblob.NewAnonymousCredential(), po)
  p = apmazure.WrapPipeline(p)
  u, err := url.Parse("https://my-account.blob.core.windows.net")
  serviceURL := azblob.NewServiceURL(*u, p)
  containerURL := serviceURL.NewContainerURL("mycontainer")
  blobURL := containerURL.NewBlobURL("readme.txt")
  // Use the blobURL to interact with Blob Storage
  ...
}
```


## module/apmpgx [builtin-modules-apmpgx]

Package apmpgx provides a means of instrumenting the [Pgx](https://github.com/jackc/pgx) for v4.17+, so that SQL queries are reported as spans within the current transaction. Also this lib have extended support of pgx, such as COPY FROM queries and BATCH which have their own span types: `db.postgresql.copy` and `db.postgresql.batch` accordingly.

To report `pgx` queries, create `pgx` connection, and then provide config to `apmpgx.Instrument()`. If logger is presented in config, then traces will be written to log. (It’s safe to use without logger)

Spans will be created for queries as long as they have context associated, and the context includes a transaction.

Example with pool:

```go
import (
    "github.com/jackc/pgx/v4/pgxpool"

    "go.elastic.co/apm/module/apmpgx/v2"
)

func main() {
    c, err := pgxpool.ParseConfig("dsn_string")
    ...
    pool, err := pgxpool.ParseConfig("dsn")
    ...
    // set custom logger before instrumenting
    apmpgx.Instrument(pool.ConnConfig)
    ...
}
```

Example without pool:

```go
import (
    "github.com/jackc/pgx/v4"

    "go.elastic.com/apm/module/apmpgx/v2"
)

func main() {
    c, err := pgx.Connect(context.Background, "dsn_string")
    ...
    // set custom logger before instrumenting
    apmpgx.Instrument(c.Config())
    ...
}
```


