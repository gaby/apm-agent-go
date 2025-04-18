---
mapped_pages:
  - https://www.elastic.co/guide/en/apm/agent/go/current/custom-instrumentation-propagation.html
---

# Context propagation [custom-instrumentation-propagation]

In Go, [context](https://golang.org/pkg/context/) is used to propagate request-scoped values along a call chain, potentially crossing between goroutines and between processes. For servers based on `net/http`, each request contains an independent context object, which allows adding values specific to that particular request.

When you start a transaction, you can add it to a context object using [apm.ContextWithTransaction](/reference/api-documentation.md#apm-context-with-transaction). This context object can be later passed to [apm.TransactionFromContext](/reference/api-documentation.md#apm-transaction-from-context) to obtain the transaction, or into [apm.StartSpan](/reference/api-documentation.md#apm-start-span) to start a span.

The simplest way to create and propagate a span is by using [apm.StartSpan](/reference/api-documentation.md#apm-start-span), which takes a context and returns a span. The span will be created as a child of the span most recently added to this context, or a transaction added to the context as described above. If the context contains neither a transaction nor a span, then the span will be dropped (i.e. will not be reported to the APM Server.)

For example, take a simple CRUD-type web service, which accepts requests over HTTP and then makes corresponding database queries. For each incoming request, a transaction will be started and added to the request context automatically. This context needs to be passed into method calls within the handler manually in order to create spans within that transaction, e.g. to measure the duration of SQL queries.

```go
import (
	"net/http"

	"go.elastic.co/apm/v2"
	"go.elastic.co/apm/module/apmhttp/v2"
	"go.elastic.co/apm/module/apmsql/v2"
	_ "go.elastic.co/apm/module/apmsql/v2/pq"
)

var db *sql.DB

func init() {
	// apmsql.Open wraps sql.Open, in order
	// to add tracing to database operations.
	db, _ = apmsql.Open("postgres", "")
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", handleList)

	// apmhttp.Wrap instruments an http.Handler, in order
	// to report any request to this handler as a transaction,
	// and to store the transaction in the request's context.
	handler := apmhttp.Wrap(mux)
	http.ListenAndServe(":8080", handler)
}

func handleList(w http.ResponseWriter, req *http.Request) {
	// By passing the request context down to getList, getList can add spans to it.
	ctx := req.Context()
	getList(ctx)
	...
}

func getList(ctx context.Context) (
	// When getList is called with a context containing a transaction or span,
	// StartSpan creates a child span. In this example, getList is always called
	// with a context containing a transaction for the handler, so we should
	// expect to see something like:
	//
	//     Transaction: handleList
	//         Span: getList
	//             Span: SELECT FROM items
	//
	span, ctx := apm.StartSpan(ctx, "getList", "custom")
	defer span.End()

	// NOTE: The context object ctx returned by StartSpan above contains
	// the current span now, so subsequent calls to StartSpan create new
	// child spans.

	// db was opened with apmsql, so queries will be reported as
	// spans when using the context methods.
	rows, err := db.QueryContext(ctx, "SELECT * FROM items")
	...
	rows.Close()
}
```

Contexts can have deadlines associated and can be explicitly canceled. In some cases you may wish to propagate the trace context (parent transaction/span) to some code without propagating the cancellation. For example, an HTTP request’s context will be canceled when the client’s connection closes. You may want to perform some operation in the request handler without it being canceled due to the client connection closing, such as in a fire-and-forget operation. To handle scenarios like this, we provide the function [apm.DetachedContext](/reference/api-documentation.md#apm-detached-context).

```go
func handleRequest(w http.ResponseWriter, req *http.Request) {
	go fireAndForget(apm.DetachedContext(req.Context()))

	// After handleRequest returns, req.Context() will be canceled,
	// but the "detached context" passed into fireAndForget will not.
	// Any spans created by fireAndForget will still be joined to
	// the handleRequest transaction.
}
```

