**阅读目录**

* [〇、前言](#_label0)
* [一、文件系统 Sink 用法](#_label1)
  + [1.1 Serilog.Sinks.File：最基础的日志组件](#_label1_0)
  + [1.2 Serilog.Sinks.Async：异步操作来避免主程阻塞](#_label1_1)
    - [1.2.1 创建两个 .net 8.0 WebAPI 示例项目，然后添加几个动态库](#_label1_1_0)
    - [1.2.2 编辑 Program.cs](#_label1_1_1)
    - [1.2.3 测试与解析](#_label1_1_2)
  + [1.3 Serilog.Sinks.RollingFile：已过时，无需单独使用](#_label1_2)
  + [1.4 Serilog.Sinks.Map：根据日志属性的值将日志时间路由到不同的 Sink](#_label1_3)
    - [1.4.1 创建测试项目并添加必要的包](#_label1_3_0)
    - [1.4.2 修改 Program.cs](#_label1_3_1)
    - [1.4.3 测试结果验证](#_label1_3_2)

---

[回到顶部](#_labelTop):[芒果加速器](https://manguojiasu.com)

## 〇、前言

前文已经介绍过什么是 Serilog，以及其核心特点，*详见：[https://github.com/hnzhengfy/p/19167414/Serilog\_basic](https://github.com/hnzhengfy/p/19167414/Serilog_basic "https://github.com/hnzhengfy/p/19167414/Serilog_basic")。*

从本文开始，后续将对各种类型的 Sink 进行简单的实践。

本文将以文件系统相关的 Sink 为主进行介绍，针对多个相关的动态库，进行了简介以及示例项目实现，供参考。

[回到顶部](#_labelTop)

## 一、文件系统 Sink 用法

### 1.1 Serilog.Sinks.File：最基础的日志组件

Serilog.Sinks.File 是 Serilog 社区维护的核心文件日志接收器，用于将日志事件写入到一个或多个文本文件中。

它**支持文本和 JSON 格式输出**，保留结构化数据，是 Serilog 生态中最基础的文件日志组件。也支持**按时间（如每天、每周）或大小滚动日志文件**，可配置**多进程共享日志文件**，支持**写入缓冲**以提高性能。

如下是简单实践，可以简单分为四步：

**1）创建 .NET 8 Web API 项目，并安装 NuGet 包：**

```
|  |  |
| --- | --- |
|  | dotnet add package Serilog.AspNetCore |
|  | dotnet add package Serilog.Sinks.File |
|  | // dotnet sdd package Serilog.Enrichers.Environment // 有需求要添加机器名时，需要添加此包 |
|  | // dotnet sdd package Swashbuckle.AspNetCore // 若需要使用 Swagger 时，需添加此包 |
|  | // 注意：使用 Serilog.AspNetCore 而不是基础的 Serilog， |
|  | //   因为它为 ASP.NET Core 提供了更好的集成（如自动记录请求、依赖注入支持等） |
```

**2）添加 appsettings.json 配置（非必要）。**

虽然当前示例项目中，主要在代码中配置 Serilog，但也可以将部分设置放在 appsettings.json 中。

例如：

```
|  |  |
| --- | --- |
|  | { |
|  | "Serilog": { // 根节点 |
|  | "MinimumLevel": { // 定义了日志记录的最低级别，只有等于或高于此级别的日志才会被记录下来 |
|  | "Default": "Debug", // 全局默认的日志最低级别是 Debug |
|  | "Override": { // 为特定命名空间或类库重写默认的日志级别 |
|  | "Microsoft": "Warning", // 所有来自 Microsoft 命名空间（例如框架内部代码）的日志，只记录 Warning 及以上级别（即 Warning, Error, Fatal） |
|  | "Microsoft.AspNetCore": "Warning" // Microsoft.AspNetCore 相关组件（如 MVC、HTTP 请求管道等）也只记录 Warning 及以上级别 |
|  | } |
|  | } |
|  | }, |
|  | "AllowedHosts": "*" // 用于控制应用响应哪些 HTTP 主机头（Host header）的请求，"*" 表示允许所有主机访问 |
|  | // 在开发环境下通常设为 "*"，但在生产环境中建议明确指定受信任的主机以增强安全性 |
|  | } |
```

当全局默认的日志最低级别是 Debug 时，应用程序中所有代码（除非特别覆盖）产生的 Debug、Information、Warning、Error、Fatal 级别的日志都会被记录。

日志级别从低到高通常是：Verbose < Debug < Information < Warning < Error < Fatal。

ASP.NET Core 框架本身会产生大量关于Microsoft、Microsoft.AspNetCore 两个命名空间的 Information 和 Debug 级别的日志，如果不加限制，在生产环境中会非常冗长且影响性能。通过将其级别提高到 Warning，可以减少噪音日志，只关注真正重要的信息。

**3）修改 Program.cs（核心配置）**

```
|  |  |
| --- | --- |
|  | using Microsoft.AspNetCore.Builder; |
|  | using Serilog; |
|  | using Serilog.Formatting.Json; |
|  |  |
|  | var builder = WebApplication.CreateBuilder(args); |
|  |  |
|  | Log.Logger = new LoggerConfiguration() |
|  | .ReadFrom.Configuration(builder.Configuration) // 读取 appsettings.json 中的 Serilog 配置 |
|  | .Enrich.FromLogContext()                      // 启用上下文（如 BeginScope） |
|  | .Enrich.WithMachineName()                    // 添加机器名 // 需要包：Serilog.Enrichers.Environment |
|  | .Enrich.WithEnvironmentUserName()            // 添加用户名 |
|  | .WriteTo.Console(new JsonFormatter())        // 控制台也输出 JSON（可选，便于开发） |
|  | .WriteTo.File( |
|  | path: "logs/api-.log", |
|  | // formatter: new JsonFormatter(),           // 关键：使用 JSON 格式保留结构化数据 |
|  | formatter: new JsonFormatter(renderMessage: true), // renderMessage 加上此选项，日志中才会输出"RenderedMessage" |
|  | rollingInterval: RollingInterval.Day,     // 按天滚动 |
|  | fileSizeLimitBytes: 10_000_000,          // 单文件最大 10MB |
|  | retainedFileCountLimit: 31,              // 最多保留 31 天 |
|  | rollOnFileSizeLimit: true,               // 达到大小限制时也滚动 |
|  | shared: true                             // 允许多个进程写入（IIS/容器场景需要） |
|  | ) |
|  | .CreateLogger(); |
|  | // 告诉 ASP.NET Core 使用 Serilog 作为日志提供者 |
|  | builder.Host.UseSerilog(); |
|  |  |
|  | // Add services to the container. |
|  | builder.Services.AddControllers(); |
|  | builder.Services.AddEndpointsApiExplorer(); |
|  | builder.Services.AddSwaggerGen(); // 要使用 Swagger 需要添加包：Swashbuckle.AspNetCore |
|  |  |
|  | var app = builder.Build(); |
|  |  |
|  | // 配置 HTTP 请求管道 |
|  | if (app.Environment.IsDevelopment()) |
|  | { |
|  | app.UseSwagger(); |
|  | app.UseSwaggerUI(); |
|  | } |
|  |  |
|  | // Configure the HTTP request pipeline. |
|  | app.UseHttpsRedirection(); |
|  | app.UseAuthorization(); |
|  | app.MapControllers(); |
|  |  |
|  | // 在应用启动时记录一条结构化日志 |
|  | app.Lifetime.ApplicationStarted.Register(() => |
|  | { |
|  | Log.Information("Application started. Environment: {Environment}, ContentRoot: {ContentRoot}", |
|  | app.Environment.EnvironmentName, |
|  | app.Environment.ContentRootPath); |
|  | }); |
|  | // 在应用停止时记录 |
|  | app.Lifetime.ApplicationStopping.Register(() => |
|  | { |
|  | Log.Information("Application is stopping."); |
|  | }); |
|  |  |
|  | app.Run(); |
```

**4）修改 WeatherForecastController.cs**

```
|  |  |
| --- | --- |
|  | using Microsoft.AspNetCore.Mvc; |
|  |  |
|  | namespace Test.WebAPI._8._0.Serilog.WriteToFile.Controllers |
|  | { |
|  | [ApiController] |
|  | [Route("[controller]")] |
|  | public class WeatherForecastController : ControllerBase |
|  | { |
|  | private static readonly string[] Summaries = new[] |
|  | { |
|  | "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching" |
|  | }; |
|  | private readonly ILogger _logger; |
|  | public WeatherForecastController(ILogger logger) |
|  | { |
|  | _logger = logger; |
|  | } |
|  | [HttpGet] |
|  | public IEnumerable Get() |
|  | { |
|  | // 记录结构化日志 |
|  | _logger.LogInformation("Fetching weather forecasts for user {UserId} from IP {ClientIp}", |
|  | HttpContext.User.Identity?.Name ?? "Anonymous", |
|  | HttpContext.Connection.RemoteIpAddress?.ToString()); |
|  | try |
|  | { |
|  | var rng = new Random(); |
|  | var forecasts = Enumerable.Range(1, 5).Select(index => new WeatherForecast |
|  | { |
|  | Date = DateTime.Now.AddDays(index), |
|  | TemperatureC = rng.Next(-20, 55), |
|  | Summary = Summaries[rng.Next(Summaries.Length)] |
|  | }).ToArray(); |
|  | // 记录复杂对象（使用 @ 解构） |
|  | _logger.LogDebug("Generated {ForecastCount} forecasts: {@Forecasts}", |
|  | forecasts.Length, forecasts); |
|  | return forecasts; |
|  | } |
|  | catch (Exception ex) |
|  | { |
|  | // 记录异常和上下文 |
|  | _logger.LogError(ex, "An error occurred while generating weather forecasts for user {UserId}", |
|  | HttpContext.User.Identity?.Name ?? "Anonymous"); |
|  | throw; |
|  | } |
|  | } |
|  | } |
|  | public class WeatherForecast |
|  | { |
|  | public DateTime Date { get; set; } |
|  | public int TemperatureC { get; set; } |
|  | public int TemperatureF => 32 + (int)(TemperatureC / 0.5556); |
|  | public string? Summary { get; set; } |
|  | } |
|  | } |
```

输出结果：

*注：实际输出为默认的 json 字符串，如下是格式化后便于查看。*

```
|  |  |
| --- | --- |
|  | { |
|  | "Timestamp": "2025-10-23T23:30:44.9346095+08:00", |
|  | "Level": "Information", |
|  | "MessageTemplate": "Fetching weather forecasts for user {UserId} from IP {ClientIp}", |
|  | "RenderedMessage": "Fetching weather forecasts for user \"Anonymous\" from IP \"::1\"", // formatter: new JsonFormatter(renderMessage: true),配置效果 |
|  | "TraceId": "8168e507fe989ade0dd4887075e3535e", |
|  | "SpanId": "9c78435a71c11ee6", |
|  | "Properties": { |
|  | "UserId": "Anonymous", |
|  | "ClientIp": "::1", |
|  | "SourceContext": "Test.WebAPI._8._0.Serilog.WriteToFile.Controllers.WeatherForecastController", |
|  | "ActionId": "15cb902a-b7c7-4b58-98e4-745dffec67e9", |
|  | "ActionName": "Test.WebAPI._8._0.Serilog.WriteToFile.Controllers.WeatherForecastController.Get (Test.WebAPI.8.0.Serilog.WriteToFile)", |
|  | "RequestId": "0HNGI94H06F88:00000001", |
|  | "RequestPath": "/weatherforecast", |
|  | "ConnectionId": "0HNGI94H06F88", |
|  | "MachineName": "WIN-J", |
|  | "EnvironmentUserName": "WIN-J\\Administrator" |
|  | } |
|  | } |
```

### 1.2 Serilog.Sinks.Async：异步操作来避免主程阻塞

Serilog.Sinks.Async 不是一个独立的 Sink，而是一个**异步包装器**，用于将任何其他同步的 Sink（如 Serilog.Sinks.File）包装在异步操作中，减少日志记录对主线程的阻塞（特别是 Web 请求线程）。

在高并发场景下，如果日志直接写入文件、数据库或网络服务，这些 I/O 操作是**同步阻塞**的。如果在每个请求中都记录日志，会拖慢响应速度，甚至影响吞吐量。

**Serilog.Sinks.Async 将日志记录工作委托给后台线程，减少 I/O 瓶颈对应用性能的影响，特别适用于文件和数据库等受 I/O 瓶颈影响的接收器。**

* **工作原理**

当程序调用 Log.Information() 时，日志事件被推入一个**内部的异步队列（默认大小是 1000 条）之后，立马返回**。一个后台线程（或线程池任务）从队列中取出日志，再交给被包装的 Sink（如 File Sink）处理。主线程无需等待写入完成，立即返回。

**另外需要注意，极端情况下可能丢失最后几条日志（程序崩溃时），需要合理配置队列大小和溢出策略。**

* **示例项目**

大概思路：分别生成两个.net8.0 WebAPI 项目，一个使用同步，一个使用异步来实现将同样的内容，以 json 字符串的格式写入到文本文件中。启动服务后，通过压测工具（本文使用：wrk）观察性能差异。

下边是实际操作的简要步骤：

#### **1.2.1 创建两个 .net 8.0 WebAPI 示例项目，然后添加几个动态库**

同步项目名称：Test.WebAPI.8.0.Serilog.WriteToFile。异步实现项目：Test.WebAPI.Net8.Serilog.FileAsync。

需要引用的动态库：

```
|  |  |
| --- | --- |
|  | // 两个项目都需要添加下边三个包： |
|  | dotnet add package Serilog.AspNetCore |
|  | dotnet add package Serilog.Sinks.File |
|  | dotnet add package Serilog.Formatting.Compact |
|  | // 异步实现另外添加 Async 专用包： |
|  | dotnet add package Serilog.Sinks.Async |
```

#### **1.2.2 编辑 Program.cs**

同步项目实现：

```
|  |  |
| --- | --- |
|  | using Serilog; |
|  | using Serilog.Events; |
|  | using Serilog.Formatting.Compact; |
|  |  |
|  | var builder = WebApplication.CreateBuilder(args); |
|  |  |
|  | // 【配置 Serilog（同步）】 |
|  | builder.Host.UseSerilog((context, config) => |
|  | { |
|  | config.WriteTo.File( |
|  | path: "logs-sync/log-.txt", |
|  | rollingInterval: RollingInterval.Day, |
|  | retainedFileCountLimit: 30, |
|  | // formatProvider: CultureInfo.InvariantCulture, |
|  | // outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj}{NewLine}{Exception}", |
|  | formatter: new RenderedCompactJsonFormatter(), // 配置使用json格式字符串输出 |
|  | restrictedToMinimumLevel: LogEventLevel.Information |
|  | ); |
|  | }); |
|  |  |
|  | var app = builder.Build(); |
|  |  |
|  | app.MapGet("/api/logSync", () => |
|  | { |
|  | Log.Information("Fetching weather forecasts for user {UserId} from IP {ClientIp}", |
|  | "Anonymous", |
|  | "This is IP"); |
|  | return "Log recorded (synchronous)"; |
|  | }); |
|  |  |
|  | app.Run(); |
```

异步项目实现：

```
|  |  |
| --- | --- |
|  | using Serilog; |
|  | using Serilog.Events; |
|  | using Serilog.Formatting.Compact; |
|  |  |
|  | var builder = WebApplication.CreateBuilder(args); |
|  |  |
|  | // 【配置 Serilog（异步）】 |
|  | builder.Host.UseSerilog((context, config) => |
|  | { |
|  | config.WriteTo.Async(a => a.File( |
|  | path: "logs-async/log-.txt", |
|  | rollingInterval: RollingInterval.Day, |
|  | retainedFileCountLimit: 30, |
|  | //formatProvider: CultureInfo.InvariantCulture, |
|  | //outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj}{NewLine}{Exception}", |
|  | formatter: new RenderedCompactJsonFormatter(), // 配置使用json格式字符串输出 |
|  | restrictedToMinimumLevel: LogEventLevel.Information |
|  | )); |
|  | }); |
|  |  |
|  | var app = builder.Build(); |
|  |  |
|  | app.MapGet("/api/logAsync", () => |
|  | { |
|  | Log.Information("Fetching weather forecasts for user {UserId} from IP {ClientIp}", |
|  | "Anonymous", |
|  | "This is IP"); |
|  | return "Log recorded (asynchronous)"; |
|  | }); |
|  |  |
|  | app.Run(); |
```

配置完成，同时启动两个项目，两个服务地址例如：

```
|  |  |
| --- | --- |
|  | // 服务地址： |
|  | http://localhost:5043/api/logSync |
|  | http://localhost:5001/api/logAsync |
```

*注：日志文件夹会在项目启动时自动创建，无需手动配置或新增。*

#### **1.2.3 测试与解析**

直接在 Windows 上通过 Docker 和 wrk 来进行的是两种方式的性能差异。

```
|  |  |
| --- | --- |
|  | // 安装和配置 Docker Desktop 步骤【略。。。】 |
|  | // 博主使用的镜像源地址："https://docker.xuanyuan.me"，也可以使用阿里和腾讯的，哪个效率高用哪个 |
|  | // 拉取 wrk 包 |
|  | C:\Users\Administrator>docker pull williamyeh/wrk |
|  | Using default tag: latest |
|  | latest: Pulling from williamyeh/wrk |
|  | 4fe2ade4980c: Pull complete |
|  | c4d7e348633d: Pull complete |
|  | 3e403d3ebdda: Pull complete |
|  | bdb672ee55d9: Pull complete |
|  | 2bfb714176a4: Pull complete |
|  | Digest: sha256:78adc0d9d51a99e6759e702a08d03eaece81c890ffcc9790ef9e5b199d54f091 |
|  | Status: Downloaded newer image for williamyeh/wrk:latest |
|  | docker.io/williamyeh/wrk:latest |
|  | // 异步记录日志测试 |
|  | // 注意：host.docker.internal 是 Docker 宿主机的特殊 DNS 名称，说明被测服务运行在宿主机上，而 wrk 在容器内发起请求 |
|  | C:\Users\Administrator>docker run -it --rm williamyeh/wrk -t12 -c100 -d10s http://host.docker.internal:5001/api/logAsync |
|  | Running 10s test @ http://host.docker.internal:5001/api/logAsync |
|  | 12 threads and 100 connections |
|  | Thread Stats   Avg      Stdev     Max   +/- Stdev |
|  | Latency    15.59ms    5.85ms 103.93ms   93.31% |
|  | Req/Sec   524.79    106.91   660.00     78.36% |
|  | 63040 requests in 10.01s, 10.82MB read |
|  | Requests/sec:   6296.32 |
|  | Transfer/sec:      1.08MB |
|  | // 同步记录日志测试 |
|  | C:\Users\Administrator>docker run -it --rm williamyeh/wrk -t12 -c100 -d10s http://host.docker.internal:5043/api/logSync |
|  | Running 10s test @ http://host.docker.internal:5043/api/logSync |
|  | 12 threads and 100 connections |
|  | Thread Stats   Avg      Stdev     Max   +/- Stdev |
|  | Latency    19.52ms    6.70ms  90.80ms   79.40% |
|  | Req/Sec   413.05     43.31   656.00     85.56% |
|  | 49576 requests in 10.05s, 8.46MB read |
|  | Requests/sec:   4930.52 |
|  | Transfer/sec:    862.05KB |
|  |  |
|  | C:\Users\Administrator> |
|  | // 关于 wrk -t12 -c100 -d10s |
|  | // 同时开启 12 个线程，每个线程与目标服务创建 100 个 TCP/HTTP keep-alive 连接，持续发起请求，持续 10 秒钟 |
|  | // 相当于，在同一时间，有 1200 个连接在持续请求（一个请求完成，接着继续发出下一个请求）目标服务 |
```

下边简单分析下测试结果。

| 指标 | 异步 (`/logAsync`) | 同步 (`/logSync`) | 对比 |
| --- | --- | --- | --- |
| **总请求数** | 63,040 | 49,576 | +27.1% ↑ |
| **吞吐量 (Requests/sec)** | **6,296.32** | **4,930.52** | **+27.7% 提升** |
| **传输速率 (Transfer/sec)** | 1.08 MB | 862.05 KB | +25.3% ↑ |
| **平均延迟 (Latency Avg)** | **15.59ms** | 19.52ms | **快 20%** |
| **最大延迟 (Max Latency)** | 103.93ms | 90.80ms | 略高（可能因队列堆积） |
| **延迟标准差 (Stdev)** | 5.85ms | 6.70ms | 更稳定 |
| **每秒请求数波动 (Req/Sec Stdev)** | ±106.91 | ±43.31 | 异步波动更大，但均值更高 |

其中，异步操作的最大延迟稍高。可能原因：异步任务积压在线程池或消息队列中，在高峰期出现短暂排队，导致个别请求的“真实完成时间”变长（但从 API 返回角度看仍是快的）。

异步虽然吞吐高，但单位时间内请求处理数量波动更大，可能是由于线程调度、异步任务调度引入了不确定性。

总结一下，就是异步显著优于同步（+27% QPS，-20% 延迟）。

### 1.3 Serilog.Sinks.RollingFile：已过时，无需单独使用

此动态库主要功能是，配置日志文件如何按规则滚动，但现在已经在 Serilog.Sink.File 中进行了实现（详见本文1.1），因此无需再使用此动态库。

这个变化是 Serilog 在 2.0 版本之后（特别是向 3.0 和 5.0 进化过程中）的一个重要的演变。

但是，**RollingFile 的功能并没有消失，而是被整合并增强到了 Serilog.Sinks.File 包中**。现在推荐的做法是：只安装 Serilog.Sinks.File 这一个包，它就能满足所有文件输出需求，包括强大的滚动功能。

Serilog.Sinks.File 的 File() 方法提供了以下关键参数来实现滚动：

**rollingInterval：**这是最主要的滚动策略参数。它是一个 RollingInterval 枚举，可选值包括：

  RollingInterval.Year：按年滚动。  RollingInterval.Month：按月滚动。  RollingInterval.Day：最常用，按天滚动。  RollingInterval.Hour：按小时滚动。  RollingInterval.Minute：按分钟滚动（适用于极高日志量的场景）。  *RollingInterval.Infinite：不滚动，所有日志写入单个文件。（注意：Infinite 在 Serilog 5.0 中以被移除，如需配置不滚动，可以通过略过此项配置来实现）。*

**rollOnFileSizeLimit：**一个布尔值。如果设置为 true，并且设置了 fileSizeLimitBytes，那么当日志文件达到指定大小时，也会触发滚动，即使当前时间间隔（如一天）还未结束。这实现了“按大小或时间”任一条件满足即滚动的逻辑。

**fileSizeLimitBytes：**设置单个日志文件的最大字节数。达到此大小后，如果 rollOnFileSizeLimit 为 true，则会创建新文件。

**文件名中的 {Date} 占位符：**当使用 rollingInterval 时，通常在文件路径中包含 {Date}。Serilog 会自动用当前日期（格式为 yyyyMMdd）替换它，从而生成如 log-20251026.txt 的文件名。

### 1.4 Serilog.Sinks.Map：根据日志属性的值将日志时间路由到不同的 Sink

Map 允许**根据日志属性的值将日志事件路由到不同的 Sink**。例如，**可以根据 UserId 将日志写入不同的文件，非常适合需要按用户或租户隔离日志的场景**。

因此，如果需要根据不同上下文（如用户ID、请求ID、租户ID等）将日志分发到不同的目标文件或处理逻辑，Map 是非常有用的；否则，一般不需要。

需要注意的地方：

**性能开销：**Map 内部使用字典缓存子 logger，频繁变化的键可能导致内存增长。  **过度设计风险：**大多数场景下，统一结构化日志 + 集中式日志分析工具（如 Seq、ELK）更合适。  **Json 写入本身不需要 Map：**你完全可以用 `WriteTo.File(..., formatter: new JsonFormatter())` 直接输出 JSON 日志。

典型应用场景就是，按 UserId 将审计日志写入不同日志文件。

#### 1.4.1 创建测试项目并添加必要的包

```
|  |  |
| --- | --- |
|  | // 添加下边三个包： |
|  | dotnet add package Serilog.AspNetCore |
|  | dotnet add package Serilog.Sinks.File |
|  | dotnet add package Serilog.Sink.Map |
```

#### 1.4.2 修改 Program.cs

```
|  |  |
| --- | --- |
|  | using Serilog; |
|  | using Serilog.Context; |
|  | using Serilog.Formatting.Compact; |
|  | using Serilog.Formatting.Json; |
|  |  |
|  | var builder = WebApplication.CreateBuilder(args); |
|  |  |
|  | // 配置 Serilog |
|  | Log.Logger = new LoggerConfiguration() |
|  | .Enrich.FromLogContext() |
|  | .MinimumLevel.Information() |
|  | // 主日志：所有日志汇总到一个 JSON 文件 |
|  | .WriteTo.File( |
|  | new JsonFormatter(), |
|  | "logs/app-all-.json", |
|  | rollingInterval: RollingInterval.Day |
|  | ) |
|  | // 特殊需求：按 UserId 分别记录审计日志到独立 JSON 文件 |
|  | .WriteTo.Map("UserId", "unknown", (id, wt) => |
|  | { |
|  | wt.File( |
|  | path: $"logs/audit/user_{id}-.json", |
|  | formatter: new RenderedCompactJsonFormatter(), // 输出为 JSON 格式 |
|  | rollingInterval: RollingInterval.Day, |
|  | fileSizeLimitBytes: 10_000_000, |
|  | rollOnFileSizeLimit: true |
|  | ); |
|  | }) |
|  | .CreateLogger(); |
|  | builder.Host.UseSerilog(); // 使用 Serilog 替代默认日志 |
|  |  |
|  | var app = builder.Build(); |
|  | // 模拟 API：记录带 UserId 的审计日志 |
|  | app.MapGet("/api/user/{id}/action", (int id, ILogger logger) => |
|  | { |
|  | using (LogContext.PushProperty("UserId", id)) // 设置上下文属性 |
|  | { |
|  | Log.Information("User performed an action {id}", id); |
|  | Log.Information("User viewed profile {TestParameters}", "测试字符串"); |
|  | } |
|  | // 也可以直接记录 |
|  | Log.ForContext("UserId", id).Information("Another action logged."); |
|  | return Results.Ok(new { Message = "Action logged", UserId = id }); |
|  | }); |
|  | app.Run(); |
```

#### 1.4.3 测试结果验证

```
|  |  |
| --- | --- |
|  | // 如下是两个测试地址，分别是用户 1001 和 1002 |
|  | http://localhost:5001/api/user/1001/action |
|  | http://localhost:5001/api/user/1002/action |
```

注意，实际记录的 json 格式日志是没有格式化的，下面展示下格式化后的 json，仅为了方便查看。

如下是 \logs\audit\user\_1001-20251031.json 文件内容：

```
|  |  |
| --- | --- |
|  | { |
|  | "@t": "2025-10-31T09:20:18.5714393Z", |
|  | "@m": "User performed an action 1001", |
|  | "@i": "0c7b7f7a", |
|  | "@tr": "b93dc8db0938aac572cf3d92d437deaf", |
|  | "@sp": "baa1c0bc7eb6880f", |
|  | "id": 1001, |
|  | "UserId": 1001, |
|  | "RequestId": "0HNGOBQMLDQHE:00000001", |
|  | "RequestPath": "/api/user/1001/action", |
|  | "ConnectionId": "0HNGOBQMLDQHE" |
|  | } |
|  | { |
|  | "@t": "2025-10-31T09:20:18.5783697Z", |
|  | "@m": "User viewed profile \"测试字符串\"", |
|  | "@i": "acc2ba35", |
|  | "@tr": "b93dc8db0938aac572cf3d92d437deaf", |
|  | "@sp": "baa1c0bc7eb6880f", |
|  | "TestParameters": "测试字符串", |
|  | "UserId": 1001, |
|  | "RequestId": "0HNGOBQMLDQHE:00000001", |
|  | "RequestPath": "/api/user/1001/action", |
|  | "ConnectionId": "0HNGOBQMLDQHE" |
|  | } |
|  | { |
|  | "@t": "2025-10-31T09:20:18.5794363Z", |
|  | "@m": "Another action logged.", |
|  | "@i": "58648cce", |
|  | "@tr": "b93dc8db0938aac572cf3d92d437deaf", |
|  | "@sp": "baa1c0bc7eb6880f", |
|  | "UserId": 1001, |
|  | "RequestId": "0HNGOBQMLDQHE:00000001", |
|  | "RequestPath": "/api/user/1001/action", |
|  | "ConnectionId": "0HNGOBQMLDQHE" |
|  | } |
```

另外两个日志文件内容就略过了：

*\logs\app-all-20251031.json*

*\logs\audit\user\_1002-20251031.json*

输出日志内容为 json 格式，然后就可以用 ELK / Seq / Loki 等工具做结构化解析和查询，这才是现代日志实践。

以上，就是对文件系统 Sink 的全部实践记录了，后续还有其他类型的 Sink 持续挖掘中。
