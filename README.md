# NOTE: this project has been deprecated and superseded by https://github.com/opentracing/opentracing-go.

## Synopsis

Defines an interface for Uber libraries to use when reporting Dapper/Zipkin-style tracing information to distributed 
tracing systems.  By programming to this API the libraries are abstracted away from the internal trace representation 
used in the tracing system, as well as other concepts like sampling.

## Sample Usage

Assume you implement an http server that calls some other service while executing the request.

```go
// Initialize global Tracer variable
var tracer = tracing.NewNoopTracer()

// Initialize Endpoint descriptor of your service (use real IP address)
var endpoint = &tracing.Endpoint{ServiceName:"my-service", IPv4: 127<<24|1, Port: 1000}

// In the http handler function
func (h *myHandler) handler(ctx context.Context, w http.ResponseWriter, r *http.Request) {
    // instrumentation code
    spanName := urlToSpanName(r)
    client := makeEndpoint(r.RemoteAddr, r.Header.Get("Requestor"))
    header := r.Header.Get("X-Tracing")
    options := &tracing.BeginOptions{Peer: client}
    // call util method to create new trace or join the existing trace
    span, err := tracing.GetSpanFromHeader(header, tracer, spanName, endpoint, options)
    if err != nil {
        // header could not be parsed, but we may still decide to create a new trace
        span = tracer.BeginTrace(spanName, endpoint, options)
    }

    // You may annotate your span with events (timestamped) or attributes.  UI can find this trace 
    // via one of these case-insensitive queries: "api-version=1.2", "api-version", "i-got-hit".
    span.AddEvent("I-got-hit", nil)
    span.AddAttribute("api-version", "1.2")
    
    // propagation - store span in the context
    newCtx = tracing.ContextWithSpan(span)

    // continue with the regular handler
    processRequest(newCtx, w, r)

    // once finished, close the span.
    span.End(nil)
}
```

Suppose somewhere in `processRequest()` you need to make a call to another service

```go
func processRequest(ctx context.Context, ...) {
    // retrieve the span from the context
    span, err := tracing.GetSpanFromContext(ctx)
    
    // start a new RPC span
    options := &tracing.BeginOptions{Peer: ...}
    childSpan := span.BeginChildSpan("another-service", options)
    
    // make the call to the remote service, passing trace ID in the header
    clientReq := ...
    clientReq.Header.Put("X-Tracing", tracer.GetStringPickler().ToString(childSpan.SpanID()))
    httpClient.call(clientReq, ...)
    
    // upon completion, close the span
    childSpan.End(nil)
}
```

## Zipkin Trace ID

When RPC calls happen over a protocol that supports arbitrary string headers, the propagation of trace ID between
services can be done using `StringPickler` as shown in the previous section.  However, some protocols like 
[TChannel](https://github.com/uber/tchannel) encode tracing information in a specific Zipkin-compatible format, 
that consists of four fields, three 64-bit integers representing trace ID, span ID, and parent span ID, 
and an 8-bit flags field. The specific tracing system may not be compatible with propagating tracing information
via this format. If it is compatible, it can implement two additional interfaces, `ZipkinCompatibleTracer` and
`ZipkinSpanID`. Then to create a span ID the instrumentation code can do:

```go
var span Span
zipkinTracer, zipkinOK := tracer.(ZipkinCompatibleTracer)
if frame.tracing != nil && zipkinOK {
    spanID := zipkinTracer.CreateSpanID(frame.tracing.traceID, frame.tracing.ID,
                                        frame.tracing.parentID, frame.tracing.flags)
    span := tracer.JoinTrace(... spanID ...)
} else {
    span = tracer.BeginTrace(...)
}
```

And when making an outgoing RPC call, it can serialize span ID:

```go
if spanID, ok := childSpan.SpanID().(ZipkinSpanID); ok {
    outFrame.traceID = spanID.TraceID()
    outFrame.ID = spanID.ID()
    outFrame.parentID = spanID.ParentID()
}
...
```

## License

`opentracing-go` is available under the MIT license. See the LICENSE file for more info.

## Tests

`gocov test -v | gocov report`
