<!--- Hugo front matter used to generate the website version of this page:
title: OTLP Specification
linkTitle: OTLP
aliases:
  - /docs/reference/specification/protocol/otlp
  - /docs/specs/otel/protocol/otlp
spelling:
  cSpell:ignore backoff backpressure dealocations errdetails nanos proto reconnections retryable
cascade:
  body_class: otel-docs-spec
  github_repo: &repo https://github.com/open-telemetry/opentelemetry-proto
  github_subdir: docs
  path_base_for_github_subdir: content/en/docs/specs/otlp/
  github_project_repo: *repo
path_base_for_github_subdir:
  from: content/en/docs/specs/otlp/_index.md
  to: specification.md
--->

# OpenTelemetry Protocol Specification

**Status**: [Stable](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/document-status.md)

开放遥测协议 (OTLP) 规范描述了遥测数据在遥测源、中间节点（例如收集器和遥测后端）之间的编码、传输和交付机制。

<details>
<summary>目录</summary>

<!-- toc -->

- [协议详情](#protocol-details)
  * [OTLP/gRPC](#otlpgrpc)
    + [OTLP/gRPC 并发请求](#otlpgrpc-concurrent-requests)
    + [OTLP/gRPC 响应](#otlpgrpc-response)
      - [Full Success](#full-success)
      - [Partial Success](#partial-success)
      - [Failures](#failures)
    + [OTLP/gRPC 节流](#otlpgrpc-throttling)
    + [OTLP/gRPC 服务和 Protobuf 定义](#otlpgrpc-service-and-protobuf-definitions)
    + [OTLP/gRPC 默认端口](#otlpgrpc-default-port)
  * [OTLP/HTTP](#otlphttp)
    + [二进制 Protobuf 编码](#binary-protobuf-encoding)
    + [JSON Protobuf 编码](#json-protobuf-encoding)
    + [OTLP/HTTP 请求](#otlphttp-request)
    + [OTLP/HTTP 响应](#otlphttp-response)
      - [Full Success](#full-success-1)
      - [Partial Success](#partial-success-1)
      - [Failures](#failures-1)
      - [Bad Data](#bad-data)
      - [OTLP/HTTP Throttling](#otlphttp-throttling)
      - [All Other Responses](#all-other-responses)
    + [OTLP/HTTP 连接](#otlphttp-connection)
    + [OTLP/HTTP 并发请求](#otlphttp-concurrent-requests)
    + [OTLP/HTTP 默认端口](#otlphttp-default-port)
- [实现建议](#implementation-recommendations)
  * [Multi-Destination Exporting](#multi-destination-exporting)
- [已知限制](#known-limitations)
  * [Request Acknowledgements](#request-acknowledgements)
    + [Duplicate Data](#duplicate-data)
- [Future Versions and Interoperability](#future-versions-and-interoperability)
- [Glossary](#glossary)
- [References](#references)

<!-- tocstop -->

</details>

OTLP 是在 OpenTelemetry 项目范围内设计的通用遥测数据传输协议。  

## 协议详情

OTLP 定义了遥测数据的编码以及用于在客户端和服务器之间交换数据的协议。  

该规范定义了如何通过 [gRPC](https://grpc.io/) 和 HTTP 1.1 传输 实现 OTLP，并指定用于 Payload 编码的 [Protocol Buffers schema](https://developers.google.com/protocol-buffers/docs/overview)。  

OTLP 是一种请求/响应式协议：客户端发送请求，服务器回复相应的响应。本文档定义了一种请求和响应类型：Export。  

所有服务器组件必须支持以下传输压缩选项：  

- 无压缩，用 表示none。
- Gzip 压缩，用 表示gzip。

### OTLP/gRPC

建立底层 gRPC 传输后，客户端开始使用 [Export*ServiceRequest](https://github.com/open-telemetry/opentelemetry-proto)
消息 ([ExportLogsServiceRequest](../opentelemetry/proto/collector/logs/v1/logs_service.proto) 用于logs,
[ExportMetricsServiceRequest](../opentelemetry/proto/collector/metrics/v1/metrics_service.proto) 用于 metrics,
[ExportTraceServiceRequest](../opentelemetry/proto/collector/trace/v1/trace_service.proto) 用于 traces) 使用一元请求发送遥测数据。  

客户端不断向服务器发送一系列请求，并期望收到每个请求的响应：

![Request-Response](img/otlp-request-response.png)

注意：该协议关注一对客户端/服务器节点之间传输的可靠性，旨在确保客户端和服务器之间的传输过程中不会丢失数据。许多遥测收集系统都有中间节点，数据必须经过这些中间节点才能到达最终目的地（例如应用程序 -> 代理 -> 收集器 -> 后端）。此类系统中的端到端交付保证超出了 OTLP 的范围。本协议中描述的确认发生在单个客户端/服务器对之间，并且不跨越多跳传送路径中的中间节点。

#### OTLP/gRPC 并发请求

发送请求后，客户端可以等待，直到收到服务器的响应。在这种情况下，最多只有一个正在运行的请求尚未被服务器确认。  

![Unary](img/otlp-sequential.png)

当希望实现简单时，并且当客户端和服务器通过非常低延迟的网络连接时，例如当客户端是一个仪表应用程序而服务器是作为本地守护进程（代理）运行的 OpenTelemetry Collector 时，建议使用顺序操作。  

需要实现高吞吐量的实现应该支持并发一元调用以实现更高的吞吐量。客户端应该发送新请求，而不等待对先前发送的请求的响应，本质上是创建一个当前正在运行但未确认的请求管道。  

![Concurrent](img/otlp-concurrent.png)

并发请求的数量应该是可配置的。  

可实现的最大吞吐量为 `max_concurrent_requests * max_request_size / (network_latency + server_response_time)`。  

例如，如果请求最多可包含 100 个 Span，网络往返延迟为 200 毫秒，服务器响应时间为 300 毫秒，则一个并发请求可实现的最大吞吐量为100 spans / (200ms+300ms)每秒 200 个 Span。很容易看出，在高延迟网络中或者当服务器响应时间很长才能获得良好的吞吐量时，请求量需要非常大或者必须执行大量并发请求。  

如果客户端正在关闭（例如，当包含进程想要退出时），客户端将选择性地等待，直到收到所有挂起的确认或直到特定于实现的超时到期。这确保了遥测数据的可靠传输。客户端实现应该公开一个选项来在关闭期间打开和关闭等待。  

如果客户端无法传送某个请求（例如，在等待确认时计时器到期），客户端应该记录数据未传送的事实。  

#### OTLP/gRPC 响应

响应必须是适当的消息 ( 请参阅下文以了解在 [完全成功](#full-success),
[部分成功](#partial-success) 和 [失败](#failures) 情况下使用的特定消息).  



##### 完全成功

成功响应表示遥测数据已被服务器成功接受。  

如果服务器收到一个空请求（不携带任何遥测数据的请求），服务器应该成功响应。  

成功后，服务器响应必须是[Export\<signal>ServiceResponse](../opentelemetry/proto/collector) 消息（ExportTraceServiceResponse用于跟踪、 ExportMetricsServiceResponse指标和 ExportLogsServiceResponse日志）。  

如果响应成功，服务器必须保留 `partial_success` 字段未设置。  

##### 部分成功

如果请求仅被部分接受（即，当服务器仅接受部分数据并拒绝其余数据时），则服务器响应必须与 [完全成功](#full-success) 情况中的 [Export\<signal>ServiceResponse](../opentelemetry/proto/collector) 消息相同。

此外，服务器必须初始化该`partial_success`字段（`ExportTracePartialSuccess` 跟踪消息、 `ExportMetricsPartialSuccess`指标消息和 `ExportLogsPartialSuccess`日志消息），并且必须设置相应的 `rejected_spans`,`rejected_data_points`或`rejected_log_records`字段及其拒绝的跨度/数据点/日志记录的数量。  

服务器`error_message`应该用人类可读的英文错误消息填充该字段。该消息应解释服务器拒绝部分数据的原因，并可能提供有关用户如何解决问题的指导。该协议不会尝试定义错误消息的结构。  

`partial_success`即使服务器完全接受请求，服务器也可以使用该字段向客户端传达警告/建议。在这种情况下，该`rejected_<signal>`字段的值必须为0，并且该`error_message`字段必须为非空。  

当客户端收到填充的部分成功响应时，不得重试请求`partial_success`。  

##### 失败

当服务器返回错误时，它分为两大类：可重试和不可重试：

- 可重试错误表示遥测数据处理失败，客户端应该记录错误并可以重试导出相同的数据。例如，当服务器暂时无法处理数据时，可能会发生这种情况。

- 不可重试的错误表示遥测数据处理失败，客户端不得重试发送相同的遥测数据。客户端必须丢弃遥测数据。例如，当请求包含错误数据并且服务器无法反序列化或处理时，就会发生这种情况。客户端应该维护此类丢弃数据的计数器。

服务器必须使用代码 [Unavailable](https://godoc.org/google.golang.org/grpc/codes) 指示可重试错误，并且可以 使用 包含 0 值的 [RetryInfo](https://github.com/googleapis/googleapis/blob/6a8c7914d1b79bd832b5157a09a9332e8cbd16d4/google/rpc/error_details.proto#L40) 的 [details via status](https://godoc.org/google.golang.org/grpc/status#Status.WithDetails) 提供其他 详细信息 。下面是一个示例 Go 代码来说明：

```go
  // Do this on server side.
  st, err := status.New(codes.Unavailable, "Server is unavailable").
    WithDetails(&errdetails.RetryInfo{RetryDelay: &duration.Duration{Seconds: 0}})
  if err != nil {
    log.Fatal(err)
  }

  return st.Err()
```

为了指示不可重试的错误，建议服务器使用代码 [InvalidArgument](https://godoc.org/google.golang.org/grpc/codes)  并可以 使用 [BadRequest](https://github.com/googleapis/googleapis/blob/6a8c7914d1b79bd832b5157a09a9332e8cbd16d4/google/rpc/error_details.proto#L119) [details via status](https://godoc.org/google.golang.org/grpc/status#Status.WithDetails) 提供其他详细信息。如果更合适，可以使用另一个 gRPC 状态代码。下面是一段示例 Go 代码片段来说明：

```go
  // Do this on the server side.
  st, err := status.New(codes.InvalidArgument, "Invalid Argument").
    WithDetails(&errdetails.BadRequest{})
  if err != nil {
    log.Fatal(err)
  }

  return st.Err()
```

如果其他 gRPC 代码更适合特定的错误情况，服务器可以使用其他 gRPC 代码来指示可重试和不可重试的错误。客户端应该根据下表将 gRPC 状态代码解释为可重试或不可重试：  

|gRPC Code|Retryable?|
|---------|----------|
|CANCELLED|Yes|
|UNKNOWN|No|
|INVALID_ARGUMENT|No|
|DEADLINE_EXCEEDED|Yes|
|NOT_FOUND|No|
|ALREADY_EXISTS|No|
|PERMISSION_DENIED|No|
|UNAUTHENTICATED|No|
|RESOURCE_EXHAUSTED|Only if the server can recover (see below)|
|FAILED_PRECONDITION|No|
|ABORTED|Yes|
|OUT_OF_RANGE|Yes|
|UNIMPLEMENTED|No|
|INTERNAL|No|
|UNAVAILABLE|Yes|
|DATA_LOSS|Yes|

重试时，客户端应该实施 exponential backoff 策略。一个例外是下面解释的限制情况，它提供了有关重试间隔的明确说明。

`RESOURCE_EXHAUSTED` 仅当服务器发出信号表明可以从资源耗尽中恢复时，客户端才应将代码解释为可重试。服务器通过返回 包含 [RetryInfo](https://github.com/googleapis/googleapis/blob/6a8c7914d1b79bd832b5157a09a9332e8cbd16d4/google/rpc/error_details.proto#L40) 的 [状态](https://godoc.org/google.golang.org/grpc/status#Status.WithDetails) 来发出信号。在这种情况下，服务器和客户端的行为与 [OTLP/gRPC 节流](#otlpgrpc-throttling) 部分中所述完全相同 。如果没有返回这样的状态，那么 `RESOURCE_EXHAUSTED` 代码应该被视为不可重试。

#### OTLP/gRPC 节流

OTLP 允许 backpressure （背压） 信号。

如果服务器无法跟上从客户端接收的数据的速度，那么它应该向客户端表明这一事实。然后，客户端必须限制自身以避免服务器不堪重负。  


为了在使用 gRPC 传输时发出 backpressure 信号，服务器必须返回代码为 [Unavailable](https://godoc.org/google.golang.org/grpc/codes) 的错误，并且可以 使用 [RetryInfo](https://github.com/googleapis/googleapis/blob/6a8c7914d1b79bd832b5157a09a9332e8cbd16d4/google/rpc/error_details.proto#L40) 在 [details via status](https://godoc.org/google.golang.org/grpc/status#Status.WithDetails) 中提供其他 详细信息。下面是一段示例 Go 代码片段来说明：

```go
  // Do this on the server side.
  st, err := status.New(codes.Unavailable, "Server is unavailable").
    WithDetails(&errdetails.RetryInfo{RetryDelay: &duration.Duration{Seconds: 30}})
  if err != nil {
    log.Fatal(err)
  }

  return st.Err()

  ...

  // Do this on the client side.
  st := status.Convert(err)
  for _, detail := range st.Details() {
    switch t := detail.(type) {
    case *errdetails.RetryInfo:
      if t.RetryDelay.Seconds > 0 || t.RetryDelay.Nanos > 0 {
        // Wait before retrying.
      }
    }
  }
```
当客户端收到此信号时，它应该遵循 [RetryInfo](https://github.com/googleapis/googleapis/blob/6a8c7914d1b79bd832b5157a09a9332e8cbd16d4/google/rpc/error_details.proto#L40) 文档中概述的建议：  

```
// Describes when the clients can retry a failed request. Clients could ignore
// the recommendation here or retry when this information is missing from the error
// responses.
//
// It's always recommended that clients should use exponential backoff when
// retrying.
//
// Clients should wait until `retry_delay` amount of time has passed since
// receiving the error response before retrying.  If retrying requests also
// fail, clients should use an exponential backoff scheme to increase gradually
// the delay between retries based on `retry_delay` until either a maximum
// number of retries has been reached, or a maximum retry delay cap has been
// reached.
```

`retry_delay` 的值由服务器确定并且取决于实现。服务器应该选择一个retry_delay足够大的值，以便为服务器提供恢复时间，但又不能太大而导致客户端在受到限制时丢失数据。

#### OTLP/gRPC Service 和 Protobuf 定义

gRPC service 定义 [在此处](../opentelemetry/proto/collector).

请求和响应的 Protobuf 消息定义 [在此处](../opentelemetry/proto).

请务必检查原型版本和  [maturity level](../README.md#maturity-level)。不同信号的模式可能处于不同的成熟度级别 - 有些是稳定的，有些是测试版。  

#### OTLP/gRPC Default Port

OTLP/gRPC 的默认网络端口是 4317。

### OTLP/HTTP

OTLP/HTTP 使用以  [二进制格式](#binary-protobuf-encoding) 或  [JSON 格式](#json-protobuf-encoding) 格式编码的 Protobuf 有效负载。无论编码如何，对于 OTLP/HTTP 和 OTLP/gRPC，消息的 Protobuf 模式都是相同的，如 [此处定义](../opentelemetry/proto)。

OTLP/HTTP 使用 HTTP POST 请求将遥测数据从客户端发送到服务器。实现可以使用 HTTP/1.1 或 HTTP/2 传输。如果无法建立 HTTP/2 连接，使用 HTTP/2 传输的实现应该回退到 HTTP/1.1 传输。

#### 二进制 Protobuf 编码

二进制 Protobuf 编码的有效负载使用 proto3 [编码标准](https://developers.google.com/protocol-buffers/docs/encoding)。  

发送二进制 Protobuf 编码的有效负载时，客户端和服务器必须设置“Content-Type: application/x-protobuf”请求和响应标头。

#### JSON Protobuf 编码

JSON Protobuf 编码的有效负载使用 proto3 标准定义的 [JSON Mapping](https://developers.google.com/protocol-buffers/docs/proto3#json) 来进行 Protobuf 和 JSON 之间的映射，映射时存在以下偏差：  

- `traceId`和`spanId`字节数组被表示为[不区分大小写的十六进制编码字符串](https://tools.ietf.org/html/rfc4648#section-8)；它们并未按照标准的[Protobuf JSON 映射](https://developers.google.com/protocol-buffers/docs/proto3#json)定义为 base64 编码。所有 OTLP Protobuf 消息中的 `traceId` 和 `spanId` 字段都使用十六进制编码，例如 `Span`，`Link`，`LogRecord`等消息。例如，Span 中的 `traceId` 字段可以这样表示：{ "traceId": "5B8EFFF798038103D269B633813FC60C", ... }

- 枚举字段的值必须被编码为整数值。与标准的[Protobuf JSON 映射](https://developers.google.com/protocol-buffers/docs/proto3#json)不同，它允许枚举字段的值被编码为整数值或枚举名称字符串，OTLP JSON Protobuf 编码只允许整数枚举值；枚举名称字符串不能被使用。例如，Span 中值为 SPAN_KIND_SERVER 的 `kind` 字段可以这样表示：{ "kind": 2, ... }

- OTLP/JSON 接收器必须忽略具有未知名称的消息字段，并且必须解组消息，就像未知字段并未出现在有效载荷中一样。这与二进制 Protobuf 解组器的行为保持一致，并确保向 OTLP 消息添加新字段不会破坏现有接收器。

- JSON 对象的键是转换为 lowerCamelCase 的字段名称。原始字段名称不能用作 JSON 对象的键。
  例如，这是资源的有效 JSON 表示：`{ "attributes": {...}, "droppedAttributesCount": 123 }`，而这不是有效的表示：`{ "attributes": {...}, "dropped_attributes_count": 123 }`。


请注意，根据[Protobuf规范](https://developers.google.com/protocol-buffers/docs/proto3#json)，JSON编码有效负载中的64位整数编码为十进制字符串，解码时接受数字或字符串。

客户端和服务器在发送JSON Protobuf编码有效负载时，必须设置“Content-Type: application/json”请求和响应头。

有关JSON有效负载示例，请参阅：[OTLP JSON请求示例](../examples/README.md)


#### OTLP/HTTP 请求

遥测数据通过HTTP POST请求发送。POST请求的主体是二进制编码的Protobuf格式或JSON编码的Protobuf格式的有效负载。

携带跟踪数据的请求的默认URL路径为`/v1/traces`（例如，连接到“example.com”服务器时的完整URL将为`https://example.com/v1/traces`）。请求主体是Protobuf编码的`ExportTraceServiceRequest`消息。

携带度量数据的请求的默认URL路径为`/v1/metrics`，请求主体是Protobuf编码的`ExportMetricsServiceRequest`消息。

携带日志数据的请求的默认URL路径为`/v1/logs`，请求主体是Protobuf编码的`ExportLogsServiceRequest`消息。

客户端可以对内容进行gzip压缩，在这种情况下，必须包含“Content-Encoding: gzip”请求头。如果客户端可以接收gzip编码的响应，则可以包含“Accept-Encoding: gzip”请求头。

客户端和服务器端可以配置非默认的请求URL路径。

#### OTLP/HTTP 响应

响应正文必须是适当的序列化 Protobuf 消息（请参阅
下面 [完全成功](#full-success-1)，
[部分成功](#partial-success-1) 和[失败](#failures-1) 案例）。  

服务器必须在响应体为二进制编码的Protobuf负载时设置"Content-Type: application/x-protobuf"头。如果响应为JSON编码的Protobuf负载，服务器必须设置"Content-Type: application/json"。服务器必须在响应中使用与请求中相同的"Content-Type"。

如果请求头中存在 "Accept-Encoding: gzip"，服务器可以对响应进行gzip编码，并设置响应头 "Content-Encoding: gzip"。

##### 完全成功

成功响应表明服务器已成功接收遥测数据。

如果服务器收到一个空请求（即一个不携带任何遥测数据的请求），服务器应该回应成功。

成功时，服务器必须响应`HTTP 200 OK`。响应体必须是一个Protobuf编码的 [Export\<signal>ServiceResponse](../opentelemetry/proto/collector) 消息（对于追踪是`ExportTraceServiceResponse`，对于指标是`ExportMetricsServiceResponse`，对于日志是`ExportLogsServiceResponse`）。

在成功响应的情况下，服务器必须保持`partial_success`字段未设置。

##### 部分成功

如果请求只被部分接受（即服务器只接受部分数据并拒绝其余部分），服务器必须响应`HTTP 200 OK`。响应体必须与 [完全成功](#full-success-1) 情况下的相同 [Export\<signal>ServiceResponse](../opentelemetry/proto/collector) 消息。

此外，服务器必须初始化`partial_success`字段（对于追踪是`ExportTracePartialSuccess`消息，对于指标是`ExportMetricsPartialSuccess`消息，对于日志是`ExportLogsPartialSuccess`消息），并且必须将相应的`rejected_spans`、`rejected_data_points`或`rejected_log_records`字段设置为其拒绝的 spans/data points/log 的数量。  

服务器应使用英语填充`error_message`字段，以提供一个易于理解的错误信息。该消息应解释服务器为什么拒绝了部分数据，并可能提供关于用户如何解决问题的指导。该协议并未试图定义错误消息的结构。

即使在完全接受请求时，服务器还可以使用`partial_success`字段向客户端传达警告/建议。在这种情况下，`rejected_<signal>`字段必须为`0`，并且`error_message`字段必须为非空。

当客户端收到填充了`partial_success`的部分成功响应时，客户端不得重试该请求。

##### 失败

如果请求处理失败，服务器必须使用适当的 `HTTP 4xx` 或 `HTTP 5xx` 状态码进行响应。有关特定故障情况和应使用的HTTP状态代码的详细信息，请参阅下面的各节。  

所有 `HTTP 4xx` 和 `HTTP 5xx` 响应的响应体必须是一个Protobuf编码的 [Status](https://godoc.org/google.golang.org/genproto/googleapis/rpc/status#Status) 消息，描述问题。  

此规范不使用 `Status.code` 字段，服务器可以省略 `Status.code` 字段。客户端不需要根据 `Status.code` 字段更改其行为，但可以记录故障排除目的。  

`Status.message` 字段应包含在 `Status` 消息模式中定义的面向开发人员的错误消息。  

服务器可以在 `Status.details` 字段中包含其他详细信息。在每个特定的故障情况下，阅读以下有关此字段可以包含的内容。  

服务器应使用HTTP响应状态码来指示特定错误情况下可重试和不可重试的错误。客户端应遵守HTTP响应状态码作为可重试或不可重试。收到以下表格中列出的响应状态码的请求应该被重试。  
所有其他 `4xx` 或 `5xx` 响应状态码都不应重试。  

|HTTP response status code|
|---------|
|429 Too Many Requests|
|502 Bad Gateway|
|503 Service Unavailable|
|504 Gateway Timeout|

##### 错误数据

如果由于请求包含无法解码或无效的数据而导致请求处理失败，并且这种失败是永久性的，则服务器必须使用 "HTTP 400 Bad Request" 响应。响应中的 "Status.details" 字段应包含一个描述不良数据的 [BadRequest](https://github.com/googleapis/googleapis/blob/d14bf59a446c14ef16e9931ebfc8e63ab549bf07/google/rpc/error_details.proto#L166)。

当客户端收到 "HTTP 400 Bad Request" 响应时，它必须不重试请求。

##### OTLP/HTTP 节流

如果服务器接收到的请求数超过了客户端允许的数量，或者服务器过载，则服务器应该使用 "HTTP 429 Too Many Requests" 或 "HTTP 503 Service Unavailable" 响应，并可能包括 ["Retry-After"](https://tools.ietf.org/html/rfc7231#section-7.1.3) 报头，其中包含在重试之前建议等待的时间间隔（以秒为单位）。

如果 "Retry-After" 报头存在，则客户端应遵循其中指定的等待间隔。如果客户端收到 "HTTP 429" 或 "HTTP 503" 响应，并且响应中没有 "Retry-After" 报头，则客户端应该在重试之间实现 exponential backoff 策略。

##### 所有其他响应
其他未在本文件中明确列出的HTTP响应应根据HTTP规范进行处理。

如果服务器没有返回响应就断开连接，客户端应该重试并发送相同的请求。客户端在重试之间应实现 exponential backoff 策略，以避免使服务器过载。

#### OTLP/HTTP 连接

如果客户端无法连接到服务器，客户端应该使用  exponential backoff 策略在重试之间尝试连接。重试之间的间隔必须具有随机抖动。

客户端应在请求之间保持连接。

服务器实现应接受在同一端口上使用二进制编码的Protobuf有效载荷和JSON编码的Protobuf有效载荷的OTLP/HTTP请求，并根据“Content-Type”请求头将请求多路复用到相应的有效载荷解码器。

服务器实现可以接受在同一端口上使用OTLP/gRPC和OTLP/HTTP请求，并根据“Content-Type”请求头将连接多路复用到相应的传输处理程序。

#### OTLP/HTTP 并发请求

为了实现更高的总吞吐量，客户端可以使用几个并行的HTTP连接发送请求。在这种情况下，最大并行连接数应该是可配置的。

#### OTLP/HTTP 默认端口

OTLP/HTTP 的默认端口是 4318.

## 实现建议

### 多目的地导出

当一个客户端必须将遥测数据发送到多个目标服务器时，必须考虑一个额外的复杂性。当其中一个服务器确认数据，而另一个服务器（尚未）确认数据时，客户端需要决定如何继续前进。

在这种情况下，客户端应该为每個目标实现队列、确认处理和重试逻辑。这确保服务器不会相互阻塞。队列应该引用要发送的共享、不可变数据，从而最小化由于多个队列而导致的内存开销。

![多目标导出](img/otlp-multi-destination.png)

这确保所有目标服务器都能接收到数据，而不管它们的接收速度（在客户端队列的大小所施加的可用限制范围内）。

## 已知限制

### 请求确认

#### 重复数据

在边界情况下（例如重新连接、网络中断等），如果客户端尚未收到确认，则无法知道最近发送的数据是否已接收。客户端通常会选择重新发送此类数据以确保交付，这可能导致服务器端的重复数据。这是一个深思熟虑的选择，被认为是遥测数据的最佳权衡。

## 未来版本和互操作性

OTLP将随着时间的推移不断发展和变化。未来版本的OTLP必须以確保实现不同版本的OTLP的客户端和服务器可以互操作并交换遥测数据的方式进行设计和实现。旧客户端必须能够与新的服务器进行通信，反之亦然。假设新版本的OTLP引入了旧版本节点无法理解和支持的新的功能。在这种情况下，协议必须从功能性的角度回归到最低公倍数。

在可能的情况下，必须确保所有未声明废弃的版本之间的互操作性。

OTLP不使用显式的协议版本号。OTLP的不同版本的客户端和服务器的互操作性基于以下概念：

1. OTLP（当前和未来的版本）定义了一组能力，其中一些是强制性的，而另一些是可选的。客户端和服务器必须实现强制性的能力，并可以选择仅实现可选能力的一小部分。

2. 对于协议的次要更改，鼓励未来的OTLP版本和扩展使用Protobuf的能力，以向后兼容的方式进化消息模式。较新版本的OTLP可能会向消息中添加新字段，这些字段将被不理解这些字段的客户端和服务器忽略。在许多情况下，仔细设计此类模式更改以及为新字段正确选择默认值就足以确保不同版本之间的互操作性，而无需节点显式检测其对等节点是否有不同的能力。

3. 更重大的变化必须被明确地定义为未来OTEP中的新可选能力。这些能力应由客户端和服务器实现者在建立底层传输后自行发现。确切的发现机制应在未来的OTEP中描述，这些OTEP定义了新的能力，通常可以通过从客户端到服务器的发现请求/响应消息交换来实现。本规范定义的强制性能力是隐含的，不需要发现。支持新可选能力的实现必须调整其行为，以匹配不支持特定能力的对等方的期望。

## 词汇表

遥测数据交换涉及2方。在这份文档中，将遥测数据作为源的方称为“客户端”，将遥测数据作为目的地的方称为“服务器”。

![客户端-服务器](img/otlp-client-server.png)

客户端的例子包括装有仪器的应用程序或遥测收集器的发送端，服务器的例子包括遥测后端或遥测收集器的接收端（因此，收集器通常既是客户端也是服务器，具体取决于您从哪边看）。

客户端和服务器都是“节点”。这个术语在文档中用于指代任何一方。

## 参考文档

- [OTEP 0035](https://github.com/open-telemetry/oteps/blob/main/text/0035-opentelemetry-protocol.md) OpenTelemetry协议规范
- [OTEP 0099](https://github.com/open-telemetry/oteps/blob/main/text/0099-otlp-http.md) OTLP/HTTP：用于OTLP的HTTP传输扩展
- [OTEP 0122](https://github.com/open-telemetry/oteps/blob/main/text/0122-otlp-http-json.md) OTLP：用于OTLP/HTTP的JSON编码