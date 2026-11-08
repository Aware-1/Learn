# راهنمای جامع API Architectures و Web Communication

## فهرست مطالب
1. انواع معماری‌های API
2. HTTP Methods عمیق
3. Headers در هر نوع API
4. Cookies و Authentication
5. Hybrid Approaches
6. ASP.NET Core عمیق
7. Angular 21 و HttpClient
8. مقایسه کامل و Best Practices

---

# بخش 1️⃣: انواع معماری‌های API

## 1.1. REST (Representational State Transfer)

### ویژگی‌ها
- بر پایه HTTP
- Stateless (بدون وضعیت)
- استفاده از HTTP Methods استاندارد
- منابع با URL مشخص می‌شوند

### ساختار Request
```http
GET /api/users/123 HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer eyJhbGc...
Cookie: session_id=abc123
User-Agent: Mozilla/5.0
```

### Headers مهم در REST
| Header | نقش | الزامی؟ | مثال |
|--------|-----|---------|------|
| **Content-Type** | نوع داده ارسالی | بله (POST/PUT) | `application/json` |
| **Accept** | نوع داده دریافتی | خیر | `application/json` |
| **Authorization** | احراز هویت | بسته به endpoint | `Bearer token...` |
| **Cookie** | Session/Auth | خیر | `sessionId=xyz` |
| **Cache-Control** | کنترل کش | خیر | `no-cache` |
| **ETag** | Versioning | خیر | `"33a64df551425fcc"` |
| **X-API-Key** | کلید API | بسته به سرویس | `abc123xyz` |
| **User-Agent** | شناسایی کلاینت | خیر | `MyApp/1.0` |
| **Origin** | CORS | بله (از مرورگر) | `https://myapp.com` |

### استفاده از Cookie در REST
```http
# Request
POST /api/auth/login HTTP/1.1
Content-Type: application/json

{
  "username": "user",
  "password": "pass"
}

# Response
HTTP/1.1 200 OK
Set-Cookie: auth_token=xyz123; HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age=86400

# درخواست بعدی (مرورگر خودکار Cookie رو میفرسته)
GET /api/users/me HTTP/1.1
Cookie: auth_token=xyz123
```

### مثال کامل REST
```csharp
// ASP.NET Core
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll(
        [FromQuery] int page = 1,
        [FromQuery] int size = 10,
        [FromHeader(Name = "X-Tenant-Id")] string tenantId = null)
    {
        var apiKey = Request.Headers["X-API-Key"].ToString();
        var userAgent = Request.Headers["User-Agent"].ToString();
        var cookie = Request.Cookies["session_id"];
        
        return Ok(new { page, size, tenantId, apiKey });
    }
    
    [HttpPost]
    public IActionResult Create(
        [FromBody] CreateUserDto dto,
        [FromHeader] string authorization)
    {
        Response.Headers.Add("X-Request-Id", Guid.NewGuid().ToString());
        Response.Headers.Add("X-RateLimit-Remaining", "99");
        
        return CreatedAtAction(nameof(GetById), new { id = 123 }, dto);
    }
}
```

---

## 1.2. GraphQL

### ویژگی‌ها
- یک endpoint واحد (معمولا `/graphql`)
- Query language برای دریافت دقیق داده
- همیشه POST (حتی برای خواندن!)
- Schema-based

### ساختار Request
```http
POST /graphql HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer eyJhbGc...
X-Apollo-Tracing: 1

{
  "query": "query GetUser($id: ID!) { user(id: $id) { name email } }",
  "variables": { "id": "123" },
  "operationName": "GetUser"
}
```

### Headers مهم در GraphQL
| Header | نقش | الزامی؟ | مثال |
|--------|-----|---------|------|
| **Content-Type** | همیشه JSON | بله | `application/json` |
| **Authorization** | احراز هویت | بسته به Query | `Bearer token...` |
| **X-Apollo-Tracing** | Performance tracking | خیر | `1` |
| **X-Request-ID** | Tracing | خیر | `uuid` |
| **Cookie** | Session | خیر | `token=xyz` |
| **Accept-Encoding** | فشرده‌سازی | خیر | `gzip, deflate` |

### استفاده از Cookie در GraphQL
```http
# Login Mutation
POST /graphql HTTP/1.1
Content-Type: application/json

{
  "query": "mutation { login(email: \"user@test.com\", password: \"pass\") { token user { id name } } }"
}

# Response
HTTP/1.1 200 OK
Set-Cookie: gql_token=xyz123; HttpOnly; Secure; Path=/

# Query بعدی
POST /graphql HTTP/1.1
Cookie: gql_token=xyz123
Content-Type: application/json

{
  "query": "{ me { id name email } }"
}
```

### مثال ASP.NET Core با HotChocolate
```csharp
// Query Type
public class Query
{
    public async Task<User> GetUser(
        int id,
        [Service] IHttpContextAccessor contextAccessor)
    {
        var httpContext = contextAccessor.HttpContext;
        
        // دسترسی به Headers
        var authorization = httpContext.Request.Headers["Authorization"];
        var tracing = httpContext.Request.Headers["X-Apollo-Tracing"];
        
        // دسترسی به Cookies
        var token = httpContext.Request.Cookies["gql_token"];
        
        return await GetUserById(id);
    }
}

// Mutation Type
public class Mutation
{
    public async Task<LoginResult> Login(
        string email, 
        string password,
        [Service] IHttpContextAccessor contextAccessor)
    {
        var user = await AuthenticateUser(email, password);
        
        if (user != null)
        {
            var token = GenerateToken(user);
            
            // تنظیم Cookie
            contextAccessor.HttpContext.Response.Cookies.Append(
                "gql_token", 
                token,
                new CookieOptions 
                { 
                    HttpOnly = true, 
                    Secure = true,
                    SameSite = SameSiteMode.Strict,
                    MaxAge = TimeSpan.FromDays(7)
                });
            
            return new LoginResult { Success = true, User = user };
        }
        
        return new LoginResult { Success = false };
    }
}

// Startup.cs
services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddMutationType<Mutation>()
    .AddHttpRequestInterceptor<AuthInterceptor>();

// Interceptor برای خواندن Cookie
public class AuthInterceptor : DefaultHttpRequestInterceptor
{
    public override ValueTask OnCreateAsync(
        HttpContext context,
        IRequestExecutor requestExecutor,
        IQueryRequestBuilder requestBuilder,
        CancellationToken cancellationToken)
    {
        var token = context.Request.Cookies["gql_token"];
        
        if (!string.IsNullOrEmpty(token))
        {
            requestBuilder.SetProperty("token", token);
        }
        
        return base.OnCreateAsync(context, requestExecutor, requestBuilder, cancellationToken);
    }
}
```

---

## 1.3. gRPC (Google Remote Procedure Call)

### ویژگی‌ها
- بر پایه HTTP/2
- استفاده از Protocol Buffers (binary)
- بسیار سریع و کارآمد
- Strongly-typed

### ساختار Request
```http
POST /UserService/GetUser HTTP/2
Host: api.example.com
Content-Type: application/grpc
TE: trailers
Grpc-Timeout: 1S
Authorization: Bearer token
Grpc-Metadata-User-Id: 123
```

### Headers مهم در gRPC
| Header | نقش | الزامی؟ | مثال |
|--------|-----|---------|------|
| **Content-Type** | همیشه grpc | بله | `application/grpc` |
| **TE** | Trailer support | بله | `trailers` |
| **Grpc-Timeout** | Timeout | خیر | `1000m` (میلی‌ثانیه) |
| **Grpc-Encoding** | فشرده‌سازی | خیر | `gzip` |
| **Grpc-Accept-Encoding** | انواع فشرده‌سازی | خیر | `gzip, deflate` |
| **Grpc-Metadata-*** | Custom metadata | خیر | هر چیزی |
| **Authorization** | احراز هویت | خیر | `Bearer token` |

### استفاده از Cookie در gRPC
⚠️ **gRPC به صورت مستقیم از Cookie پشتیبانی نمی‌کند!**

راه‌حل‌ها:
1. استفاده از Metadata برای ارسال token
2. Interceptor برای خواندن/نوشتن Cookie در HTTP layer

```csharp
// Proto file
syntax = "proto3";

service UserService {
  rpc GetUser (GetUserRequest) returns (UserResponse);
  rpc Login (LoginRequest) returns (LoginResponse);
}

message GetUserRequest {
  int32 id = 1;
}

message UserResponse {
  int32 id = 1;
  string name = 2;
  string email = 3;
}

// Server Implementation
public class UserService : UserService.UserServiceBase
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    
    public override Task<UserResponse> GetUser(
        GetUserRequest request, 
        ServerCallContext context)
    {
        // خواندن Metadata (Headers)
        var authHeader = context.RequestHeaders.GetValue("authorization");
        var userId = context.RequestHeaders.GetValue("grpc-metadata-user-id");
        
        // دسترسی به HttpContext برای Cookie
        var httpContext = _httpContextAccessor.HttpContext;
        var cookie = httpContext?.Request.Cookies["auth_token"];
        
        return Task.FromResult(new UserResponse 
        { 
            Id = request.Id,
            Name = "User Name"
        });
    }
    
    public override Task<LoginResponse> Login(
        LoginRequest request,
        ServerCallContext context)
    {
        // احراز هویت
        var token = AuthenticateUser(request.Email, request.Password);
        
        if (token != null)
        {
            var httpContext = _httpContextAccessor.HttpContext;
            
            // تنظیم Cookie
            httpContext?.Response.Cookies.Append(
                "auth_token",
                token,
                new CookieOptions 
                { 
                    HttpOnly = true,
                    Secure = true 
                });
            
            // یا اضافه کردن به Response Trailers
            context.ResponseTrailers.Add("set-token", token);
            
            return Task.FromResult(new LoginResponse { Success = true });
        }
        
        return Task.FromResult(new LoginResponse { Success = false });
    }
}

// Interceptor برای خواندن Cookie
public class AuthInterceptor : Interceptor
{
    public override Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request,
        ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        // خواندن Cookie از HttpContext
        var httpContext = context.GetHttpContext();
        var token = httpContext.Request.Cookies["auth_token"];
        
        if (!string.IsNullOrEmpty(token))
        {
            // اضافه کردن به Metadata
            var metadata = new Metadata
            {
                { "authorization", $"Bearer {token}" }
            };
            
            // ساخت context جدید با metadata
            var newContext = new ServerCallContext(/* ... */);
            return continuation(request, newContext);
        }
        
        return continuation(request, context);
    }
}

// Startup.cs
services.AddGrpc(options =>
{
    options.Interceptors.Add<AuthInterceptor>();
});
```

### Client-Side gRPC
```csharp
// C# Client
var channel = GrpcChannel.ForAddress("https://api.example.com");
var client = new UserService.UserServiceClient(channel);

// ارسال Metadata (Headers)
var headers = new Metadata
{
    { "authorization", "Bearer token123" },
    { "grpc-metadata-user-id", "456" }
};

var response = await client.GetUserAsync(
    new GetUserRequest { Id = 123 },
    headers: headers);

// ارسال "Cookie" از طریق Metadata
var cookieHeaders = new Metadata
{
    { "cookie", "auth_token=xyz123" }
};

var responseWithCookie = await client.GetUserAsync(
    new GetUserRequest { Id = 123 },
    headers: cookieHeaders);
```

---

## 1.4. SOAP (Simple Object Access Protocol)

### ویژگی‌ها
- بر پایه XML
- Protocol سنگین
- WSDL برای تعریف سرویس
- معمولا در سیستم‌های Enterprise قدیمی

### ساختار Request
```http
POST /UserService.asmx HTTP/1.1
Host: api.example.com
Content-Type: text/xml; charset=utf-8
SOAPAction: "http://tempuri.org/GetUser"
Content-Length: 500

<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Header>
    <Authentication xmlns="http://tempuri.org/">
      <Username>user</Username>
      <Password>pass</Password>
    </Authentication>
  </soap:Header>
  <soap:Body>
    <GetUser xmlns="http://tempuri.org/">
      <userId>123</userId>
    </GetUser>
  </soap:Body>
</soap:Envelope>
```

### Headers مهم در SOAP
| Header | نقش | الزامی؟ | مثال |
|--------|-----|---------|------|
| **Content-Type** | نوع XML | بله | `text/xml` یا `application/soap+xml` |
| **SOAPAction** | عملیات SOAP | بله (SOAP 1.1) | `"http://tempuri.org/GetUser"` |
| **Content-Length** | طول محتوا | بله | `500` |
| **Cookie** | Session | خیر | `ASP.NET_SessionId=...` |

### استفاده از Cookie در SOAP
```http
# Request با Cookie
POST /UserService.asmx HTTP/1.1
Content-Type: text/xml
Cookie: ASP.NET_SessionId=abc123; AuthToken=xyz789

<?xml version="1.0"?>
<soap:Envelope>
  <soap:Body>
    <GetUser>
      <userId>123</userId>
    </GetUser>
  </soap:Body>
</soap:Envelope>

# Response با Set-Cookie
HTTP/1.1 200 OK
Set-Cookie: AuthToken=newtoken456; Path=/; HttpOnly
Content-Type: text/xml

<?xml version="1.0"?>
<soap:Envelope>
  <soap:Body>
    <GetUserResponse>
      <name>Ali</name>
    </GetUserResponse>
  </soap:Body>
</soap:Envelope>
```

### مثال ASP.NET Core
```csharp
// معمولا از WCF استفاده می‌شود، ولی برای ASP.NET Core:
[Route("UserService.asmx")]
public class SoapController : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> ProcessSoapRequest()
    {
        // خواندن Cookie
        var sessionId = Request.Cookies["ASP.NET_SessionId"];
        var authToken = Request.Cookies["AuthToken"];
        
        // خواندن SOAP Body
        using var reader = new StreamReader(Request.Body);
        var soapRequest = await reader.ReadToEndAsync();
        
        // پردازش SOAP
        var soapResponse = ProcessSoap(soapRequest, authToken);
        
        // تنظیم Cookie جدید
        Response.Cookies.Append("AuthToken", "newtoken", new CookieOptions
        {
            HttpOnly = true,
            Path = "/"
        });
        
        return Content(soapResponse, "text/xml");
    }
}
```

---

## 1.5. WebSocket

### ویژگی‌ها
- Full-duplex communication
- Connection دائمی
- Real-time data
- Upgrade از HTTP

### ساختار Handshake
```http
GET /chat HTTP/1.1
Host: api.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Cookie: session_id=abc123
Authorization: Bearer token123
```

### Headers مهم در WebSocket
| Header | نقش | الزامی؟ | مثال |
|--------|-----|---------|------|
| **Upgrade** | نوع ارتقا | بله | `websocket` |
| **Connection** | نوع اتصال | بله | `Upgrade` |
| **Sec-WebSocket-Key** | کلید امنیتی | بله | `base64...` |
| **Sec-WebSocket-Version** | نسخه | بله | `13` |
| **Sec-WebSocket-Protocol** | پروتکل | خیر | `chat, superchat` |
| **Cookie** | Session | خیر | `session_id=xyz` |
| **Authorization** | Auth token | خیر | `Bearer token` |

### استفاده از Cookie در WebSocket
```csharp
// Startup.cs
app.UseWebSockets();

app.Use(async (context, next) =>
{
    if (context.Request.Path == "/ws")
    {
        if (context.WebSockets.IsWebSocketRequest)
        {
            // خواندن Cookie قبل از Upgrade
            var sessionId = context.Request.Cookies["session_id"];
            var token = context.Request.Headers["Authorization"];
            
            // اعتبارسنجی
            if (!ValidateSession(sessionId) && !ValidateToken(token))
            {
                context.Response.StatusCode = 401;
                return;
            }
            
            var webSocket = await context.WebSockets.AcceptWebSocketAsync();
            await HandleWebSocketConnection(webSocket, sessionId);
        }
        else
        {
            context.Response.StatusCode = 400;
        }
    }
    else
    {
        await next();
    }
});

async Task HandleWebSocketConnection(WebSocket webSocket, string sessionId)
{
    var buffer = new byte[1024 * 4];
    
    while (webSocket.State == WebSocketState.Open)
    {
        var result = await webSocket.ReceiveAsync(
            new ArraySegment<byte>(buffer), 
            CancellationToken.None);
        
        if (result.MessageType == WebSocketMessageType.Text)
        {
            var message = Encoding.UTF8.GetString(buffer, 0, result.Count);
            
            // پردازش پیام با استفاده از sessionId
            var response = ProcessMessage(message, sessionId);
            
            await webSocket.SendAsync(
                new ArraySegment<byte>(Encoding.UTF8.GetBytes(response)),
                WebSocketMessageType.Text,
                true,
                CancellationToken.None);
        }
        else if (result.MessageType == WebSocketMessageType.Close)
        {
            await webSocket.CloseAsync(
                WebSocketCloseStatus.NormalClosure,
                "Closing",
                CancellationToken.None);
        }
    }
}
```

---

## 1.6. Server-Sent Events (SSE)

### ویژگی‌ها
- یک‌طرفه: Server → Client
- بر پایه HTTP
- Auto-reconnect
- ساده‌تر از WebSocket

### ساختار Request
```http
GET /events HTTP/1.1
Host: api.example.com
Accept: text/event-stream
Cache-Control: no-cache
Cookie: session_id=abc123
Authorization: Bearer token123
```

### Headers مهم در SSE
| Header | نقش | الزامی؟ | مثال |
|--------|-----|---------|------|
| **Accept** | نوع داده | بله | `text/event-stream` |
| **Cache-Control** | غیرفعال کش | بله | `no-cache` |
| **Cookie** | Session | خیر | `session_id=xyz` |
| **Authorization** | Auth | خیر | `Bearer token` |
| **Last-Event-ID** | آخرین event | خیر | `123` |

### Response
```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

id: 1
event: message
data: {"text": "Hello"}

id: 2
event: notification
data: {"type": "info", "message": "New user joined"}

: this is a comment

id: 3
data: This is a message
data: with multiple lines
```

### مثال ASP.NET Core
```csharp
[Route("api/events")]
public class EventsController : ControllerBase
{
    [HttpGet]
    public async Task GetEvents()
    {
        // خواندن Cookie
        var sessionId = Request.Cookies["session_id"];
        var token = Request.Headers["Authorization"];
        
        // اعتبارسنجی
        if (!ValidateAuth(sessionId, token))
        {
            Response.StatusCode = 401;
            return;
        }
        
        // تنظیم Headers
        Response.ContentType = "text/event-stream";
        Response.Headers.Add("Cache-Control", "no-cache");
        Response.Headers.Add("Connection", "keep-alive");
        
        var lastEventId = Request.Headers["Last-Event-ID"].ToString();
        var startId = string.IsNullOrEmpty(lastEventId) ? 0 : int.Parse(lastEventId);
        
        try
        {
            for (int i = startId + 1; i <= 100; i++)
            {
                // ارسال event
                var message = $"id: {i}\nevent: message\ndata: {{\"id\": {i}, \"text\": \"Event {i}\"}}\n\n";
                await Response.WriteAsync(message);
                await Response.Body.FlushAsync();
                
                await Task.Delay(1000); // یک ثانیه تاخیر
                
                // بررسی قطع اتصال
                if (HttpContext.RequestAborted.IsCancellationRequested)
                    break;
            }
        }
        catch (Exception ex)
        {
            // Client disconnected
        }
    }
}
```

---

# بخش 2️⃣: HTTP Methods عمیق

## جدول کامل HTTP Methods

| Method | Body دارد؟ | Query دارد؟ | Idempotent | Safe | Cache | استفاده |
|--------|-----------|-------------|-----------|------|-------|---------|
| **GET** | ❌ | ✅ | ✅ | ✅ | ✅ | دریافت داده |
| **POST** | ✅ | ✅ | ❌ | ❌ | ❌* | ایجاد |
| **PUT** | ✅ | ✅ | ✅ | ❌ | ❌ | به‌روزرسانی کامل |
| **PATCH** | ✅ | ✅ | ❌ | ❌ | ❌ | به‌روزرسانی جزئی |
| **DELETE** | ⚠️ | ✅ | ✅ | ❌ | ❌ | حذف |
| **HEAD** | ❌ | ✅ | ✅ | ✅ | ✅ | فقط Headers |
| **OPTIONS** | ❌ | ✅ | ✅ | ✅ | ❌ | CORS/Methods |
| **TRACE** | ❌ | ✅ | ✅ | ✅ | ❌ | Debug |
| **CONNECT** | ❌ | ❌ | ❌ | ❌ | ❌ | Tunnel |

*POST می‌تواند با Cache-Control قابل کش شود

---

## توضیح تکمیلی هر Method

### GET - دریافت منابع
```http
# ساده
GET /api/users

# با Query Parameters
GET /api/users?page=1&size=10&sort=name

# با Route Parameters  
GET /api/users/123

# با Headers
GET /api/users
Authorization: Bearer token
Accept: application/json
Accept-Language: fa
If-None-Match: "etag123"
If-Modified-Since: Wed, 21 Oct 2015 07:28:00 GMT
```

**Headers معمول:**
- `Authorization`: احراز هویت
- `Accept`: نوع داده مورد انتظار
- `Accept-Language`: زبان
- `If-None-Match`: برای cache (با ETag)
- `If-Modified-Since`: برای cache
- `Cookie`: session/auth

**ASP.NET Core:**
```csharp
[HttpGet("users")]
public IActionResult GetUsers(
    [FromQuery] int page = 1,
    [FromQuery] int size = 10,
    [FromQuery] string sort = "name",
    [FromHeader(Name = "Accept-Language")] string language = "en",
    [FromHeader] string authorization = null)
{
    var cookie = Request.Cookies["session_id"];
    var ifNoneMatch = Request.Headers["If-None-Match"].ToString();
    
    // بررسی cache
    if (!string.IsNullOrEmpty(ifNoneMatch) && ifNoneMatch == CurrentETag)
    {
        return StatusCode(304); // Not Modified
    }
    
    var users = GetUsersFromDb(page, size, sort, language);
    
    Response.Headers.Add("ETag", GenerateETag(users));
    Response.Headers.Add("Cache-Control", "public, max-age=3600");
    
    return Ok(users);
}
```

---

### POST - ایجاد منابع
```http
POST /api/users HTTP/1.1
Content-Type: application/json
Authorization: Bearer token
X-Request-ID: uuid-123

{
  "name": "Ali",
  "email": "ali@test.com"
}
```

**Headers معمول:**
- `Content-Type`: نوع body (الزامی)
- `Authorization`: احراز هویت
- `X-Request-ID`: tracking
- `Idempotency-Key`: جلوگیری از ایجاد تکراری
- `Cookie`: session

**ASP.NET Core:**
```csharp
[HttpPost("users")]
public IActionResult CreateUser(
    [FromBody] CreateUserDto dto,
    [FromHeader(Name = "Idempotency-Key")] string idempotencyKey = null,
    [FromHeader(Name = "X-Request-ID")] string requestId = null)
{
    var cookie = Request.Cookies["session_id"];
    var contentType = Request.ContentType;
    
    // بررسی Idempotency
    if (!string.IsNullOrEmpty(idempotencyKey))
    {
        var existing = CheckIdempotencyKey(idempotencyKey);
        if (existing != null)
            return Ok(existing); // قبلا ایجاد شده
    }
    
    var user = CreateUserInDb(dto);
    
    // ذخیره Idempotency Key
    if (!string.IsNullOrEmpty(idempotencyKey))
        SaveIdempotencyKey(idempotencyKey, user);
    
    // تنظیم Cookie
    Response.Cookies.Append("last_created", user.Id.ToString());
    
    // تنظیم Headers
    Response.Headers.Add("X-Request-ID", requestId ?? Guid.NewGuid().ToString());
    Response.Headers.Add("Location", $"/api/users/{user.Id}");
    
    return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
}
```

---

### PUT - به‌روزرسانی کامل
```http
PUT /api/users/123 HTTP/1.1
Content-Type: application/json
If-Match: "etag123"

{
  "name": "Ali Updated",
  "email": "ali@updated.com",
  "age": 30
}
```

**Headers معمول:**
- `Content-Type`: الزامی
- `If-Match`: optimistic locking با ETag
- `If-Unmodified-Since`: optimistic locking با تاریخ

**ASP.NET Core:**
```csharp
[HttpPut("users/{id}")]
public IActionResult UpdateUser(
    int id,
    [FromBody] UpdateUserDto dto,
    [FromHeader(Name = "If-Match")] string ifMatch = null)
{
    var user = GetUserFromDb(id);
    
    if (user == null)
        return NotFound();
    
    // بررسی ETag
    if (!string.IsNullOrEmpty(ifMatch))
    {
        var currentETag = GenerateETag(user);
        if (ifMatch != currentETag)
            return StatusCode(412, "Precondition Failed"); // Resource changed
    }
    
    // به‌روزرسانی کامل
    user.Name = dto.Name;
    user.Email = dto.Email;
    user.Age = dto.Age;
    
    SaveToDb(user);
    
    // ETag جدید
    Response.Headers.Add("ETag", GenerateETag(user));
    
    return Ok(user);
}
```

---

### PATCH - به‌روزرسانی جزئی
```http
PATCH /api/users/123 HTTP/1.1
Content-Type: application/json-patch+json

[
  { "op": "replace", "path": "/name", "value": "New Name" },
  { "op": "add", "path": "/phone", "value": "123456" }
]

# یا ساده:
PATCH /api/users/123 HTTP/1.1
Content-Type: application/json

{
  "name": "New Name"
}
```

**ASP.NET Core:**
```csharp
[HttpPatch("users/{id}")]
public IActionResult PatchUser(
    int id,
    [FromBody] JsonPatchDocument<User> patchDoc)
{
    var user = GetUserFromDb(id);
    
    if (user == null)
        return NotFound();
    
    patchDoc.ApplyTo(user, ModelState);
    
    if (!ModelState.IsValid)
        return BadRequest(ModelState);
    
    SaveToDb(user);
    
    return Ok(user);
}

// یا ساده‌تر:
[HttpPatch("users/{id}")]
public IActionResult PatchUserSimple(
    int id,
    [FromBody] Dictionary<string, object> updates)
{
    var user = GetUserFromDb(id);
    
    if (user == null)
        return NotFound();
    
    // به‌روزرسانی فقط فیلدهای ارسال شده
    if (updates.ContainsKey("name"))
        user.Name = updates["name"].ToString();
    
    if (updates.ContainsKey("email"))
        user.Email = updates["email"].ToString();
    
    SaveToDb(user);
    
    return Ok(user);
}
```

---

### DELETE - حذف منابع
```http
DELETE /api/users/123 HTTP/1.1
Authorization: Bearer token
```

**ASP.NET Core:**
```csharp
[HttpDelete("users/{id}")]
public IActionResult DeleteUser(
    int id,
    [FromHeader] string authorization)
{
    var user = GetUserFromDb(id);
    
    if (user == null)
        return NotFound();
    
    DeleteFromDb(user);
    
    return NoContent(); // 204
}

// Soft Delete
[HttpDelete("users/{id}")]
public IActionResult SoftDeleteUser(int id)
{
    var user = GetUserFromDb(id);
    
    if (user == null)
        return NotFound();
    
    user.IsDeleted = true;
    user.DeletedAt = DateTime.UtcNow;
    
    SaveToDb(user);
    
    return NoContent();
}
```

---

### HEAD - دریافت فقط Headers
```http
HEAD /api/users/123 HTTP/1.1
```

**Response (بدون Body):**
```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 1234
ETag: "abc123"
Last-Modified: Wed, 21 Oct 2015 07:28:00 GMT
```

**ASP.NET Core:**
```csharp
// ASP.NET Core خودکار HEAD را پشتیبانی می‌کند
// فقط body ارسال نمی‌شود

[HttpGet("users/{id}")]
[HttpHead("users/{id}")] // اختیاری
public IActionResult GetUser(int id)
{
    var user = GetUserFromDb(id);
    
    if (user == null)
        return NotFound();
    
    // Headers تنظیم می‌شوند
    Response.Headers.Add("ETag", GenerateETag(user));
    Response.Headers.Add("Last-Modified", user.UpdatedAt.ToString("R"));
    
    return Ok(user); // برای HEAD، body ارسال نمی‌شود
}
```

---

### OPTIONS - دریافت Methods مجاز
```http
OPTIONS /api/users HTTP/1.1
Origin: https://myapp.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type
```

**Response:**
```http
HTTP/1.1 200 OK
Allow: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Origin: https://myapp.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
```

**ASP.NET Core:**
```csharp
// معمولا توسط CORS Middleware خودکار است
// Startup.cs
services.AddCors(options =>
{
    options.AddPolicy("AllowAll", builder =>
    {
        builder
            .WithOrigins("https://myapp.com")
            .WithMethods("GET", "POST", "PUT", "DELETE")
            .WithHeaders("Content-Type", "Authorization")
            .AllowCredentials() // برای ارسال Cookie
            .SetPreflightMaxAge(TimeSpan.FromDays(1));
    });
});

app.UseCors("AllowAll");
```

---

# بخش 3️⃣: Cookies در عمق

## انواع Cookies

### 1. Session Cookie (موقت)
```csharp
// بدون Expires یا MaxAge
Response.Cookies.Append("temp_data", "value");

// حذف می‌شود وقتی مرورگر بسته شود
```

### 2. Persistent Cookie (دائمی)
```csharp
Response.Cookies.Append("remember_me", "true", new CookieOptions
{
    Expires = DateTimeOffset.Now.AddMonths(1),
    // یا
    MaxAge = TimeSpan.FromDays(30)
});
```

### 3. Secure Cookie (فقط HTTPS)
```csharp
Response.Cookies.Append("auth_token", token, new CookieOptions
{
    Secure = true // فقط HTTPS
});
```

### 4. HttpOnly Cookie (غیرقابل دسترس از JS)
```csharp
Response.Cookies.Append("auth_token", token, new CookieOptions
{
    HttpOnly = true // جلوگیری از XSS
});
```

### 5. SameSite Cookie (محافظت CSRF)
```csharp
Response.Cookies.Append("auth_token", token, new CookieOptions
{
    SameSite = SameSiteMode.Strict,  // فقط same-site
    // یا
    SameSite = SameSiteMode.Lax,     // GET از سایت دیگر OK
    // یا
    SameSite = SameSiteMode.None     // همه جا (نیاز به Secure)
});
```

## Cookie در سناریوهای مختلف

### REST API با Cookie Authentication
```csharp
[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    [HttpPost("login")]
    public IActionResult Login([FromBody] LoginDto dto)
    {
        var user = Authenticate(dto);
        
        if (user == null)
            return Unauthorized();
        
        var token = GenerateJWT(user);
        
        // Cookie امن
        Response.Cookies.Append("access_token", token, new CookieOptions
        {
            HttpOnly = true,
            Secure = true,
            SameSite = SameSiteMode.Strict,
            Expires = DateTimeOffset.Now.AddHours(1)
        });
        
        // Refresh token
        var refreshToken = GenerateRefreshToken();
        Response.Cookies.Append("refresh_token", refreshToken, new CookieOptions
        {
            HttpOnly = true,
            Secure = true,
            SameSite = SameSiteMode.Strict,
            Expires = DateTimeOffset.Now.AddDays(7),
            Path = "/api/auth/refresh" // فقط در این endpoint
        });
        
        return Ok(new { message = "Logged in" });
    }
    
    [HttpPost("refresh")]
    public IActionResult Refresh()
    {
        var refreshToken = Request.Cookies["refresh_token"];
        
        if (string.IsNullOrEmpty(refreshToken))
            return Unauthorized();
        
        var newAccessToken = RefreshAccessToken(refreshToken);
        
        Response.Cookies.Append("access_token", newAccessToken, new CookieOptions
        {
            HttpOnly = true,
            Secure = true,
            SameSite = SameSiteMode.Strict,
            Expires = DateTimeOffset.Now.AddHours(1)
        });
        
        return Ok();
    }
    
    [HttpPost("logout")]
    public IActionResult Logout()
    {
        Response.Cookies.Delete("access_token");
        Response.Cookies.Delete("refresh_token");
        
        return Ok(new { message = "Logged out" });
    }
}

// Middleware برای خواندن Cookie
public class CookieAuthMiddleware
{
    private readonly RequestDelegate _next;
    
    public async Task InvokeAsync(HttpContext context)
    {
        var token = context.Request.Cookies["access_token"];
        
        if (!string.IsNullOrEmpty(token))
        {
            // اضافه کردن به Authorization Header
            context.Request.Headers["Authorization"] = $"Bearer {token}";
        }
        
        await _next(context);
    }
}

// Startup.cs
app.UseMiddleware<CookieAuthMiddleware>();
```

---

# بخش 4️⃣: Hybrid Approaches

## 1. REST + GraphQL

```csharp
// یک API که هم REST هم GraphQL دارد
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // REST
        services.AddControllers();
        
        // GraphQL
        services
            .AddGraphQLServer()
            .AddQueryType<Query>()
            .AddMutationType<Mutation>();
    }
    
    public void Configure(IApplicationBuilder app)
    {
        app.UseRouting();
        
        app.UseEndpoints(endpoints =>
        {
            // REST endpoints
            endpoints.MapControllers();
            
            // GraphQL endpoint
            endpoints.MapGraphQL("/graphql");
        });
    }
}

// استفاده
// REST: GET /api/users?page=1
// GraphQL: POST /graphql
```

**مزایا:**
- REST برای عملیات ساده
- GraphQL برای query پیچیده
- یک سیستم احراز هویت مشترک

**Cookie Management:**
```csharp
// هر دو از همان Cookie استفاده می‌کنند
public class SharedAuthService
{
    public void SetAuthCookie(HttpContext context, string token)
    {
        context.Response.Cookies.Append("auth_token", token, new CookieOptions
        {
            HttpOnly = true,
            Secure = true,
            Path = "/" // برای هر دو /api و /graphql
        });
    }
    
    public string GetAuthCookie(HttpContext context)
    {
        return context.Request.Cookies["auth_token"];
    }
}
```

---

## 2. REST + gRPC

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers(); // REST
    services.AddGrpc();        // gRPC
}

public void Configure(IApplicationBuilder app)
{
    app.UseRouting();
    
    app.UseEndpoints(endpoints =>
    {
        // REST
        endpoints.MapControllers();
        
        // gRPC
        endpoints.MapGrpcService<UserGrpcService>();
    });
}
```

**استفاده:**
- REST برای external clients (Web, Mobile)
- gRPC برای internal microservices (سریع‌تر)

---

## 3. REST + WebSocket

```csharp
public void Configure(IApplicationBuilder app)
{
    app.UseWebSockets();
    
    app.UseRouting();
    
    app.UseEndpoints(endpoints =>
    {
        // REST
        endpoints.MapControllers();
    });
    
    // WebSocket
    app.Use(async (context, next) =>
    {
        if (context.Request.Path == "/ws")
        {
            if (context.WebSockets.IsWebSocketRequest)
            {
                // خواندن cookie از REST login
                var token = context.Request.Cookies["auth_token"];
                
                if (ValidateToken(token))
                {
                    var ws = await context.WebSockets.AcceptWebSocketAsync();
                    await HandleWebSocket(ws, token);
                }
                else
                {
                    context.Response.StatusCode = 401;
                }
            }
        }
        else
        {
            await next();
        }
    });
}
```

---

## 4. REST + SSE

```csharp
[ApiController]
[Route("api")]
public class HybridController : ControllerBase
{
    // REST endpoint
    [HttpPost("messages")]
    public IActionResult SendMessage([FromBody] MessageDto dto)
    {
        var cookie = Request.Cookies["session_id"];
        BroadcastMessage(dto, cookie);
        return Ok();
    }
    
    // SSE endpoint
    [HttpGet("events")]
    public async Task GetEvents()
    {
        var cookie = Request.Cookies["session_id"];
        
        Response.ContentType = "text/event-stream";
        
        while (!HttpContext.RequestAborted.IsCancellationRequested)
        {
            var message = await GetNextMessage(cookie);
            await Response.WriteAsync($"data: {JsonSerializer.Serialize(message)}\n\n");
            await Response.Body.FlushAsync();
        }
    }
}
```

---

# بخش 5️⃣: Angular 21 و HttpClient

## مدرن Angular HttpClient

### تنظیمات اولیه
```typescript
// app.config.ts
import { ApplicationConfig, provideZoneChangeDetection } from '@angular/core';
import { provideHttpClient, withInterceptors, withFetch } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideZoneChangeDetection({ eventCoalescing: true }),
    provideHttpClient(
      withFetch(), // استفاده از Fetch API به جای XMLHttpRequest
      withInterceptors([authInterceptor, errorInterceptor])
    )
  ]
};
```

---

## 1. REST API در Angular 21

### Service با Signals
```typescript
// user.service.ts
import { Injectable, inject, signal } from '@angular/core';
import { HttpClient, HttpHeaders, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';

export interface User {
  id: number;
  name: string;
  email: string;
}

@Injectable({ providedIn: 'root' })
export class UserService {
  private http = inject(HttpClient);
  private baseUrl = 'https://api.example.com/api';
  
  // Signal برای state management
  users = signal<User[]>([]);
  loading = signal(false);
  
  // GET با Query Parameters
  getUsers(page: number = 1, size: number = 10): Observable<User[]> {
    // روش 1: با HttpParams
    const params = new HttpParams()
      .set('page', page)
      .set('size', size)
      .set('sort', 'name');
    
    return this.http.get<User[]>(`${this.baseUrl}/users`, { 
      params,
      headers: new HttpHeaders({
        'Accept': 'application/json',
        'X-Custom-Header': 'value'
      })
    });
    
    // روش 2: ساده‌تر (Angular 21+)
    return this.http.get<User[]>(`${this.baseUrl}/users`, {
      params: { page, size, sort: 'name' }
    });
  }
  
  // GET با Route Parameter
  getUser(id: number): Observable<User> {
    return this.http.get<User>(`${this.baseUrl}/users/${id}`);
  }
  
  // POST با Body
  createUser(user: Partial<User>): Observable<User> {
    return this.http.post<User>(`${this.baseUrl}/users`, user, {
      headers: new HttpHeaders({
        'Content-Type': 'application/json',
        'X-Request-ID': crypto.randomUUID()
      })
    });
  }
  
  // PUT
  updateUser(id: number, user: User): Observable<User> {
    return this.http.put<User>(`${this.baseUrl}/users/${id}`, user);
  }
  
  // PATCH
  patchUser(id: number, updates: Partial<User>): Observable<User> {
    return this.http.patch<User>(`${this.baseUrl}/users/${id}`, updates);
  }
  
  // DELETE
  deleteUser(id: number): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/users/${id}`);
  }
  
  // HEAD
  checkUserExists(id: number): Observable<any> {
    return this.http.head(`${this.baseUrl}/users/${id}`, {
      observe: 'response' // دریافت کامل response با headers
    });
  }
  
  // OPTIONS
  getUserOptions(): Observable<any> {
    return this.http.options(`${this.baseUrl}/users`, {
      observe: 'response'
    });
  }
}
```

---

### Component با Signals (Angular 21)
```typescript
// user.component.ts
import { Component, inject, signal, computed, effect } from '@angular/core';
import { UserService, User } from './user.service';

@Component({
  selector: 'app-user',
  standalone: true,
  template: `
    <div>
      @if (loading()) {
        <p>Loading...</p>
      }
      
      @if (error()) {
        <p class="error">{{ error() }}</p>
      }
      
      <ul>
        @for (user of users(); track user.id) {
          <li>{{ user.name }} - {{ user.email }}</li>
        }
      </ul>
      
      <p>Total: {{ totalUsers() }}</p>
    </div>
  `
})
export class UserComponent {
  private userService = inject(UserService);
  
  // Signals
  users = signal<User[]>([]);
  loading = signal(false);
  error = signal<string | null>(null);
  
  // Computed signal
  totalUsers = computed(() => this.users().length);
  
  constructor() {
    this.loadUsers();
    
    // Effect برای log کردن تغییرات
    effect(() => {
      console.log('Users changed:', this.users());
    });
  }
  
  loadUsers() {
    this.loading.set(true);
    this.error.set(null);
    
    this.userService.getUsers(1, 10).subscribe({
      next: (users) => {
        this.users.set(users);
        this.loading.set(false);
      },
      error: (err) => {
        this.error.set(err.message);
        this.loading.set(false);
      }
    });
  }
}
```

---

## 2. Cookie Management در Angular

### Interceptor برای خواندن Cookie
```typescript
// auth.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  // خواندن Cookie (اگر HttpOnly نباشد)
  const token = getCookie('auth_token');
  
  if (token) {
    // اضافه کردن به header
    const cloned = req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`
      }
    });
    return next(cloned);
  }
  
  return next(req);
};

function getCookie(name: string): string | null {
  const value = `; ${document.cookie}`;
  const parts = value.split(`; ${name}=`);
  
  if (parts.length === 2) {
    return parts.pop()?.split(';').shift() || null;
  }
  
  return null;
}
```

### ارسال Cookie با withCredentials
```typescript
// user.service.ts
@Injectable({ providedIn: 'root' })
export class UserService {
  private http = inject(HttpClient);
  
  login(email: string, password: string): Observable<any> {
    return this.http.post('https://api.example.com/api/auth/login', 
      { email, password },
      {
        withCredentials: true // مهم! برای ارسال و دریافت Cookie
      }
    );
  }
  
  getProfile(): Observable<User> {
    return this.http.get<User>('https://api.example.com/api/users/me', {
      withCredentials: true // Cookie خودکار ارسال می‌شود
    });
  }
  
  logout(): Observable<void> {
    return this.http.post<void>('https://api.example.com/api/auth/logout', {}, {
      withCredentials: true
    });
  }
}
```

### Global withCredentials
```typescript
// app.config.ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';

export const withCredentialsInterceptor: HttpInterceptorFn = (req, next) => {
  // اضافه کردن withCredentials به همه درخواست‌ها
  const cloned = req.clone({
    withCredentials: true
  });
  return next(cloned);
};

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([withCredentialsInterceptor])
    )
  ]
};
```

---

## 3. GraphQL در Angular 21

### نصب Apollo Client
```bash
npm install @apollo/client graphql
npm install @apollo/angular
```

### تنظیمات
```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient } from '@angular/common/http';
import { provideApollo } from 'apollo-angular';
import { HttpLink } from 'apollo-angular/http';
import { InMemoryCache, ApolloLink } from '@apollo/client/core';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(),
    provideApollo(() => {
      const httpLink = inject(HttpLink);
      
      // Link برای اضافه کردن Headers و Cookie
      const authLink = new ApolloLink((operation, forward) => {
        const token = getCookie('gql_token');
        
        operation.setContext({
          headers: {
            Authorization: token ? `Bearer ${token}` : '',
            'X-Request-ID': crypto.randomUUID()
          },
          // برای ارسال Cookie
          credentials: 'include'
        });
        
        return forward(operation);
      });
      
      return {
        link: authLink.concat(httpLink.create({ 
          uri: 'https://api.example.com/graphql' 
        })),
        cache: new InMemoryCache()
      };
    })
  ]
};

function getCookie(name: string): string | null {
  const value = `; ${document.cookie}`;
  const parts = value.split(`; ${name}=`);
  return parts.length === 2 ? parts.pop()?.split(';').shift() || null : null;
}
```

### Service با GraphQL
```typescript
// user-graphql.service.ts
import { Injectable, inject } from '@angular/core';
import { Apollo, gql } from 'apollo-angular';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

const GET_USER = gql`
  query GetUser($id: ID!) {
    user(id: $id) {
      id
      name
      email
    }
  }
`;

const GET_USERS = gql`
  query GetUsers($page: Int!, $size: Int!) {
    users(page: $page, size: $size) {
      items {
        id
        name
        email
      }
      total
      hasMore
    }
  }
`;

const CREATE_USER = gql`
  mutation CreateUser($input: CreateUserInput!) {
    createUser(input: $input) {
      id
      name
      email
    }
  }
`;

const LOGIN = gql`
  mutation Login($email: String!, $password: String!) {
    login(email: $email, password: $password) {
      token
      user {
        id
        name
        email
      }
    }
  }
`;

@Injectable({ providedIn: 'root' })
export class UserGraphQLService {
  private apollo = inject(Apollo);
  
  getUser(id: number): Observable<User> {
    return this.apollo
      .watchQuery<{ user: User }>({
        query: GET_USER,
        variables: { id },
        // Context برای Headers سفارشی
        context: {
          headers: {
            'X-Custom-Header': 'value'
          }
        }
      })
      .valueChanges
      .pipe(map(result => result.data.user));
  }
  
  getUsers(page: number, size: number): Observable<any> {
    return this.apollo
      .watchQuery({
        query: GET_USERS,
        variables: { page, size }
      })
      .valueChanges
      .pipe(map(result => result.data));
  }
  
  createUser(input: Partial<User>): Observable<User> {
    return this.apollo
      .mutate<{ createUser: User }>({
        mutation: CREATE_USER,
        variables: { input }
      })
      .pipe(map(result => result.data!.createUser));
  }
  
  login(email: string, password: string): Observable<any> {
    return this.apollo
      .mutate({
        mutation: LOGIN,
        variables: { email, password },
        // Cookie تنظیم می‌شود توسط سرور
      })
      .pipe(map(result => result.data));
  }
}
```

### Subscription (Real-time)
```typescript
const MESSAGE_SUBSCRIPTION = gql`
  subscription OnMessageAdded {
    messageAdded {
      id
      text
      user {
        name
      }
    }
  }
`;

@Injectable({ providedIn: 'root' })
export class MessageService {
  private apollo = inject(Apollo);
  
  onMessageAdded(): Observable<any> {
    return this.apollo
      .subscribe({
        query: MESSAGE_SUBSCRIPTION
      })
      .pipe(map(result => result.data));
  }
}
```

---

## 4. WebSocket در Angular

### Service
```typescript
// websocket.service.ts
import { Injectable, signal } from '@angular/core';
import { Observable, Subject } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class WebSocketService {
  private socket: WebSocket | null = null;
  private messageSubject = new Subject<any>();
  
  connected = signal(false);
  
  connect(url: string): void {
    // اضافه کردن token به URL یا Headers (محدودیت WebSocket)
    const token = this.getCookie('auth_token');
    const wsUrl = `${url}?token=${token}`;
    
    this.socket = new WebSocket(wsUrl);
    
    this.socket.onopen = () => {
      console.log('WebSocket connected');
      this.connected.set(true);
    };
    
    this.socket.onmessage = (event) => {
      const data = JSON.parse(event.data);
      this.messageSubject.next(data);
    };
    
    this.socket.onerror = (error) => {
      console.error('WebSocket error:', error);
    };
    
    this.socket.onclose = () => {
      console.log('WebSocket closed');
      this.connected.set(false);
      
      // Auto-reconnect
      setTimeout(() => this.connect(url), 5000);
    };
  }
  
  send(data: any): void {
    if (this.socket && this.socket.readyState === WebSocket.OPEN) {
      this.socket.send(JSON.stringify(data));
    }
  }
  
  onMessage(): Observable<any> {
    return this.messageSubject.asObservable();
  }
  
  disconnect(): void {
    if (this.socket) {
      this.socket.close();
      this.socket = null;
    }
  }
  
  private getCookie(name: string): string | null {
    const value = `; ${document.cookie}`;
    const parts = value.split(`; ${name}=`);
    return parts.length === 2 ? parts.pop()?.split(';').shift() || null : null;
  }
}
```

### Component
```typescript
@Component({
  selector: 'app-chat',
  template: `
    <div>
      <p>Connected: {{ ws.connected() }}</p>
      
      <div class="messages">
        @for (msg of messages(); track msg.id) {
          <p>{{ msg.text }}</p>
        }
      </div>
      
      <input #input (keyup.enter)="sendMessage(input.value); input.value = ''">
    </div>
  `
})
export class ChatComponent {
  ws = inject(WebSocketService);
  messages = signal<any[]>([]);
  
  constructor() {
    this.ws.connect('wss://api.example.com/ws');
    
    this.ws.onMessage().subscribe(msg => {
      this.messages.update(msgs => [...msgs, msg]);
    });
  }
  
  sendMessage(text: string) {
    this.ws.send({ type: 'message', text });
  }
  
  ngOnDestroy() {
    this.ws.disconnect();
  }
}
```

---

## 5. Server-Sent Events (SSE) در Angular

### Service
```typescript
// sse.service.ts
import { Injectable, signal } from '@angular/core';
import { Observable, Subject } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class SSEService {
  private eventSource: EventSource | null = null;
  private eventSubject = new Subject<any>();
  
  connected = signal(false);
  
  connect(url: string): void {
    // EventSource خودکار Cookie ارسال می‌کند
    this.eventSource = new EventSource(url, {
      withCredentials: true // برای ارسال Cookie
    });
    
    this.eventSource.onopen = () => {
      console.log('SSE connected');
      this.connected.set(true);
    };
    
    this.eventSource.onmessage = (event) => {
      const data = JSON.parse(event.data);
      this.eventSubject.next(data);
    };
    
    // Custom event
    this.eventSource.addEventListener('notification', (event: any) => {
      const data = JSON.parse(event.data);
      this.eventSubject.next({ type: 'notification', ...data });
    });
    
    this.eventSource.onerror = (error) => {
      console.error('SSE error:', error);
      this.connected.set(false);
    };
  }
  
  onEvent(): Observable<any> {
    return this.eventSubject.asObservable();
  }
  
  disconnect(): void {
    if (this.eventSource) {
      this.eventSource.close();
      this.eventSource = null;
      this.connected.set(false);
    }
  }
}
```

### Component
```typescript
@Component({
  selector: 'app-notifications',
  template: `
    <div>
      <p>Connected: {{ sse.connected() }}</p>
      
      <ul>
        @for (event of events(); track event.id) {
          <li>{{ event.text }}</li>
        }
      </ul>
    </div>
  `
})
export class NotificationsComponent {
  sse = inject(SSEService);
  events = signal<any[]>([]);
  
  constructor() {
    this.sse.connect('https://api.example.com/api/events');
    
    this.sse.onEvent().subscribe(event => {
      this.events.update(events => [...events, event]);
    });
  }
  
  ngOnDestroy() {
    this.sse.disconnect();
  }
}
```

---

## 6. File Upload در Angular

### Service
```typescript
// upload.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpEvent, HttpEventType } from '@angular/common/http';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class UploadService {
  private http = inject(HttpClient);
  private baseUrl = 'https://api.example.com/api';
  
  uploadFile(file: File, description: string): Observable<number | any> {
    const formData = new FormData();
    formData.append('file', file);
    formData.append('description', description);
    
    return this.http.post(`${this.baseUrl}/files/upload`, formData, {
      reportProgress: true, // برای Progress Bar
      observe: 'events',    // دریافت همه events
      withCredentials: true, // ارسال Cookie
      params: {
        folder: 'documents'
      },
      headers: {
        // Content-Type رو خودکار تشخیص میده برای FormData
        'X-Request-ID': crypto.randomUUID()
      }
    }).pipe(
      map(event => {
        if (event.type === HttpEventType.UploadProgress) {
          // محاسبه درصد
          const progress = event.total 
            ? Math.round((100 * event.loaded) / event.total)
            : 0;
          return progress;
        } else if (event.type === HttpEventType.Response) {
          // آپلود کامل
          return event.body;
        }
        return null;
      })
    );
  }
  
  uploadMultiple(files: File[]): Observable<any> {
    const formData = new FormData();
    
    files.forEach((file, index) => {
      formData.append(`files`, file);
    });
    
    return this.http.post(`${this.baseUrl}/files/upload-multiple`, formData, {
      reportProgress: true,
      observe: 'events',
      withCredentials: true
    });
  }
}
```

### Component
```typescript
@Component({
  selector: 'app-upload',
  template: `
    <div>
      <input type="file" (change)="onFileSelected($event)">
      <input type="text" [(ngModel)]="description" placeholder="Description">
      <button (click)="upload()" [disabled]="!selectedFile">Upload</button>
      
      @if (uploading()) {
        <div class="progress">
          <div class="bar" [style.width.%]="progress()"></div>
        </div>
        <p>{{ progress() }}%</p>
      }
      
      @if (result()) {
        <p>Uploaded: {{ result().fileName }}</p>
      }
    </div>
  `
})
export class UploadComponent {
  private uploadService = inject(UploadService);
  
  selectedFile: File | null = null;
  description = '';
  uploading = signal(false);
  progress = signal(0);
  result = signal<any>(null);
  
  onFileSelected(event: any) {
    this.selectedFile = event.target.files[0];
  }
  
  upload() {
    if (!this.selectedFile) return;
    
    this.uploading.set(true);
    this.progress.set(0);
    
    this.uploadService.uploadFile(this.selectedFile, this.description)
      .subscribe({
        next: (data) => {
          if (typeof data === 'number') {
            // Progress
            this.progress.set(data);
          } else if (data) {
            // Complete
            this.result.set(data);
            this.uploading.set(false);
          }
        },
        error: (err) => {
          console.error('Upload failed:', err);
          this.uploading.set(false);
        }
      });
  }
}
```

---

## 7. Error Handling در Angular 21

### Global Error Interceptor
```typescript
// error.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { inject } from '@angular/core';
import { Router } from '@angular/router';
import { catchError, throwError } from 'rxjs';

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);
  
  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      let errorMessage = 'An error occurred';
      
      if (error.error instanceof ErrorEvent) {
        // Client-side error
        errorMessage = `Error: ${error.error.message}`;
      } else {
        // Server-side error
        switch (error.status) {
          case 0:
            errorMessage = 'No connection to server';
            break;
          case 400:
            errorMessage = error.error?.message || 'Bad Request';
            break;
          case 401:
            errorMessage = 'Unauthorized';
            // حذف Cookie و redirect
            document.cookie = 'auth_token=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/;';
            router.navigate(['/login']);
            break;
          case 403:
            errorMessage = 'Forbidden';
            break;
          case 404:
            errorMessage = 'Not Found';
            break;
          case 500:
            errorMessage = 'Internal Server Error';
            break;
          default:
            errorMessage = `Error Code: ${error.status}\nMessage: ${error.message}`;
        }
      }
      
      // Log to console
      console.error(errorMessage);
      
      // Show to user (میتونی از toast service استفاده کنی)
      alert(errorMessage);
      
      return throwError(() => new Error(errorMessage));
    })
  );
};
```

---

## 8. Request Caching در Angular

### Cache Interceptor
```typescript
// cache.interceptor.ts
import { HttpInterceptorFn, HttpResponse } from '@angular/common/http';
import { of, tap } from 'rxjs';

const cache = new Map<string, HttpResponse<any>>();

export const cacheInterceptor: HttpInterceptorFn = (req, next) => {
  // فقط GET را کش کن
  if (req.method !== 'GET') {
    return next(req);
  }
  
  // بررسی کش
  const cachedResponse = cache.get(req.url);
  if (cachedResponse) {
    console.log('Returning cached response for:', req.url);
    return of(cachedResponse.clone());
  }
  
  // اگر توی کش نیست، درخواست بزن و کش کن
  return next(req).pipe(
    tap(event => {
      if (event instanceof HttpResponse) {
        cache.set(req.url, event.clone());
        
        // حذف بعد از 5 دقیقه
        setTimeout(() => {
          cache.delete(req.url);
        }, 5 * 60 * 1000);
      }
    })
  );
};
```

---

## 9. Retry Logic در Angular

### Retry Interceptor
```typescript
// retry.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { retry, timer } from 'rxjs';

export const retryInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    retry({
      count: 3, // تلاش 3 بار
      delay: (error, retryCount) => {
        // Exponential backoff: 1s, 2s, 4s
        const delayMs = Math.pow(2, retryCount - 1) * 1000;
        console.log(`Retrying ${retryCount}/3 after ${delayMs}ms`);
        return timer(delayMs);
      }
    })
  );
};
```

---

## 10. Request Timeout در Angular

### Timeout Interceptor
```typescript
// timeout.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { timeout, catchError, throwError } from 'rxjs';

export const timeoutInterceptor: HttpInterceptorFn = (req, next) => {
  const timeoutDuration = 30000; // 30 ثانیه
  
  return next(req).pipe(
    timeout(timeoutDuration),
    catchError(error => {
      if (error.name === 'TimeoutError') {
        return throwError(() => new Error('Request timeout'));
      }
      return throwError(() => error);
    })
  );
};
```

---

# بخش 6️⃣: جدول مقایسه کامل

## مقایسه معماری‌ها

| ویژگی | REST | GraphQL | gRPC | SOAP | WebSocket | SSE |
|-------|------|---------|------|------|-----------|-----|
| **Protocol** | HTTP/1.1 | HTTP/1.1+ | HTTP/2 | HTTP/1.1 | WS Protocol | HTTP/1.1 |
| **Format** | JSON/XML | JSON | Protobuf | XML | Binary/Text | Text |
| **Request Methods** | GET/POST/PUT/DELETE | POST | POST | POST | Bidirectional | GET |
| **Body** | بله (غیر GET) | همیشه | همیشه | همیشه | Messages | Server→Client |
| **Cookie Support** | ✅ کامل | ✅ کامل | ⚠️ محدود | ✅ کامل | ⚠️ Initial only | ✅ کامل |
| **Headers** | ✅ کامل | ✅ کامل | ⚠️ Metadata | ✅ کامل | ⚠️ Initial only | ✅ کامل |
| **Real-time** | ❌ | ⚠️ Subscription | ✅ Stream | ❌ | ✅ | ✅ |
| **Caching** | ✅ | ⚠️ | ❌ | ❌ | ❌ | ❌ |
| **Learning Curve** | آسان | متوسط | سخت | سخت | متوسط | آسان |
| **Performance** | خوب | خوب | عالی | ضعیف | عالی | خوب |
| **Browser Support** | ✅ | ✅ | ⚠️ | ✅ | ✅ | ✅ |
| **Mobile Support** | ✅ | ✅ | ✅ | ✅ | ✅ | ⚠️ |

---

## مقایسه Authentication Methods

| روش | استفاده از Cookie | Header | Best For | Security |
|-----|-------------------|--------|----------|----------|
| **Cookie-based** | ✅ | ❌ | Web Apps | متوسط (با HttpOnly بالا) |
| **Bearer Token** | ❌ | Authorization | Mobile/SPA | بالا |
| **JWT in Cookie** | ✅ | ❌ | Web Apps | خیلی بالا |
| **API Key** | ❌ | X-API-Key | Server-to-Server | متوسط |
| **OAuth 2.0** | ✅/❌ | Authorization | Third-party | بالا |
| **Basic Auth** | ❌ | Authorization | Simple APIs | ضعیف |

---

## مقایسه Headers در هر نوع API

### REST
```typescript
{
  'Content-Type': 'application/json',
  'Accept': 'application/json',
  'Authorization': 'Bearer token',
  'Cookie': 'session=abc',
  'Cache-Control': 'no-cache',
  'ETag': '"abc123"',
  'If-None-Match': '"abc123"',
  'X-Request-ID': 'uuid'
}
```

### GraphQL
```typescript
{
  'Content-Type': 'application/json',  // همیشه
  'Authorization': 'Bearer token',
  'Cookie': 'gql_token=abc',
  'X-Apollo-Tracing': '1',
  'X-Request-ID': 'uuid'
}
```

### gRPC
```typescript
{
  'Content-Type': 'application/grpc',  // همیشه
  'TE': 'trailers',                    // همیشه
  'Grpc-Timeout': '1000m',
  'Grpc-Metadata-Authorization': 'Bearer token',
  'Grpc-Metadata-User-Id': '123'
  // Cookie: محدود، باید از Metadata استفاده کنی
}
```

### WebSocket
```typescript
// Initial handshake only
{
  'Upgrade': 'websocket',
  'Connection': 'Upgrade',
  'Sec-WebSocket-Key': 'base64key',
  'Sec-WebSocket-Version': '13',
  'Cookie': 'session=abc',
  'Authorization': 'Bearer token'
}
// بعد از handshake، فقط messages
```

### SSE
```typescript
{
  'Accept': 'text/event-stream',
  'Cache-Control': 'no-cache',
  'Cookie': 'session=abc',
  'Authorization': 'Bearer token',
  'Last-Event-ID': '123'
}
```

---

# بخش 7️⃣: Best Practices

## ✅ Security Best Practices

### 1. Cookie Security
```csharp
// ✅ انجام دهید
Response.Cookies.Append("auth_token", token, new CookieOptions
{
    HttpOnly = true,      // جلوگیری از XSS
    Secure = true,        // فقط HTTPS
    SameSite = SameSiteMode.Strict, // جلوگیری از CSRF
    MaxAge = TimeSpan.FromHours(1),
    Path = "/api"         // محدود کردن scope
});

// ❌ انجام ندهید
Response.Cookies.Append("auth_token", token); // ناامن!
```

### 2. CORS Configuration
```csharp
// ✅ تنظیمات امن
services.AddCors(options =>
{
    options.AddPolicy("Secure", builder =>
    {
        builder
            .WithOrigins("https://myapp.com") // مشخص
            .WithMethods("GET", "POST")        // محدود
            .WithHeaders("Content-Type", "Authorization")
            .AllowCredentials()                // برای Cookie
            .SetPreflightMaxAge(TimeSpan.FromHours(1));
    });
});

// ❌ ناامن
builder.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader();
```

### 3. Input Validation
```csharp
// ✅ با Data Annotations
public class CreateUserDto
{
    [Required]
    [EmailAddress]
    public string Email { get; set; }
    
    [Required]
    [MinLength(3)]
    [MaxLength(100)]
    public string Name { get; set; }
    
    [Range(18, 120)]
    public int Age { get; set; }
}

[HttpPost("users")]
public IActionResult CreateUser([FromBody] CreateUserDto dto)
{
    if (!ModelState.IsValid)
        return BadRequest(ModelState);
    
    // ...
}
```

### 4. Rate Limiting
```csharp
// با AspNetCoreRateLimit
services.AddMemoryCache();
services.Configure<IpRateLimitOptions>(Configuration.GetSection("IpRateLimiting"));
services.AddSingleton<IRateLimitCounterStore, MemoryCacheRateLimitCounterStore>();
services.AddSingleton<IIpPolicyStore, MemoryCacheIpPolicyStore>();
services.AddSingleton<IRateLimitConfiguration, RateLimitConfiguration>();

// appsettings.json
{
  "IpRateLimiting": {
    "EnableEndpointRateLimiting": true,
    "StackBlockedRequests": false,
    "GeneralRules": [
      {
        "Endpoint": "*",
        "Period": "1m",
        "Limit": 100
      }
    ]
  }
}
```

---

## ✅ Performance Best Practices

### 1. Response Caching
```csharp
[HttpGet("users/{id}")]
[ResponseCache(Duration = 60, VaryByQueryKeys = new[] { "id" })]
public IActionResult GetUser(int id)
{
    // ...
}
```

### 2. Compression
```csharp
services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<GzipCompressionProvider>();
    options.Providers.Add<BrotliCompressionProvider>();
});
```

### 3. Async/Await
```csharp
// ✅ درست
[HttpGet("users")]
public async Task<IActionResult> GetUsers()
{
    var users = await _context.Users.ToListAsync();
    return Ok(users);
}

// ❌ اشتباه (blocking)
[HttpGet("users")]
public IActionResult GetUsers()
{
    var users = _context.Users.ToList(); // Synchronous!
    return Ok(users);
}
```

---

## خلاصه نهایی

### چه زمانی از چه استفاده کنیم؟

| سناریو | بهترین انتخاب | Cookie؟ | چرا؟ |
|--------|---------------|---------|------|
| **CRUD ساده** | REST | ✅ | ساده و استاندارد |
| **Query پیچیده** | GraphQL | ✅ | دریافت دقیق داده |
| **Microservices** | gRPC | ⚠️ | سرعت بالا |
| **Real-time Chat** | WebSocket | ⚠️ | دو طرفه |
| **Live Updates** | SSE | ✅ | یک طرفه، ساده |
| **Legacy System** | SOAP | ✅ | سازگاری |
| **Mobile App** | REST + JWT | ❌ | بدون Cookie |
| **Web App** | REST + Cookie | ✅ | امن‌تر |
| **Hybrid** | REST + GraphQL | ✅ | بهترین هر دو |

---

---

# بخش 8️⃣: CORS (Cross-Origin Resource Sharing) عمیق

## CORS چیست؟
وقتی فرانت شما روی `https://myapp.com` هست و API روی `https://api.example.com`، مرورگر به صورت پیش‌فرض درخواست را بلاک می‌کند. CORS این مشکل را حل می‌کند.

## انواع CORS Requests

### 1. Simple Request
```http
GET /api/users HTTP/1.1
Host: api.example.com
Origin: https://myapp.com
Accept: application/json
```

**شرایط Simple Request:**
- Methods: GET، POST، HEAD
- Headers: فقط Simple Headers (Accept, Accept-Language, Content-Language, Content-Type)
- Content-Type: application/x-www-form-urlencoded، multipart/form-data، text/plain

**Response:**
```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://myapp.com
Access-Control-Allow-Credentials: true
Content-Type: application/json
```

### 2. Preflight Request (پیچیده‌تر)
```http
OPTIONS /api/users HTTP/1.1
Host: api.example.com
Origin: https://myapp.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type, Authorization
```

**Response به Preflight:**
```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://myapp.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
Access-Control-Allow-Credentials: true
```

**سپس Actual Request:**
```http
POST /api/users HTTP/1.1
Host: api.example.com
Origin: https://myapp.com
Content-Type: application/json
Authorization: Bearer token
```

---

## CORS در ASP.NET Core

### پیکربندی کامل
```csharp
// Startup.cs یا Program.cs
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // روش 1: با نام Policy
        services.AddCors(options =>
        {
            // Policy سخت‌گیرانه (Production)
            options.AddPolicy("Production", builder =>
            {
                builder
                    .WithOrigins("https://myapp.com", "https://app.myapp.com")
                    .WithMethods("GET", "POST", "PUT", "DELETE")
                    .WithHeaders("Content-Type", "Authorization", "X-Request-ID")
                    .AllowCredentials() // برای Cookie
                    .SetIsOriginAllowedToAllowWildcardSubdomains()
                    .SetPreflightMaxAge(TimeSpan.FromHours(1));
            });
            
            // Policy آزاد (Development)
            options.AddPolicy("Development", builder =>
            {
                builder
                    .AllowAnyOrigin()
                    .AllowAnyMethod()
                    .AllowAnyHeader();
                    // توجه: با AllowAnyOrigin نمی‌تونی AllowCredentials بزاری!
            });
            
            // Policy با Wildcard
            options.AddPolicy("Wildcard", builder =>
            {
                builder
                    .WithOrigins("https://*.myapp.com")
                    .SetIsOriginAllowedToAllowWildcardSubdomains()
                    .AllowAnyMethod()
                    .AllowAnyHeader()
                    .AllowCredentials();
            });
            
            // Policy پویا
            options.AddPolicy("Dynamic", builder =>
            {
                builder
                    .SetIsOriginAllowed(origin => 
                    {
                        // بررسی پویای Origin
                        var allowedOrigins = new[] 
                        { 
                            "https://myapp.com",
                            "https://localhost:4200"
                        };
                        return allowedOrigins.Contains(origin) || 
                               origin.EndsWith(".myapp.com");
                    })
                    .AllowAnyMethod()
                    .AllowAnyHeader()
                    .AllowCredentials();
            });
        });
        
        services.AddControllers();
    }
    
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseCors("Development");
        }
        else
        {
            app.UseCors("Production");
        }
        
        app.UseRouting();
        app.UseAuthentication();
        app.UseAuthorization();
        
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
    }
}
```

### استفاده در Controller
```csharp
// برای کل Controller
[EnableCors("Production")]
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll()
    {
        return Ok(users);
    }
    
    // غیرفعال کردن CORS برای یک Action خاص
    [DisableCors]
    [HttpGet("internal")]
    public IActionResult GetInternal()
    {
        return Ok(internalData);
    }
    
    // Policy متفاوت برای یک Action
    [EnableCors("Development")]
    [HttpPost("test")]
    public IActionResult Test()
    {
        return Ok();
    }
}
```

### CORS Middleware سفارشی
```csharp
public class CustomCorsMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<CustomCorsMiddleware> _logger;
    
    public CustomCorsMiddleware(
        RequestDelegate next,
        ILogger<CustomCorsMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var origin = context.Request.Headers["Origin"].ToString();
        
        _logger.LogInformation($"CORS request from: {origin}");
        
        // بررسی سفارشی
        if (IsAllowedOrigin(origin))
        {
            context.Response.Headers.Add("Access-Control-Allow-Origin", origin);
            context.Response.Headers.Add("Access-Control-Allow-Credentials", "true");
            context.Response.Headers.Add("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS");
            context.Response.Headers.Add("Access-Control-Allow-Headers", "Content-Type, Authorization");
            context.Response.Headers.Add("Access-Control-Max-Age", "86400");
            
            // پاسخ به Preflight
            if (context.Request.Method == "OPTIONS")
            {
                context.Response.StatusCode = 204;
                return;
            }
        }
        else
        {
            _logger.LogWarning($"Blocked CORS request from: {origin}");
            context.Response.StatusCode = 403;
            await context.Response.WriteAsync("Origin not allowed");
            return;
        }
        
        await _next(context);
    }
    
    private bool IsAllowedOrigin(string origin)
    {
        var allowedOrigins = new[]
        {
            "https://myapp.com",
            "https://localhost:4200"
        };
        
        return allowedOrigins.Contains(origin) || 
               origin?.EndsWith(".myapp.com") == true;
    }
}

// Startup.cs
app.UseMiddleware<CustomCorsMiddleware>();
```

---

## CORS در Angular 21

### HttpClient با CORS
```typescript
// تنظیمات برای CORS
@Injectable({ providedIn: 'root' })
export class ApiService {
  private http = inject(HttpClient);
  
  // با withCredentials برای ارسال Cookie
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>('https://api.example.com/api/users', {
      withCredentials: true, // مهم! برای CORS با Cookie
      headers: {
        'Content-Type': 'application/json'
      }
    });
  }
  
  // بدون Credential
  getPublicData(): Observable<any> {
    return this.http.get('https://api.example.com/api/public');
  }
}
```

### Global withCredentials
```typescript
// app.config.ts
export const corsInterceptor: HttpInterceptorFn = (req, next) => {
  // اضافه کردن withCredentials به همه درخواست‌های cross-origin
  const apiUrl = 'https://api.example.com';
  
  if (req.url.startsWith(apiUrl)) {
    const cloned = req.clone({
      withCredentials: true
    });
    return next(cloned);
  }
  
  return next(req);
};

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([corsInterceptor])
    )
  ]
};
```

### Handle CORS Errors
```typescript
export const corsErrorInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 0) {
        // معمولا خطای CORS
        console.error('CORS Error or Network Error:', error);
        
        // بررسی بیشتر
        if (error.error instanceof ProgressEvent) {
          console.error('Possible CORS issue - check server CORS configuration');
        }
      }
      
      return throwError(() => error);
    })
  );
};
```

---

# بخش 9️⃣: Webhooks

## Webhook چیست؟
Webhook یک HTTP callback است. به جای اینکه client مدام polling کنه، server وقتی event اتفاق افتاد خودش به client اطلاع می‌ده.

## معماری Webhook
```
Event در Server → HTTP POST → Client URL (Webhook Endpoint)
```

---

## پیاده‌سازی Webhook Server در ASP.NET Core

### 1. مدل‌ها
```csharp
// Models/Webhook.cs
public class Webhook
{
    public int Id { get; set; }
    public string Url { get; set; }
    public string Secret { get; set; }
    public List<string> Events { get; set; } // ["user.created", "order.completed"]
    public bool IsActive { get; set; }
    public DateTime CreatedAt { get; set; }
}

public class WebhookPayload
{
    public string EventType { get; set; }
    public DateTime Timestamp { get; set; }
    public object Data { get; set; }
    public string Signature { get; set; } // HMAC signature برای امنیت
}

public class WebhookLog
{
    public int Id { get; set; }
    public int WebhookId { get; set; }
    public string EventType { get; set; }
    public string Url { get; set; }
    public int StatusCode { get; set; }
    public string Response { get; set; }
    public DateTime SentAt { get; set; }
    public int RetryCount { get; set; }
}
```

### 2. Service برای ارسال Webhook
```csharp
// Services/WebhookService.cs
public interface IWebhookService
{
    Task SendWebhookAsync(string eventType, object data);
    Task<Webhook> RegisterWebhookAsync(string url, List<string> events, string secret);
    Task<bool> UnregisterWebhookAsync(int webhookId);
}

public class WebhookService : IWebhookService
{
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly ApplicationDbContext _context;
    private readonly ILogger<WebhookService> _logger;
    
    public WebhookService(
        IHttpClientFactory httpClientFactory,
        ApplicationDbContext context,
        ILogger<WebhookService> logger)
    {
        _httpClientFactory = httpClientFactory;
        _context = context;
        _logger = logger;
    }
    
    public async Task SendWebhookAsync(string eventType, object data)
    {
        // پیدا کردن webhookهایی که برای این event ثبت شده‌اند
        var webhooks = await _context.Webhooks
            .Where(w => w.IsActive && w.Events.Contains(eventType))
            .ToListAsync();
        
        foreach (var webhook in webhooks)
        {
            _ = Task.Run(async () => await SendToWebhookAsync(webhook, eventType, data));
        }
    }
    
    private async Task SendToWebhookAsync(Webhook webhook, string eventType, object data)
    {
        var payload = new WebhookPayload
        {
            EventType = eventType,
            Timestamp = DateTime.UtcNow,
            Data = data
        };
        
        // محاسبه Signature برای امنیت
        var json = JsonSerializer.Serialize(payload);
        payload.Signature = GenerateSignature(json, webhook.Secret);
        
        var client = _httpClientFactory.CreateClient();
        var content = new StringContent(
            JsonSerializer.Serialize(payload),
            Encoding.UTF8,
            "application/json");
        
        int retryCount = 0;
        int maxRetries = 3;
        bool success = false;
        
        while (!success && retryCount < maxRetries)
        {
            try
            {
                var response = await client.PostAsync(webhook.Url, content);
                
                // Log
                await LogWebhookAsync(webhook.Id, eventType, webhook.Url, 
                    (int)response.StatusCode, await response.Content.ReadAsStringAsync(), retryCount);
                
                if (response.IsSuccessStatusCode)
                {
                    success = true;
                    _logger.LogInformation($"Webhook sent successfully to {webhook.Url}");
                }
                else
                {
                    _logger.LogWarning($"Webhook failed: {response.StatusCode}");
                    retryCount++;
                    await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, retryCount))); // Exponential backoff
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, $"Error sending webhook to {webhook.Url}");
                retryCount++;
                await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, retryCount)));
            }
        }
        
        if (!success)
        {
            _logger.LogError($"Webhook failed after {maxRetries} retries: {webhook.Url}");
            // می‌تونی webhook رو غیرفعال کنی یا به admin اطلاع بدی
        }
    }
    
    private string GenerateSignature(string payload, string secret)
    {
        using var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(secret));
        var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(payload));
        return Convert.ToBase64String(hash);
    }
    
    private async Task LogWebhookAsync(int webhookId, string eventType, string url, 
        int statusCode, string response, int retryCount)
    {
        var log = new WebhookLog
        {
            WebhookId = webhookId,
            EventType = eventType,
            Url = url,
            StatusCode = statusCode,
            Response = response,
            SentAt = DateTime.UtcNow,
            RetryCount = retryCount
        };
        
        _context.WebhookLogs.Add(log);
        await _context.SaveChangesAsync();
    }
    
    public async Task<Webhook> RegisterWebhookAsync(string url, List<string> events, string secret)
    {
        var webhook = new Webhook
        {
            Url = url,
            Events = events,
            Secret = secret ?? Guid.NewGuid().ToString(),
            IsActive = true,
            CreatedAt = DateTime.UtcNow
        };
        
        _context.Webhooks.Add(webhook);
        await _context.SaveChangesAsync();
        
        return webhook;
    }
    
    public async Task<bool> UnregisterWebhookAsync(int webhookId)
    {
        var webhook = await _context.Webhooks.FindAsync(webhookId);
        if (webhook == null) return false;
        
        _context.Webhooks.Remove(webhook);
        await _context.SaveChangesAsync();
        
        return true;
    }
}
```

### 3. Controller برای مدیریت Webhooks
```csharp
[ApiController]
[Route("api/[controller]")]
public class WebhooksController : ControllerBase
{
    private readonly IWebhookService _webhookService;
    
    public WebhooksController(IWebhookService webhookService)
    {
        _webhookService = webhookService;
    }
    
    // ثبت webhook جدید
    [HttpPost("register")]
    public async Task<IActionResult> Register([FromBody] RegisterWebhookDto dto)
    {
        var webhook = await _webhookService.RegisterWebhookAsync(
            dto.Url, 
            dto.Events, 
            dto.Secret);
        
        return Ok(new 
        { 
            id = webhook.Id,
            url = webhook.Url,
            secret = webhook.Secret,
            events = webhook.Events
        });
    }
    
    // حذف webhook
    [HttpDelete("{id}")]
    public async Task<IActionResult> Unregister(int id)
    {
        var result = await _webhookService.UnregisterWebhookAsync(id);
        
        if (!result)
            return NotFound();
        
        return NoContent();
    }
    
    // تست webhook
    [HttpPost("{id}/test")]
    public async Task<IActionResult> Test(int id)
    {
        await _webhookService.SendWebhookAsync("test.event", new { message = "Test webhook" });
        return Ok(new { message = "Test webhook sent" });
    }
}
```

### 4. استفاده در Application
```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IWebhookService _webhookService;
    
    [HttpPost]
    public async Task<IActionResult> CreateUser([FromBody] CreateUserDto dto)
    {
        var user = await CreateUserInDb(dto);
        
        // ارسال webhook
        await _webhookService.SendWebhookAsync("user.created", new
        {
            userId = user.Id,
            name = user.Name,
            email = user.Email,
            createdAt = user.CreatedAt
        });
        
        return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
    }
}
```

---

## دریافت Webhook در ASP.NET Core (Client)

```csharp
[ApiController]
[Route("webhooks")]
public class WebhookReceiverController : ControllerBase
{
    private readonly ILogger<WebhookReceiverController> _logger;
    private const string SECRET = "your-webhook-secret";
    
    [HttpPost("payment")]
    public async Task<IActionResult> ReceivePaymentWebhook([FromBody] WebhookPayload payload)
    {
        // بررسی Signature
        var signature = Request.Headers["X-Webhook-Signature"].ToString();
        
        if (!VerifySignature(payload, signature))
        {
            _logger.LogWarning("Invalid webhook signature");
            return Unauthorized();
        }
        
        // پردازش event
        switch (payload.EventType)
        {
            case "payment.success":
                await HandlePaymentSuccess(payload.Data);
                break;
            case "payment.failed":
                await HandlePaymentFailed(payload.Data);
                break;
            default:
                _logger.LogWarning($"Unknown event type: {payload.EventType}");
                break;
        }
        
        return Ok();
    }
    
    private bool VerifySignature(WebhookPayload payload, string signature)
    {
        var json = JsonSerializer.Serialize(payload);
        using var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(SECRET));
        var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(json));
        var expectedSignature = Convert.ToBase64String(hash);
        
        return signature == expectedSignature;
    }
    
    private async Task HandlePaymentSuccess(object data)
    {
        _logger.LogInformation($"Payment successful: {JsonSerializer.Serialize(data)}");
        // پردازش پرداخت موفق
    }
    
    private async Task HandlePaymentFailed(object data)
    {
        _logger.LogWarning($"Payment failed: {JsonSerializer.Serialize(data)}");
        // پردازش پرداخت ناموفق
    }
}
```

---

## Webhook در Angular (Client)

### ثبت Webhook
```typescript
// webhook.service.ts
@Injectable({ providedIn: 'root' })
export class WebhookService {
  private http = inject(HttpClient);
  private baseUrl = 'https://api.example.com/api';
  
  registerWebhook(url: string, events: string[]): Observable<any> {
    return this.http.post(`${this.baseUrl}/webhooks/register`, {
      url,
      events,
      secret: this.generateSecret()
    });
  }
  
  unregisterWebhook(id: number): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/webhooks/${id}`);
  }
  
  testWebhook(id: number): Observable<any> {
    return this.http.post(`${this.baseUrl}/webhooks/${id}/test`, {});
  }
  
  private generateSecret(): string {
    return crypto.randomUUID();
  }
}
```

---

# بخش 🔟: HttpContext و کلاس‌های مهم ASP.NET Core

## HttpContext - قلب ASP.NET Core

### ساختار HttpContext
```csharp
public class HttpContext
{
    public HttpRequest Request { get; }
    public HttpResponse Response { get; }
    public ConnectionInfo Connection { get; }
    public WebSocketManager WebSockets { get; }
    public ClaimsPrincipal User { get; }
    public ISession Session { get; }
    public IServiceProvider RequestServices { get; }
    public IDictionary<object, object> Items { get; }
    public CancellationToken RequestAborted { get; }
}
```

---

## 1. HttpRequest - دریافت اطلاعات درخواست

```csharp
[ApiController]
[Route("api/[controller]")]
public class RequestExampleController : ControllerBase
{
    [HttpGet("info")]
    public IActionResult GetRequestInfo()
    {
        var request = HttpContext.Request;
        
        return Ok(new
        {
            // URL Info
            scheme = request.Scheme,              // "https"
            host = request.Host.ToString(),       // "api.example.com"
            path = request.Path.ToString(),       // "/api/requestexample/info"
            query = request.QueryString.ToString(), // "?page=1&size=10"
            fullUrl = $"{request.Scheme}://{request.Host}{request.Path}{request.QueryString}",
            
            // Method
            method = request.Method,              // "GET"
            
            // Headers
            headers = request.Headers.ToDictionary(h => h.Key, h => h.Value.ToString()),
            contentType = request.ContentType,
            userAgent = request.Headers["User-Agent"].ToString(),
            authorization = request.Headers["Authorization"].ToString(),
            
            // Cookies
            cookies = request.Cookies.ToDictionary(c => c.Key, c => c.Value),
            
            // Query Parameters
            queryParams = request.Query.ToDictionary(q => q.Key, q => q.Value.ToString()),
            
            // Form Data (اگر form باشد)
            hasFormContentType = request.HasFormContentType,
            
            // Connection Info
            remoteIp = request.HttpContext.Connection.RemoteIpAddress?.ToString(),
            localIp = request.HttpContext.Connection.LocalIpAddress?.ToString(),
            
            // Protocol
            protocol = request.Protocol,          // "HTTP/2"
            
            // Body
            contentLength = request.ContentLength,
            
            // Route Values
            routeValues = request.RouteValues.ToDictionary(r => r.Key, r => r.Value?.ToString())
        });
    }
    
    [HttpPost("read-body")]
    public async Task<IActionResult> ReadBody()
    {
        var request = HttpContext.Request;
        
        // خواندن Body (مهم: فقط یکبار قابل خواندنه!)
        request.EnableBuffering(); // اجازه خواندن چند باره
        
        using var reader = new StreamReader(request.Body, Encoding.UTF8, leaveOpen: true);
        var body = await reader.ReadToEndAsync();
        request.Body.Position = 0; // Reset برای خواندن بعدی
        
        return Ok(new { body });
    }
}
```

---

## 2. HttpResponse - ارسال پاسخ

```csharp
[ApiController]
[Route("api/[controller]")]
public class ResponseExampleController : ControllerBase
{
    [HttpGet("custom")]
    public async Task<IActionResult> CustomResponse()
    {
        var response = HttpContext.Response;
        
        // تنظیم Status Code
        response.StatusCode = 200;
        
        // تنظیم Headers
        response.Headers.Add("X-Custom-Header", "My Value");
        response.Headers.Add("X-Request-ID", Guid.NewGuid().ToString());
        response.Headers.Add("X-RateLimit-Limit", "100");
        response.Headers.Add("X-RateLimit-Remaining", "99");
        response.Headers.Add("X-RateLimit-Reset", DateTimeOffset.UtcNow.AddHours(1).ToUnixTimeSeconds().ToString());
        
        // تنظیم Content-Type
        response.ContentType = "application/json";
        
        // تنظیم Cookie
        response.Cookies.Append("my_cookie", "value", new CookieOptions
        {
            HttpOnly = true,
            Secure = true,
            SameSite = SameSiteMode.Strict,
            Expires = DateTimeOffset.Now.AddDays(7)
        });
        
        // نوشتن به Response
        await response.WriteAsync(JsonSerializer.Serialize(new { message = "Custom response" }));
        
        return new EmptyResult();
    }
    
    [HttpGet("stream")]
    public async Task StreamResponse()
    {
        var response = HttpContext.Response;
        response.ContentType = "text/plain";
        
        for (int i = 0; i < 10; i++)
        {
            await response.WriteAsync($"Line {i}\n");
            await response.Body.FlushAsync();
            await Task.Delay(1000);
        }
    }
    
    [HttpGet("file")]
    public IActionResult DownloadFile()
    {
        var response = HttpContext.Response;
        
        // تنظیم Headers برای دانلود
        response.Headers.Add("Content-Disposition", "attachment; filename=\"data.txt\"");
        
        var bytes = Encoding.UTF8.GetBytes("File content");
        return File(bytes, "text/plain", "data.txt");
    }
}
```

---

## 3. ConnectionInfo - اطلاعات اتصال

```csharp
[HttpGet("connection")]
public IActionResult GetConnectionInfo()
{
    var connection = HttpContext.Connection;
    
    return Ok(new
    {
        // IP Addresses
        remoteIpAddress = connection.RemoteIpAddress?.ToString(),
        remotePort = connection.RemotePort,
        localIpAddress = connection.LocalIpAddress?.ToString(),
        localPort = connection.LocalPort,
        
        // Connection ID
        id = connection.Id,
        
        // Client Certificate (اگر HTTPS با client cert باشد)
        clientCertificate = connection.ClientCertificate?.Subject
    });
}
```

---

## 4. User & Authentication

```csharp
[ApiController]
[Route("api/[controller]")]
public class AuthExampleController : ControllerBase
{
    [HttpGet("user-info")]
    [Authorize] // نیاز به Authentication
    public IActionResult GetUserInfo()
    {
        var user = HttpContext.User;
        
        return Ok(new
        {
            // شناسه کاربر
            userId = user.FindFirst(ClaimTypes.NameIdentifier)?.Value,
            
            // نام کاربر
            userName = user.FindFirst(ClaimTypes.Name)?.Value,
            
            // ایمیل
            email = user.FindFirst(ClaimTypes.Email)?.Value,
            
            // نقش‌ها
            roles = user.FindAll(ClaimTypes.Role).Select(c => c.Value).ToList(),
            
            // همه Claims
            allClaims = user.Claims.Select(c => new 
            { 
                type = c.Type, 
                value = c.Value 
            }).ToList(),
            
            // بررسی نقش
            isAdmin = user.IsInRole("Admin"),
            
            // Identity
            authenticationType = user.Identity?.AuthenticationType,
            isAuthenticated = user.Identity?.IsAuthenticated
        });
    }
    
    [HttpGet("check-claim")]
    public IActionResult CheckClaim()
    {
        var hasClaim = HttpContext.User.HasClaim(c => 
            c.Type == "permission" && c.Value == "edit");
        
        return Ok(new { hasClaim });
    }
}
```

---

## 5. Session

```csharp
// Startup.cs
services.AddDistributedMemoryCache();
services.AddSession(options =>
{
    options.IdleTimeout = TimeSpan.FromMinutes(30);
    options.Cookie.HttpOnly = true;
    options.Cookie.IsEssential = true;
});

app.UseSession();

// Controller
[ApiController]
[Route("api/[controller]")]
public class SessionExampleController : ControllerBase
{
    [HttpPost("set")]
    public IActionResult SetSession([FromBody] Dictionary<string, string> data)
    {
        var session = HttpContext.Session;
        
        // ذخیره string
        session.SetString("username", data["username"]);
        
        // ذخیره int
        session.SetInt32("user_id", 123);
        
        // ذخیره object (به صورت JSON)
        var user = new User { Id = 1, Name = "Ali" };
        session.SetString("user", JsonSerializer.Serialize(user));
        
        return Ok(new { message = "Session set" });
    }
    
    [HttpGet("get")]
    public IActionResult GetSession()
    {
        var session = HttpContext.Session;
        
        var username = session.GetString("username");
        var userId = session.GetInt32("user_id");
        var userJson = session.GetString("user");
        var user = userJson != null ? JsonSerializer.Deserialize<User>(userJson) : null;
        
        return Ok(new { username, userId, user });
    }
    
    [HttpPost("clear")]
    public IActionResult ClearSession()
    {
        HttpContext.Session.Clear();
        return Ok(new { message = "Session cleared" });
    }
}
```

---

## 6. Items - ذخیره موقت در Request

```csharp
// Middleware
public class MyMiddleware
{
    private readonly RequestDelegate _next;
    
    public async Task InvokeAsync(HttpContext context)
    {
        // ذخیره داده برای استفاده در Controller
        context.Items["RequestStartTime"] = DateTime.UtcNow;
        context.Items["CustomData"] = new { Key = "Value" };
        
        await _next(context);
        
        // محاسبه زمان پردازش
        var startTime = (DateTime)context.Items["RequestStartTime"]!;
        var duration = DateTime.UtcNow - startTime;
        Console.WriteLine($"Request took: {duration.TotalMilliseconds}ms");
    }
}

// Controller
[HttpGet]
public IActionResult Get()
{
    var startTime = (DateTime)HttpContext.Items["RequestStartTime"]!;
    var customData = HttpContext.Items["CustomData"];
    
    return Ok(new { startTime, customData });
}
```

---

## 7. RequestServices - Dependency Injection

```csharp
[HttpGet("di-example")]
public IActionResult DIExample()
{
    // دریافت service از DI container
    var logger = HttpContext.RequestServices.GetService<ILogger<MyController>>();
    var dbContext = HttpContext.RequestServices.GetRequiredService<ApplicationDbContext>();
    
    logger?.LogInformation("Accessed via RequestServices");
    
    return Ok();
}
```

---

## 8. RequestAborted - لغو درخواست

```csharp
[HttpGet("long-running")]
public async Task<IActionResult> LongRunning()
{
    var cancellationToken = HttpContext.RequestAborted;
    
    try
    {
        for (int i = 0; i < 100; i++)
        {
            // بررسی لغو
            cancellationToken.ThrowIfCancellationRequested();
            
            await Task.Delay(1000, cancellationToken);
            
            // ارسال جزئی پاسخ
            await Response.WriteAsync($"Progress: {i}%\n");
            await Response.Body.FlushAsync(cancellationToken);
        }
        
        return Ok();
    }
    catch (OperationCanceledException)
    {
        // Client قطع کرد
        return StatusCode(499, "Client Closed Request");
    }
}
```

---

## 9. IHttpContextAccessor - دسترسی در Services

```csharp
// Startup.cs
services.AddHttpContextAccessor();

// Service
public class MyService
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    
    public MyService(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }
    
    public string GetUserIp()
    {
        var httpContext = _httpContextAccessor.HttpContext;
        return httpContext?.Connection.RemoteIpAddress?.ToString() ?? "Unknown";
    }
    
    public string GetCurrentUserId()
    {
        var httpContext = _httpContextAccessor.HttpContext;
        return httpContext?.User.FindFirst(ClaimTypes.NameIdentifier)?.Value ?? "";
    }
    
    public void SetResponseHeader(string key, string value)
    {
        var httpContext = _httpContextAccessor.HttpContext;
        httpContext?.Response.Headers.Add(key, value);
    }
}
```

---

# بخش 1️⃣1️⃣: Middleware در ASP.NET Core

## Middleware چیست؟
Middleware یک component است که در pipeline درخواست قرار می‌گیرد و می‌تواند request و response را پردازش کند.

## Pipeline Middleware
```
Request → M1 → M2 → M3 → Endpoint → M3 → M2 → M1 → Response
```

---

## انواع Middleware

### 1. Inline Middleware
```csharp
app.Use(async (context, next) =>
{
    // قبل از next middleware
    Console.WriteLine($"Request: {context.Request.Path}");
    
    await next.Invoke();
    
    // بعد از next middleware
    Console.WriteLine($"Response: {context.Response.StatusCode}");
});
```

### 2. Terminal Middleware
```csharp
app.Run(async context =>
{
    // این middleware pipeline رو قطع می‌کنه
    await context.Response.WriteAsync("Hello from terminal middleware");
});
```

### 3. Custom Middleware Class
```csharp
// RequestLoggingMiddleware.cs
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;
    
    public RequestLoggingMiddleware(
        RequestDelegate next,
        ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var startTime = DateTime.UtcNow;
        var requestId = Guid.NewGuid().ToString();
        
        // ذخیره Request ID
        context.Items["RequestId"] = requestId;
        context.Response.Headers.Add("X-Request-ID", requestId);
        
        // Log Request
        _logger.LogInformation(
            "Request {RequestId}: {Method} {Path} from {IP}",
            requestId,
            context.Request.Method,
            context.Request.Path,
            context.Connection.RemoteIpAddress);
        
        try
        {
            await _next(context);
        }
        finally
        {
            var duration = DateTime.UtcNow - startTime;
            
            // Log Response
            _logger.LogInformation(
                "Response {RequestId}: {StatusCode} in {Duration}ms",
                requestId,
                context.Response.StatusCode,
                duration.TotalMilliseconds);
        }
    }
}

// Extension Method
public static class RequestLoggingMiddlewareExtensions
{
    public static IApplicationBuilder UseRequestLogging(
        this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<RequestLoggingMiddleware>();
    }
}

// Startup.cs
app.UseRequestLogging();
```

---

## Middleware Examples

### Authentication Middleware
```csharp
public class CustomAuthMiddleware
{
    private readonly RequestDelegate _next;
    
    public async Task InvokeAsync(HttpContext context)
    {
        var token = context.Request.Headers["Authorization"].ToString();
        
        if (string.IsNullOrEmpty(token))
        {
            token = context.Request.Cookies["auth_token"];
        }
        
        if (!string.IsNullOrEmpty(token))
        {
            token = token.Replace("Bearer ", "");
            
            if (ValidateToken(token))
            {
                var claims = GetClaimsFromToken(token);
                var identity = new ClaimsIdentity(claims, "Custom");
                context.User = new ClaimsPrincipal(identity);
            }
        }
        
        await _next(context);
    }
}
```

### Rate Limiting Middleware
```csharp
public class RateLimitingMiddleware
{
    private readonly RequestDelegate _next;
    private static readonly Dictionary<string, (int Count, DateTime ResetTime)> _requests = new();
    private const int MaxRequests = 100;
    private static readonly TimeSpan Window = TimeSpan.FromMinutes(1);
    
    public async Task InvokeAsync(HttpContext context)
    {
        var clientIp = context.Connection.RemoteIpAddress?.ToString() ?? "unknown";
        
        lock (_requests)
        {
            if (!_requests.ContainsKey(clientIp))
            {
                _requests[clientIp] = (1, DateTime.UtcNow.Add(Window));
            }
            else
            {
                var (count, resetTime) = _requests[clientIp];
                
                if (DateTime.UtcNow > resetTime)
                {
                    _requests[clientIp] = (1, DateTime.UtcNow.Add(Window));
                }
                else if (count >= MaxRequests)
                {
                    context.Response.StatusCode = 429; // Too Many Requests
                    context.Response.Headers.Add("Retry-After", ((int)(resetTime - DateTime.UtcNow).TotalSeconds).ToString());
                    await context.Response.WriteAsync("Rate limit exceeded");
                    return;
                }
                else
                {
                    _requests[clientIp] = (count + 1, resetTime);
                }
            }
            
            var (currentCount, currentResetTime) = _requests[clientIp];
            context.Response.Headers.Add("X-RateLimit-Limit", MaxRequests.ToString());
            context.Response.Headers.Add("X-RateLimit-Remaining", (MaxRequests - currentCount).ToString());
            context.Response.Headers.Add("X-RateLimit-Reset", new DateTimeOffset(currentResetTime).ToUnixTimeSeconds().ToString());
        }
        
        await _next(context);
    }
}
```

### Request/Response Logging Middleware
```csharp
public class RequestResponseLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger _logger;
    
    public async Task InvokeAsync(HttpContext context)
    {
        // Log Request
        context.Request.EnableBuffering();
        var requestBody = await new StreamReader(context.Request.Body).ReadToEndAsync();
        context.Request.Body.Position = 0;
        
        _logger.LogInformation("Request: {Method} {Path} Body: {Body}",
            context.Request.Method,
            context.Request.Path,
            requestBody);
        
        // Capture Response
        var originalBodyStream = context.Response.Body;
        using var responseBody = new MemoryStream();
        context.Response.Body = responseBody;
        
        await _next(context);
        
        // Log Response
        context.Response.Body.Seek(0, SeekOrigin.Begin);
        var responseBodyText = await new StreamReader(context.Response.Body).ReadToEndAsync();
        context.Response.Body.Seek(0, SeekOrigin.Begin);
        
        _logger.LogInformation("Response: {StatusCode} Body: {Body}",
            context.Response.StatusCode,
            responseBodyText);
        
        await responseBody.CopyToAsync(originalBodyStream);
    }
}
```

---

---

# بخش 1️⃣2️⃣: HTTP Status Codes و Response Types

## Status Codes کامل

### 1xx - Informational
| Code | نام | کاربرد |
|------|-----|--------|
| 100 | Continue | ادامه ارسال request |
| 101 | Switching Protocols | تغییر به WebSocket |

### 2xx - Success
| Code | نام | استفاده در ASP.NET Core |
|------|-----|------------------------|
| 200 | OK | `Ok(data)` |
| 201 | Created | `CreatedAtAction()`, `CreatedAtRoute()` |
| 202 | Accepted | `Accepted()` - برای async operations |
| 204 | No Content | `NoContent()` - معمولا DELETE |
| 206 | Partial Content | `File()` با range |

### 3xx - Redirection
| Code | نام | استفاده |
|------|-----|---------|
| 301 | Moved Permanently | `RedirectPermanent()` |
| 302 | Found | `Redirect()` |
| 304 | Not Modified | برای caching با ETag |
| 307 | Temporary Redirect | `RedirectPreserveMethod()` |
| 308 | Permanent Redirect | `RedirectPermanentPreserveMethod()` |

### 4xx - Client Errors
| Code | نام | استفاده | مثال |
|------|-----|---------|------|
| 400 | Bad Request | `BadRequest()` | validation خطا |
| 401 | Unauthorized | `Unauthorized()` | نیاز به login |
| 403 | Forbidden | `Forbid()` | دسترسی ندارد |
| 404 | Not Found | `NotFound()` | منبع پیدا نشد |
| 405 | Method Not Allowed | خودکار | GET به جای POST |
| 406 | Not Acceptable | خودکار | Accept header نامعتبر |
| 408 | Request Timeout | - | timeout |
| 409 | Conflict | `Conflict()` | تداخل (مثلا duplicate) |
| 410 | Gone | - | منبع دیگه وجود نداره |
| 415 | Unsupported Media Type | خودکار | Content-Type اشتباه |
| 422 | Unprocessable Entity | `UnprocessableEntity()` | validation منطقی |
| 429 | Too Many Requests | سفارشی | rate limit |

### 5xx - Server Errors
| Code | نام | استفاده |
|------|-----|---------|
| 500 | Internal Server Error | خودکار یا `StatusCode(500)` |
| 501 | Not Implemented | `StatusCode(501)` |
| 502 | Bad Gateway | Gateway error |
| 503 | Service Unavailable | `StatusCode(503)` |
| 504 | Gateway Timeout | Gateway timeout |

---

## Response Types در ASP.NET Core

### 1. OkResult (200)
```csharp
[HttpGet]
public IActionResult Get()
{
    var data = GetData();
    return Ok(data); // 200 + data
}

// یا
public ActionResult<List<User>> Get()
{
    return Ok(users);
}
```

### 2. CreatedResult (201)
```csharp
[HttpPost]
public IActionResult Create([FromBody] CreateUserDto dto)
{
    var user = CreateUser(dto);
    
    // روش 1: با location
    return Created($"/api/users/{user.Id}", user);
    
    // روش 2: با action name
    return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
    
    // روش 3: با route name
    return CreatedAtRoute("GetUser", new { id = user.Id }, user);
}

[HttpGet("{id}", Name = "GetUser")]
public IActionResult GetById(int id)
{
    return Ok(GetUser(id));
}
```

### 3. NoContentResult (204)
```csharp
[HttpPut("{id}")]
public IActionResult Update(int id, [FromBody] UpdateUserDto dto)
{
    UpdateUser(id, dto);
    return NoContent(); // 204 - بدون body
}

[HttpDelete("{id}")]
public IActionResult Delete(int id)
{
    DeleteUser(id);
    return NoContent();
}
```

### 4. BadRequestResult (400)
```csharp
[HttpPost]
public IActionResult Create([FromBody] CreateUserDto dto)
{
    if (!ModelState.IsValid)
    {
        return BadRequest(ModelState); // 400 + validation errors
    }
    
    // بررسی سفارشی
    if (UserExists(dto.Email))
    {
        return BadRequest(new { error = "Email already exists" });
    }
    
    return Ok();
}
```

### 5. NotFoundResult (404)
```csharp
[HttpGet("{id}")]
public IActionResult GetById(int id)
{
    var user = FindUser(id);
    
    if (user == null)
    {
        return NotFound(); // 404
        // یا با پیام
        // return NotFound(new { message = $"User {id} not found" });
    }
    
    return Ok(user);
}
```

### 6. UnauthorizedResult (401)
```csharp
[HttpGet("protected")]
public IActionResult Protected()
{
    if (!User.Identity.IsAuthenticated)
    {
        return Unauthorized(); // 401
        // یا با پیام
        // return Unauthorized(new { message = "Please login" });
    }
    
    return Ok(data);
}
```

### 7. ForbidResult (403)
```csharp
[HttpDelete("{id}")]
public IActionResult Delete(int id)
{
    if (!User.IsInRole("Admin"))
    {
        return Forbid(); // 403
        // یا
        // return StatusCode(403, new { message = "Admin role required" });
    }
    
    DeleteUser(id);
    return NoContent();
}
```

### 8. ConflictResult (409)
```csharp
[HttpPost]
public IActionResult Create([FromBody] CreateUserDto dto)
{
    if (EmailExists(dto.Email))
    {
        return Conflict(new { message = "Email already exists" }); // 409
    }
    
    var user = CreateUser(dto);
    return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
}
```

### 9. Custom Status Code
```csharp
[HttpGet("rate-limited")]
public IActionResult RateLimited()
{
    if (IsRateLimited())
    {
        Response.Headers.Add("Retry-After", "60");
        return StatusCode(429, new { message = "Too many requests" });
    }
    
    return Ok();
}
```

### 10. FileResult
```csharp
[HttpGet("download")]
public IActionResult Download()
{
    var bytes = System.IO.File.ReadAllBytes("file.pdf");
    return File(bytes, "application/pdf", "document.pdf");
}

[HttpGet("stream")]
public IActionResult Stream()
{
    var stream = new FileStream("large-file.zip", FileMode.Open);
    return File(stream, "application/zip", "archive.zip");
}

[HttpGet("image")]
public IActionResult GetImage()
{
    var image = GetImageBytes();
    return File(image, "image/jpeg");
}
```

---

# بخش 1️⃣3️⃣: API Versioning

## چرا Versioning؟
- Breaking changes بدون مشکل برای کلاینت‌های قدیمی
- مدیریت بهتر تغییرات
- پشتیبانی از چند نسخه همزمان

---

## روش‌های Versioning

### 1. URL Path Versioning
```csharp
// Startup.cs
services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true; // اضافه کردن header: api-supported-versions
});

// Controllers/V1/UsersController.cs
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiController]
public class UsersController : ControllerBase
{
    [HttpGet]
    public IActionResult GetV1()
    {
        return Ok(new { version = "1.0", users = GetUsersV1() });
    }
}

// Controllers/V2/UsersController.cs
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiController]
public class UsersController : ControllerBase
{
    [HttpGet]
    public IActionResult GetV2()
    {
        return Ok(new { version = "2.0", users = GetUsersV2() });
    }
}

// استفاده:
// GET /api/v1/users
// GET /api/v2/users
```

### 2. Query String Versioning
```csharp
services.AddApiVersioning(options =>
{
    options.ApiVersionReader = new QueryStringApiVersionReader("api-version");
});

[ApiVersion("1.0")]
[ApiVersion("2.0")]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet]
    [MapToApiVersion("1.0")]
    public IActionResult GetV1()
    {
        return Ok(new { version = "1.0" });
    }
    
    [HttpGet]
    [MapToApiVersion("2.0")]
    public IActionResult GetV2()
    {
        return Ok(new { version = "2.0" });
    }
}

// استفاده:
// GET /api/users?api-version=1.0
// GET /api/users?api-version=2.0
```

### 3. Header Versioning
```csharp
services.AddApiVersioning(options =>
{
    options.ApiVersionReader = new HeaderApiVersionReader("X-API-Version");
});

// استفاده:
// GET /api/users
// Header: X-API-Version: 1.0
```

### 4. Media Type Versioning
```csharp
services.AddApiVersioning(options =>
{
    options.ApiVersionReader = new MediaTypeApiVersionReader();
});

// استفاده:
// GET /api/users
// Header: Accept: application/json; version=1.0
```

---

## Versioning در Angular

```typescript
// api.service.ts
@Injectable({ providedIn: 'root' })
export class ApiService {
  private http = inject(HttpClient);
  private baseUrl = 'https://api.example.com/api';
  
  // URL Path Version
  getUsersV1(): Observable<User[]> {
    return this.http.get<User[]>(`${this.baseUrl}/v1/users`);
  }
  
  getUsersV2(): Observable<User[]> {
    return this.http.get<User[]>(`${this.baseUrl}/v2/users`);
  }
  
  // Query String Version
  getUsers(version: string = '1.0'): Observable<User[]> {
    return this.http.get<User[]>(`${this.baseUrl}/users`, {
      params: { 'api-version': version }
    });
  }
  
  // Header Version
  getUsersWithHeader(version: string = '1.0'): Observable<User[]> {
    return this.http.get<User[]>(`${this.baseUrl}/users`, {
      headers: { 'X-API-Version': version }
    });
  }
}

// Interceptor برای version خودکار
export const versionInterceptor: HttpInterceptorFn = (req, next) => {
  // اضافه کردن version به همه درخواست‌ها
  const cloned = req.clone({
    setHeaders: {
      'X-API-Version': '2.0'
    }
  });
  return next(cloned);
};
```

---

# بخش 1️⃣4️⃣: Pagination, Filtering & Sorting

## Pagination

### Offset-based Pagination
```csharp
public class PaginationParams
{
    public int Page { get; set; } = 1;
    public int PageSize { get; set; } = 10;
    
    public int Skip => (Page - 1) * PageSize;
}

public class PagedResult<T>
{
    public List<T> Items { get; set; }
    public int TotalCount { get; set; }
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPrevious => Page > 1;
    public bool HasNext => Page < TotalPages;
}

[HttpGet]
public IActionResult GetUsers([FromQuery] PaginationParams pagination)
{
    var query = _context.Users.AsQueryable();
    
    var totalCount = query.Count();
    
    var items = query
        .Skip(pagination.Skip)
        .Take(pagination.PageSize)
        .ToList();
    
    var result = new PagedResult<User>
    {
        Items = items,
        TotalCount = totalCount,
        Page = pagination.Page,
        PageSize = pagination.PageSize
    };
    
    // اضافه کردن Pagination Headers
    Response.Headers.Add("X-Total-Count", totalCount.ToString());
    Response.Headers.Add("X-Page", pagination.Page.ToString());
    Response.Headers.Add("X-Page-Size", pagination.PageSize.ToString());
    Response.Headers.Add("X-Total-Pages", result.TotalPages.ToString());
    
    return Ok(result);
}
```

### Cursor-based Pagination
```csharp
public class CursorPaginationParams
{
    public string? Cursor { get; set; } // ID یا timestamp آخرین آیتم
    public int Limit { get; set; } = 10;
}

[HttpGet("cursor")]
public IActionResult GetUsersCursor([FromQuery] CursorPaginationParams pagination)
{
    var query = _context.Users.OrderBy(u => u.Id).AsQueryable();
    
    if (!string.IsNullOrEmpty(pagination.Cursor))
    {
        var cursorId = int.Parse(pagination.Cursor);
        query = query.Where(u => u.Id > cursorId);
    }
    
    var items = query.Take(pagination.Limit + 1).ToList();
    
    var hasMore = items.Count > pagination.Limit;
    var results = items.Take(pagination.Limit).ToList();
    
    var nextCursor = hasMore ? results.Last().Id.ToString() : null;
    
    return Ok(new
    {
        items = results,
        nextCursor,
        hasMore
    });
}
```

---

## Filtering

```csharp
public class UserFilterParams
{
    public string? Name { get; set; }
    public string? Email { get; set; }
    public int? MinAge { get; set; }
    public int? MaxAge { get; set; }
    public bool? IsActive { get; set; }
    public List<string>? Roles { get; set; }
}

[HttpGet("filter")]
public IActionResult GetFiltered([FromQuery] UserFilterParams filter)
{
    var query = _context.Users.AsQueryable();
    
    // Name filter
    if (!string.IsNullOrEmpty(filter.Name))
    {
        query = query.Where(u => u.Name.Contains(filter.Name));
    }
    
    // Email filter
    if (!string.IsNullOrEmpty(filter.Email))
    {
        query = query.Where(u => u.Email.Contains(filter.Email));
    }
    
    // Age range
    if (filter.MinAge.HasValue)
    {
        query = query.Where(u => u.Age >= filter.MinAge.Value);
    }
    
    if (filter.MaxAge.HasValue)
    {
        query = query.Where(u => u.Age <= filter.MaxAge.Value);
    }
    
    // Boolean filter
    if (filter.IsActive.HasValue)
    {
        query = query.Where(u => u.IsActive == filter.IsActive.Value);
    }
    
    // List filter
    if (filter.Roles != null && filter.Roles.Any())
    {
        query = query.Where(u => u.Roles.Any(r => filter.Roles.Contains(r.Name)));
    }
    
    var results = query.ToList();
    
    return Ok(results);
}
```

---

## Sorting

```csharp
public class SortingParams
{
    public string? SortBy { get; set; } = "id";
    public string? Order { get; set; } = "asc"; // asc یا desc
}

[HttpGet("sort")]
public IActionResult GetSorted([FromQuery] SortingParams sorting)
{
    var query = _context.Users.AsQueryable();
    
    // Dynamic sorting
    query = sorting.SortBy?.ToLower() switch
    {
        "name" => sorting.Order == "desc" 
            ? query.OrderByDescending(u => u.Name) 
            : query.OrderBy(u => u.Name),
        "email" => sorting.Order == "desc"
            ? query.OrderByDescending(u => u.Email)
            : query.OrderBy(u => u.Email),
        "age" => sorting.Order == "desc"
            ? query.OrderByDescending(u => u.Age)
            : query.OrderBy(u => u.Age),
        _ => sorting.Order == "desc"
            ? query.OrderByDescending(u => u.Id)
            : query.OrderBy(u => u.Id)
    };
    
    var results = query.ToList();
    
    return Ok(results);
}
```

---

## همه با هم (Pagination + Filtering + Sorting)
```csharp
public class QueryParams
{
    // Pagination
    public int Page { get; set; } = 1;
    public int PageSize { get; set; } = 10;
    
    // Filtering
    public string? Search { get; set; }
    public int? MinAge { get; set; }
    public int? MaxAge { get; set; }
    public bool? IsActive { get; set; }
    
    // Sorting
    public string? SortBy { get; set; } = "id";
    public string? Order { get; set; } = "asc";
}

[HttpGet("advanced")]
public IActionResult GetAdvanced([FromQuery] QueryParams queryParams)
{
    var query = _context.Users.AsQueryable();
    
    // Filtering
    if (!string.IsNullOrEmpty(queryParams.Search))
    {
        query = query.Where(u => 
            u.Name.Contains(queryParams.Search) || 
            u.Email.Contains(queryParams.Search));
    }
    
    if (queryParams.MinAge.HasValue)
    {
        query = query.Where(u => u.Age >= queryParams.MinAge.Value);
    }
    
    if (queryParams.MaxAge.HasValue)
    {
        query = query.Where(u => u.Age <= queryParams.MaxAge.Value);
    }
    
    if (queryParams.IsActive.HasValue)
    {
        query = query.Where(u => u.IsActive == queryParams.IsActive.Value);
    }
    
    // Sorting
    query = queryParams.SortBy?.ToLower() switch
    {
        "name" => queryParams.Order == "desc" 
            ? query.OrderByDescending(u => u.Name) 
            : query.OrderBy(u => u.Name),
        "age" => queryParams.Order == "desc"
            ? query.OrderByDescending(u => u.Age)
            : query.OrderBy(u => u.Age),
        _ => queryParams.Order == "desc"
            ? query.OrderByDescending(u => u.Id)
            : query.OrderBy(u => u.Id)
    };
    
    // Total count before pagination
    var totalCount = query.Count();
    
    // Pagination
    var items = query
        .Skip((queryParams.Page - 1) * queryParams.PageSize)
        .Take(queryParams.PageSize)
        .ToList();
    
    var result = new PagedResult<User>
    {
        Items = items,
        TotalCount = totalCount,
        Page = queryParams.Page,
        PageSize = queryParams.PageSize
    };
    
    return Ok(result);
}
```

---

## Pagination/Filtering/Sorting در Angular

```typescript
// models/query-params.ts
export interface QueryParams {
  page?: number;
  pageSize?: number;
  search?: string;
  sortBy?: string;
  order?: 'asc' | 'desc';
  minAge?: number;
  maxAge?: number;
  isActive?: boolean;
}

export interface PagedResult<T> {
  items: T[];
  totalCount: number;
  page: number;
  pageSize: number;
  totalPages: number;
  hasPrevious: boolean;
  hasNext: boolean;
}

// user.service.ts
@Injectable({ providedIn: 'root' })
export class UserService {
  private http = inject(HttpClient);
  private baseUrl = 'https://api.example.com/api';
  
  getUsers(params: QueryParams): Observable<PagedResult<User>> {
    // حذف undefined values
    const httpParams = new HttpParams({ fromObject: this.cleanParams(params) });
    
    return this.http.get<PagedResult<User>>(`${this.baseUrl}/users/advanced`, {
      params: httpParams,
      observe: 'response' // برای دریافت headers
    }).pipe(
      map(response => {
        // خواندن pagination info از headers
        const totalCount = response.headers.get('X-Total-Count');
        const page = response.headers.get('X-Page');
        
        return response.body!;
      })
    );
  }
  
  private cleanParams(params: any): any {
    const cleaned: any = {};
    Object.keys(params).forEach(key => {
      if (params[key] !== undefined && params[key] !== null) {
        cleaned[key] = params[key];
      }
    });
    return cleaned;
  }
}

// user-list.component.ts
@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [CommonModule, FormsModule],
  template: `
    <div>
      <!-- Search -->
      <input [(ngModel)]="queryParams.search" 
             (ngModelChange)="onSearchChange()"
             placeholder="Search...">
      
      <!-- Filters -->
      <select [(ngModel)]="queryParams.sortBy" (ngModelChange)="loadUsers()">
        <option value="id">ID</option>
        <option value="name">Name</option>
        <option value="age">Age</option>
      </select>
      
      <select [(ngModel)]="queryParams.order" (ngModelChange)="loadUsers()">
        <option value="asc">Ascending</option>
        <option value="desc">Descending</option>
      </select>
      
      <!-- Users Table -->
      @if (loading()) {
        <p>Loading...</p>
      } @else {
        <table>
          <thead>
            <tr>
              <th (click)="sort('name')">Name</th>
              <th (click)="sort('email')">Email</th>
              <th (click)="sort('age')">Age</th>
            </tr>
          </thead>
          <tbody>
            @for (user of users(); track user.id) {
              <tr>
                <td>{{ user.name }}</td>
                <td>{{ user.email }}</td>
                <td>{{ user.age }}</td>
              </tr>
            }
          </tbody>
        </table>
        
        <!-- Pagination -->
        <div class="pagination">
          <button (click)="previousPage()" 
                  [disabled]="!pagedResult()?.hasPrevious">
            Previous
          </button>
          
          <span>Page {{ pagedResult()?.page }} of {{ pagedResult()?.totalPages }}</span>
          
          <button (click)="nextPage()" 
                  [disabled]="!pagedResult()?.hasNext">
            Next
          </button>
          
          <span>Total: {{ pagedResult()?.totalCount }}</span>
        </div>
      }
    </div>
  `
})
export class UserListComponent implements OnInit {
  private userService = inject(UserService);
  
  users = signal<User[]>([]);
  pagedResult = signal<PagedResult<User> | null>(null);
  loading = signal(false);
  
  queryParams: QueryParams = {
    page: 1,
    pageSize: 10,
    sortBy: 'id',
    order: 'asc'
  };
  
  private searchSubject = new Subject<string>();
  
  ngOnInit() {
    this.loadUsers();
    
    // Debounce search
    this.searchSubject.pipe(
      debounceTime(300),
      distinctUntilChanged()
    ).subscribe(() => {
      this.queryParams.page = 1;
      this.loadUsers();
    });
  }
  
  loadUsers() {
    this.loading.set(true);
    
    this.userService.getUsers(this.queryParams).subscribe({
      next: (result) => {
        this.users.set(result.items);
        this.pagedResult.set(result);
        this.loading.set(false);
      },
      error: (err) => {
        console.error(err);
        this.loading.set(false);
      }
    });
  }
  
  onSearchChange() {
    this.searchSubject.next(this.queryParams.search || '');
  }
  
  sort(field: string) {
    if (this.queryParams.sortBy === field) {
      this.queryParams.order = this.queryParams.order === 'asc' ? 'desc' : 'asc';
    } else {
      this.queryParams.sortBy = field;
      this.queryParams.order = 'asc';
    }
    this.loadUsers();
  }
  
  nextPage() {
    if (this.pagedResult()?.hasNext) {
      this.queryParams.page!++;
      this.loadUsers();
    }
  }
  
  previousPage() {
    if (this.pagedResult()?.hasPrevious) {
      this.queryParams.page!--;
      this.loadUsers();
    }
  }
}
```

---

# بخش 1️⃣5️⃣: Validation عمیق

## Data Annotations

```csharp
public class CreateUserDto
{
    [Required(ErrorMessage = "Name is required")]
    [MinLength(3, ErrorMessage = "Name must be at least 3 characters")]
    [MaxLength(100, ErrorMessage = "Name cannot exceed 100 characters")]
    public string Name { get; set; }
    
    [Required]
    [EmailAddress(ErrorMessage = "Invalid email format")]
    public string Email { get; set; }
    
    [Required]
    [Range(18, 120, ErrorMessage = "Age must be between 18 and 120")]
    public int Age { get; set; }
    
    [Phone(ErrorMessage = "Invalid phone number")]
    public string? Phone { get; set; }
    
    [Url(ErrorMessage = "Invalid URL")]
    public string? Website { get; set; }
    
    [RegularExpression(@"^[a-zA-Z0-9_]+$", ErrorMessage = "Username can only contain letters, numbers, and underscores")]
    public string Username { get; set; }
    
    [Compare("Password", ErrorMessage = "Passwords do not match")]
    public string ConfirmPassword { get; set; }
    
    [Required]
    [StringLength(100, MinimumLength = 8, ErrorMessage = "Password must be 8-100 characters")]
    public string Password { get; set; }
}

[HttpPost]
public IActionResult Create([FromBody] CreateUserDto dto)
{
    if (!ModelState.IsValid)
    {
        var errors = ModelState
            .Where(x => x.Value.Errors.Count > 0)
            .ToDictionary(
                kvp => kvp.Key,
                kvp => kvp.Value.Errors.Select(e => e.ErrorMessage).ToArray()
            );
        
        return BadRequest(new { errors });
    }
    
    // Create user...
    return Ok();
}
```

---

## Custom Validation Attributes

```csharp
// CustomValidationAttributes.cs
public class FutureDate : ValidationAttribute
{
    protected override ValidationResult? IsValid(object? value, ValidationContext validationContext)
    {
        if (value is DateTime date)
        {
            if (date > DateTime.Now)
            {
                return ValidationResult.Success;
            }
            return new ValidationResult("Date must be in the future");
        }
        return new ValidationResult("Invalid date");
    }
}

public class ValidAge : ValidationAttribute
{
    private readonly int _minAge;
    private readonly int _maxAge;
    
    public ValidAge(int minAge, int maxAge)
    {
        _minAge = minAge;
        _maxAge = maxAge;
    }
    
    protected override ValidationResult? IsValid(object? value, ValidationContext validationContext)
    {
        if (value is int age)
        {
            if (age >= _minAge && age <= _maxAge)
            {
                return ValidationResult.Success;
            }
            return new ValidationResult($"Age must be between {_minAge} and {_maxAge}");
        }
        return new ValidationResult("Invalid age");
    }
}

// استفاده
public class CreateEventDto
{
    [Required]
    public string Title { get; set; }
    
    [FutureDate]
    public DateTime EventDate { get; set; }
    
    [ValidAge(18, 65)]
    public int ParticipantAge { get; set; }
}
```

---

## FluentValidation

```bash
dotnet add package FluentValidation.AspNetCore
```

```csharp
// Validators/CreateUserDtoValidator.cs
public class CreateUserDtoValidator : AbstractValidator<CreateUserDto>
{
    public CreateUserDtoValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required")
            .MinimumLength(3).WithMessage("Name must be at least 3 characters")
            .MaximumLength(100).WithMessage("Name cannot exceed 100 characters");
        
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email is required")
            .EmailAddress().WithMessage("Invalid email format")
            .MustAsync(async (email, cancellation) => !await EmailExists(email))
                .WithMessage("Email already exists");
        
        RuleFor(x => x.Age)
            .InclusiveBetween(18, 120).WithMessage("Age must be between 18 and 120");
        
        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(8)
            .Matches(@"[A-Z]").WithMessage("Password must contain uppercase letter")
            .Matches(@"[a-z]").WithMessage("Password must contain lowercase letter")
            .Matches(@"[0-9]").WithMessage("Password must contain number")
            .Matches(@"[\W]").WithMessage("Password must contain special character");
        
        RuleFor(x => x.ConfirmPassword)
            .Equal(x => x.Password).WithMessage("Passwords must match");
        
        RuleFor(x => x.Phone)
            .Matches(@"^\+?[1-9]\d{1,14}$").When(x => !string.IsNullOrEmpty(x.Phone))
            .WithMessage("Invalid phone number");
    }
    
    private async Task<bool> EmailExists(string email)
    {
        // Check database
        return await _context.Users.AnyAsync(u => u.Email == email);
    }
}

// Startup.cs
services.AddFluentValidation(fv =>
{
    fv.RegisterValidatorsFromAssemblyContaining<CreateUserDtoValidator>();
});

// یا Manual:
[HttpPost]
public async Task<IActionResult> Create([FromBody] CreateUserDto dto)
{
    var validator = new CreateUserDtoValidator();
    var result = await validator.ValidateAsync(dto);
    
    if (!result.IsValid)
    {
        var errors = result.Errors
            .GroupBy(e => e.PropertyName)
            .ToDictionary(
                g => g.Key,
                g => g.Select(e => e.ErrorMessage).ToArray()
            );
        
        return BadRequest(new { errors });
    }
    
    return Ok();
}
```

---

## Validation در Angular

```typescript
// reactive-form-validation.component.ts
import { Component } from '@angular/core';
import { FormBuilder, FormGroup, Validators, AbstractControl, ValidationErrors } from '@angular/forms';

// Custom Validators
export class CustomValidators {
  // تطابق رمز عبور
  static passwordMatch(control: AbstractControl): ValidationErrors | null {
    const password = control.get('password');
    const confirmPassword = control.get('confirmPassword');
    
    if (!password || !confirmPassword) {
      return null;
    }
    
    return password.value === confirmPassword.value ? null : { passwordMismatch: true };
  }
  
  // سن معتبر
  static validAge(min: number, max: number) {
    return (control: AbstractControl): ValidationErrors | null => {
      const age = control.value;
      if (age < min || age > max) {
        return { invalidAge: { min, max, actual: age } };
      }
      return null;
    };
  }
  
  // ایمیل یونیک (async)
  static emailExists(userService: UserService) {
    return (control: AbstractControl): Observable<ValidationErrors | null> => {
      if (!control.value) {
        return of(null);
      }
      
      return userService.checkEmailExists(control.value).pipe(
        map(exists => exists ? { emailExists: true } : null),
        catchError(() => of(null))
      );
    };
  }
}

@Component({
  selector: 'app-user-form',
  standalone: true,
  imports: [ReactiveFormsModule, CommonModule],
  template: `
    <form [formGroup]="userForm" (ngSubmit)="onSubmit()">
      <!-- Name -->
      <div>
        <label>Name:</label>
        <input formControlName="name">
        @if (userForm.get('name')?.invalid && userForm.get('name')?.touched) {
          <div class="error">
            @if (userForm.get('name')?.errors?.['required']) {
              <span>Name is required</span>
            }
            @if (userForm.get('name')?.errors?.['minlength']) {
              <span>Name must be at least 3 characters</span>
            }
          </div>
        }
      </div>
      
      <!-- Email -->
      <div>
        <label>Email:</label>
        <input formControlName="email">
        @if (userForm.get('email')?.invalid && userForm.get('email')?.touched) {
          <div class="error">
            @if (userForm.get('email')?.errors?.['required']) {
              <span>Email is required</span>
            }
            @if (userForm.get('email')?.errors?.['email']) {
              <span>Invalid email format</span>
            }
            @if (userForm.get('email')?.errors?.['emailExists']) {
              <span>Email already exists</span>
            }
          </div>
        }
        @if (userForm.get('email')?.pending) {
          <span>Checking email...</span>
        }
      </div>
      
      <!-- Age -->
      <div>
        <label>Age:</label>
        <input type="number" formControlName="age">
        @if (userForm.get('age')?.invalid && userForm.get('age')?.touched) {
          <div class="error">
            @if (userForm.get('age')?.errors?.['invalidAge']; as error) {
              <span>Age must be between {{ error.min }} and {{ error.max }}</span>
            }
          </div>
        }
      </div>
      
      <!-- Password -->
      <div formGroupName="passwords">
        <div>
          <label>Password:</label>
          <input type="password" formControlName="password">
          @if (passwords.get('password')?.invalid && passwords.get('password')?.touched) {
            <div class="error">
              @if (passwords.get('password')?.errors?.['required']) {
                <span>Password is required</span>
              }
              @if (passwords.get('password')?.errors?.['minlength']) {
                <span>Password must be at least 8 characters</span>
              }
            </div>
          }
        </div>
        
        <div>
          <label>Confirm Password:</label>
          <input type="password" formControlName="confirmPassword">
          @if (passwords.errors?.['passwordMismatch'] && passwords.touched) {
            <div class="error">
              <span>Passwords do not match</span>
            </div>
          }
        </div>
      </div>
      
      <button type="submit" [disabled]="userForm.invalid || userForm.pending">
        Submit
      </button>
    </form>
    
    <div>
      <h3>Form Status:</h3>
      <p>Valid: {{ userForm.valid }}</p>
      <p>Invalid: {{ userForm.invalid }}</p>
      <p>Pending: {{ userForm.pending }}</p>
      <p>Touched: {{ userForm.touched }}</p>
      <p>Dirty: {{ userForm.dirty }}</p>
    </div>
  `
})
export class UserFormComponent {
  private fb = inject(FormBuilder);
  private userService = inject(UserService);
  
  userForm: FormGroup;
  
  constructor() {
    this.userForm = this.fb.group({
      name: ['', [Validators.required, Validators.minLength(3), Validators.maxLength(100)]],
      email: ['', 
        [Validators.required, Validators.email],
        [CustomValidators.emailExists(this.userService)] // async validator
      ],
      age: ['', [CustomValidators.validAge(18, 120)]],
      passwords: this.fb.group({
        password: ['', [Validators.required, Validators.minLength(8)]],
        confirmPassword: ['', Validators.required]
      }, { validators: CustomValidators.passwordMatch })
    });
  }
  
  get passwords() {
    return this.userForm.get('passwords') as FormGroup;
  }
  
  onSubmit() {
    if (this.userForm.valid) {
      console.log(this.userForm.value);
      
      const userData = {
        name: this.userForm.value.name,
        email: this.userForm.value.email,
        age: this.userForm.value.age,
        password: this.userForm.value.passwords.password
      };
      
      this.userService.createUser(userData).subscribe({
        next: (user) => console.log('User created:', user),
        error: (err) => console.error('Error:', err)
      });
    }
  }
}
```

---

# بخش 1️⃣6️⃣: Exception Handling و Error Management

## Global Exception Handler

```csharp
// Middleware/GlobalExceptionHandlerMiddleware.cs
public class GlobalExceptionHandlerMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionHandlerMiddleware> _logger;
    private readonly IHostEnvironment _env;
    
    public GlobalExceptionHandlerMiddleware(
        RequestDelegate next,
        ILogger<GlobalExceptionHandlerMiddleware> logger,
        IHostEnvironment env)
    {
        _next = next;
        _logger = logger;
        _env = env;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "An unhandled exception occurred");
            await HandleExceptionAsync(context, ex);
        }
    }
    
    private async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/json";
        
        var response = new ErrorResponse
        {
            RequestId = context.Items["RequestId"]?.ToString() ?? Guid.NewGuid().ToString(),
            Timestamp = DateTime.UtcNow
        };
        
        switch (exception)
        {
            case NotFoundException notFoundEx:
                context.Response.StatusCode = StatusCodes.Status404NotFound;
                response.Message = notFoundEx.Message;
                response.ErrorCode = "NOT_FOUND";
                break;
                
            case ValidationException validationEx:
                context.Response.StatusCode = StatusCodes.Status400BadRequest;
                response.Message = "Validation failed";
                response.ErrorCode = "VALIDATION_ERROR";
                response.Errors = validationEx.Errors;
                break;
                
            case UnauthorizedAccessException:
                context.Response.StatusCode = StatusCodes.Status401Unauthorized;
                response.Message = "Unauthorized access";
                response.ErrorCode = "UNAUTHORIZED";
                break;
                
            case ForbiddenException:
                context.Response.StatusCode = StatusCodes.Status403Forbidden;
                response.Message = "Access forbidden";
                response.ErrorCode = "FORBIDDEN";
                break;
                
            case ConflictException conflictEx:
                context.Response.StatusCode = StatusCodes.Status409Conflict;
                response.Message = conflictEx.Message;
                response.ErrorCode = "CONFLICT";
                break;
                
            default:
                context.Response.StatusCode = StatusCodes.Status500InternalServerError;
                response.Message = "An internal server error occurred";
                response.ErrorCode = "INTERNAL_ERROR";
                
                // فقط در Development جزئیات exception رو نشون بده
                if (_env.IsDevelopment())
                {
                    response.Details = exception.ToString();
                }
                break;
        }
        
        await context.Response.WriteAsJsonAsync(response);
    }
}

// Models/ErrorResponse.cs
public class ErrorResponse
{
    public string RequestId { get; set; }
    public DateTime Timestamp { get; set; }
    public string Message { get; set; }
    public string ErrorCode { get; set; }
    public Dictionary<string, string[]>? Errors { get; set; }
    public string? Details { get; set; }
}

// Custom Exceptions
public class NotFoundException : Exception
{
    public NotFoundException(string message) : base(message) { }
}

public class ValidationException : Exception
{
    public Dictionary<string, string[]> Errors { get; }
    
    public ValidationException(Dictionary<string, string[]> errors) 
        : base("Validation failed")
    {
        Errors = errors;
    }
}

public class ConflictException : Exception
{
    public ConflictException(string message) : base(message) { }
}

public class ForbiddenException : Exception
{
    public ForbiddenException() : base("Access forbidden") { }
}

// Startup.cs
app.UseMiddleware<GlobalExceptionHandlerMiddleware>();
```

---

## استفاده از Custom Exceptions

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(int id)
    {
        var user = await _context.Users.FindAsync(id);
        
        if (user == null)
        {
            throw new NotFoundException($"User with ID {id} not found");
        }
        
        return Ok(user);
    }
    
    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateUserDto dto)
    {
        if (await _context.Users.AnyAsync(u => u.Email == dto.Email))
        {
            throw new ConflictException($"User with email {dto.Email} already exists");
        }
        
        var user = new User { Name = dto.Name, Email = dto.Email };
        _context.Users.Add(user);
        await _context.SaveChangesAsync();
        
        return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
    }
    
    [HttpDelete("{id}")]
    [Authorize(Roles = "Admin")]
    public async Task<IActionResult> Delete(int id)
    {
        if (!User.IsInRole("Admin"))
        {
            throw new ForbiddenException();
        }
        
        var user = await _context.Users.FindAsync(id);
        
        if (user == null)
        {
            throw new NotFoundException($"User with ID {id} not found");
        }
        
        _context.Users.Remove(user);
        await _context.SaveChangesAsync();
        
        return NoContent();
    }
}
```

---

## Error Handling در Angular

```typescript
// error.interceptor.ts
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      let errorMessage = 'An error occurred';
      let errorDetails: any = null;
      
      if (error.error instanceof ErrorEvent) {
        // Client-side error
        errorMessage = `Client Error: ${error.error.message}`;
      } else {
        // Server-side error
        errorMessage = error.error?.message || error.message;
        errorDetails = error.error;
        
        switch (error.status) {
          case 400:
            console.error('Bad Request:', errorDetails);
            break;
          case 401:
            console.error('Unauthorized - redirecting to login');
            inject(Router).navigate(['/login']);
            break;
          case 403:
            console.error('Forbidden');
            break;
          case 404:
            console.error('Not Found:', errorMessage);
            break;
          case 409:
            console.error('Conflict:', errorMessage);
            break;
          case 500:
            console.error('Internal Server Error');
            break;
        }
      }
      
      // Show error to user (با toast service یا notification)
      const notificationService = inject(NotificationService);
      notificationService.showError(errorMessage);
      
      return throwError(() => ({
        message: errorMessage,
        status: error.status,
        details: errorDetails
      }));
    })
  );
};

// notification.service.ts
@Injectable({ providedIn: 'root' })
export class NotificationService {
  private notifications = signal<Notification[]>([]);
  
  showError(message: string) {
    this.addNotification({ type: 'error', message });
  }
  
  showSuccess(message: string) {
    this.addNotification({ type: 'success', message });
  }
  
  private addNotification(notification: Notification) {
    this.notifications.update(notifications => [...notifications, notification]);
    
    // حذف بعد از 5 ثانیه
    setTimeout(() => {
      this.notifications.update(notifications => 
        notifications.filter(n => n !== notification)
      );
    }, 5000);
  }
}
```

---

# بخش 1️⃣7️⃣: Swagger / OpenAPI

## نصب و پیکربندی Swagger

```bash
dotnet add package Swashbuckle.AspNetCore
```

```csharp
// Program.cs یا Startup.cs
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllers();
        
        services.AddSwaggerGen(options =>
        {
            options.SwaggerDoc("v1", new OpenApiInfo
            {
                Version = "v1",
                Title = "My API",
                Description = "API Documentation",
                Contact = new OpenApiContact
                {
                    Name = "Support",
                    Email = "support@example.com",
                    Url = new Uri("https://example.com/support")
                },
                License = new OpenApiLicense
                {
                    Name = "MIT",
                    Url = new Uri("https://opensource.org/licenses/MIT")
                }
            });
            
            // XML Documentation
            var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
            var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
            options.IncludeXmlComments(xmlPath);
            
            // Bearer Token Authentication
            options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
            {
                Description = "JWT Authorization header using the Bearer scheme. Example: \"Bearer {token}\"",
                Name = "Authorization",
                In = ParameterLocation.Header,
                Type = SecuritySchemeType.ApiKey,
                Scheme = "Bearer"
            });
            
            options.AddSecurityRequirement(new OpenApiSecurityRequirement
            {
                {
                    new OpenApiSecurityScheme
                    {
                        Reference = new OpenApiReference
                        {
                            Type = ReferenceType.SecurityScheme,
                            Id = "Bearer"
                        }
                    },
                    Array.Empty<string>()
                }
            });
            
            // Custom Header
            options.AddSecurityDefinition("ApiKey", new OpenApiSecurityScheme
            {
                Description = "API Key authentication",
                Name = "X-API-Key",
                In = ParameterLocation.Header,
                Type = SecuritySchemeType.ApiKey
            });
        });
    }
    
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseSwagger();
            app.UseSwaggerUI(options =>
            {
                options.SwaggerEndpoint("/swagger/v1/swagger.json", "My API V1");
                options.RoutePrefix = "swagger"; // URL: /swagger
                options.DisplayRequestDuration();
                options.EnableDeepLinking();
                options.EnableFilter();
            });
        }
        
        app.UseRouting();
        app.UseAuthentication();
        app.UseAuthorization();
        
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
    }
}

// در .csproj برای XML Documentation
<PropertyGroup>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
  <NoWarn>$(NoWarn);1591</NoWarn>
</PropertyGroup>
```

---

## استفاده از Swagger Attributes

```csharp
/// <summary>
/// User management endpoints
/// </summary>
[ApiController]
[Route("api/[controller]")]
[Produces("application/json")]
public class UsersController : ControllerBase
{
    /// <summary>
    /// Get all users
    /// </summary>
    /// <param name="page">Page number</param>
    /// <param name="pageSize">Items per page</param>
    /// <returns>List of users</returns>
    /// <response code="200">Returns the list of users</response>
    /// <response code="400">If the parameters are invalid</response>
    /// <response code="401">If unauthorized</response>
    [HttpGet]
    [ProducesResponseType(typeof(PagedResult<User>), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ErrorResponse), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status401Unauthorized)]
    public async Task<ActionResult<PagedResult<User>>> GetAll(
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 10)
    {
        var users = await GetUsersAsync(page, pageSize);
        return Ok(users);
    }
    
    /// <summary>
    /// Get user by ID
    /// </summary>
    /// <param name="id">User ID</param>
    /// <returns>User details</returns>
    /// <response code="200">Returns the user</response>
    /// <response code="404">If user not found</response>
    [HttpGet("{id}")]
    [ProducesResponseType(typeof(User), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ErrorResponse), StatusCodes.Status404NotFound)]
    public async Task<ActionResult<User>> GetById(
        /// <summary>User unique identifier</summary>
        int id)
    {
        var user = await FindUserAsync(id);
        
        if (user == null)
            return NotFound();
        
        return Ok(user);
    }
    
    /// <summary>
    /// Create a new user
    /// </summary>
    /// <param name="dto">User creation data</param>
    /// <returns>Created user</returns>
    /// <response code="201">User created successfully</response>
    /// <response code="400">If validation fails</response>
    /// <response code="409">If email already exists</response>
    [HttpPost]
    [ProducesResponseType(typeof(User), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ErrorResponse), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(typeof(ErrorResponse), StatusCodes.Status409Conflict)]
    public async Task<ActionResult<User>> Create(
        /// <summary>User data</summary>
        [FromBody] CreateUserDto dto)
    {
        var user = await CreateUserAsync(dto);
        return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
    }
    
    /// <summary>
    /// Update user
    /// </summary>
    /// <param name="id">User ID</param>
    /// <param name="dto">Updated user data</param>
    /// <returns>No content</returns>
    [HttpPut("{id}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Update(int id, [FromBody] UpdateUserDto dto)
    {
        await UpdateUserAsync(id, dto);
        return NoContent();
    }
    
    /// <summary>
    /// Delete user (Admin only)
    /// </summary>
    /// <param name="id">User ID</param>
    /// <returns>No content</returns>
    [HttpDelete("{id}")]
    [Authorize(Roles = "Admin")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status401Unauthorized)]
    [ProducesResponseType(StatusCodes.Status403Forbidden)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(int id)
    {
        await DeleteUserAsync(id);
        return NoContent();
    }
}

/// <summary>
/// User creation DTO
/// </summary>
public class CreateUserDto
{
    /// <summary>
    /// User's full name
    /// </summary>
    /// <example>John Doe</example>
    [Required]
    public string Name { get; set; }
    
    /// <summary>
    /// User's email address
    /// </summary>
    /// <example>john@example.com</example>
    [Required]
    [EmailAddress]
    public string Email { get; set; }
    
    /// <summary>
    /// User's age
    /// </summary>
    /// <example>25</example>
    [Range(18, 120)]
    public int Age { get; set; }
}
```

---

# بخش 1️⃣8️⃣: Health Checks

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddHealthChecks()
        .AddCheck("self", () => HealthCheckResult.Healthy())
        .AddDbContextCheck<ApplicationDbContext>()
        .AddSqlServer(
            connectionString: Configuration.GetConnectionString("Default"),
            name: "sql-server",
            failureStatus: HealthStatus.Degraded)
        .AddRedis(
            redisConnectionString: Configuration.GetConnectionString("Redis"),
            name: "redis")
        .AddUrlGroup(
            new Uri("https://api.example.com"),
            name: "external-api",
            failureStatus: HealthStatus.Degraded);
}

public void Configure(IApplicationBuilder app)
{
    app.UseHealthChecks("/health", new HealthCheckOptions
    {
        ResponseWriter = async (context, report) =>
        {
            context.Response.ContentType = "application/json";
            
            var result = JsonSerializer.Serialize(new
            {
                status = report.Status.ToString(),
                checks = report.Entries.Select(e => new
                {
                    name = e.Key,
                    status = e.Value.Status.ToString(),
                    description = e.Value.Description,
                    duration = e.Value.Duration.TotalMilliseconds
                }),
                totalDuration = report.TotalDuration.TotalMilliseconds
            });
            
            await context.Response.WriteAsync(result);
        }
    });
    
    // Simple health check
    app.UseHealthChecks("/health/ready");
    app.UseHealthChecks("/health/live");
}

// Custom Health Check
public class CustomHealthCheck : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            // بررسی سلامت سیستم
            var isHealthy = CheckSystemHealth();
            
            if (isHealthy)
            {
                return Task.FromResult(
                    HealthCheckResult.Healthy("System is healthy"));
            }
            
            return Task.FromResult(
                HealthCheckResult.Degraded("System is degraded"));
        }
        catch (Exception ex)
        {
            return Task.FromResult(
                HealthCheckResult.Unhealthy("System is unhealthy", ex));
        }
    }
    
    private bool CheckSystemHealth()
    {
        // بررسی CPU, Memory, Disk, etc.
        return true;
    }
}

// Registration
services.AddHealthChecks()
    .AddCheck<CustomHealthCheck>("custom");
```

---

# بخش 1️⃣9️⃣: Logging

```csharp
// appsettings.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  }
}

// Controller با Logging
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly ILogger<UsersController> _logger;
    
    public UsersController(ILogger<UsersController> logger)
    {
        _logger = logger;
    }
    
    [HttpGet]
    public IActionResult GetAll()
    {
        _logger.LogInformation("Getting all users");
        
        try
        {
            var users = GetUsers();
            
            _logger.LogInformation("Retrieved {Count} users", users.Count);
            
            return Ok(users);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting users");
            throw;
        }
    }
    
    [HttpGet("{id}")]
    public IActionResult GetById(int id)
    {
        using (_logger.BeginScope("UserId:{UserId}", id))
        {
            _logger.LogDebug("Searching for user {UserId}", id);
            
            var user = FindUser(id);
            
            if (user == null)
            {
                _logger.LogWarning("User {UserId} not found", id);
                return NotFound();
            }
            
            _logger.LogInformation("Found user {UserId}: {UserName}", id, user.Name);
            return Ok(user);
        }
    }
}

// Serilog (بهترین کتابخانه Logging)
// dotnet add package Serilog.AspNetCore
// dotnet add package Serilog.Sinks.Console
// dotnet add package Serilog.Sinks.File

// Program.cs
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithThreadId()
    .WriteTo.Console()
    .WriteTo.File(
        "logs/log-.txt",
        rollingInterval: RollingInterval.Day,
        outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj}{NewLine}{Exception}")
    .CreateLogger();

builder.Host.UseSerilog();
```

---

# بخش 2️⃣0️⃣: Security Best Practices

## 1. HTTPS و HSTS
```csharp
// Program.cs
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
    app.UseHttpsRedirection();
}

// appsettings.json
{
  "Kestrel": {
    "Endpoints": {
      "Https": {
        "Url": "https://localhost:5001"
      }
    }
  }
}
```

## 2. CORS امن
```csharp
services.AddCors(options =>
{
    options.AddPolicy("Secure", builder =>
    {
        builder
            .WithOrigins("https://myapp.com") // مشخص
            .WithMethods("GET", "POST")        // محدود
            .WithHeaders("Content-Type", "Authorization")
            .AllowCredentials()
            .SetPreflightMaxAge(TimeSpan.FromHours(1));
    });
});
```

## 3. Cookie امن
```csharp
Response.Cookies.Append("auth_token", token, new CookieOptions
{
    HttpOnly = true,      // جلوگیری از XSS
    Secure = true,        // فقط HTTPS
    SameSite = SameSiteMode.Strict, // جلوگیری از CSRF
    MaxAge = TimeSpan.FromHours(1)
});
```

## 4. Content Security Policy
```csharp
app.Use(async (context, next) =>
{
    context.Response.Headers.Add("Content-Security-Policy", 
        "default-src 'self'; " +
        "script-src 'self' 'unsafe-inline'; " +
        "style-src 'self' 'unsafe-inline'; " +
        "img-src 'self' data: https:;");
    
    context.Response.Headers.Add("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Add("X-Frame-Options", "DENY");
    context.Response.Headers.Add("X-XSS-Protection", "1; mode=block");
    context.Response.Headers.Add("Referrer-Policy", "no-referrer");
    
    await next();
});
```

## 5. Rate Limiting
```csharp
// dotnet add package AspNetCoreRateLimit

services.AddMemoryCache();
services.Configure<IpRateLimitOptions>(options =>
{
    options.EnableEndpointRateLimiting = true;
    options.StackBlockedRequests = false;
    options.GeneralRules = new List<RateLimitRule>
    {
        new RateLimitRule
        {
            Endpoint = "*",
            Period = "1m",
            Limit = 100
        },
        new RateLimitRule
        {
            Endpoint = "*",
            Period = "1h",
            Limit = 1000
        }
    };
});

services.AddSingleton<IRateLimitConfiguration, RateLimitConfiguration>();
services.AddInMemoryRateLimiting();

app.UseIpRateLimiting();
```

## 6. API Key Authentication
```csharp
public class ApiKeyAuthMiddleware
{
    private readonly RequestDelegate _next;
    private const string API_KEY_HEADER = "X-API-Key";
    
    public async Task InvokeAsync(HttpContext context)
    {
        if (!context.Request.Headers.TryGetValue(API_KEY_HEADER, out var apiKey))
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsync("API Key is missing");
            return;
        }
        
        if (!IsValidApiKey(apiKey))
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsync("Invalid API Key");
            return;
        }
        
        await _next(context);
    }
    
    private bool IsValidApiKey(string apiKey)
    {
        // بررسی در دیتابیس یا کش
        var validKeys = new[] { "key1", "key2", "key3" };
        return validKeys.Contains(apiKey);
    }
}
```

---

# بخش 2️⃣1️⃣: Testing

## Unit Tests

```csharp
// dotnet add package xunit
// dotnet add package Moq
// dotnet add package Microsoft.AspNetCore.Mvc.Testing

public class UsersControllerTests
{
    private readonly Mock<IUserService> _mockUserService;
    private readonly UsersController _controller;
    
    public UsersControllerTests()
    {
        _mockUserService = new Mock<IUserService>();
        _controller = new UsersController(_mockUserService.Object);
    }
    
    [Fact]
    public async Task GetById_WithValidId_ReturnsOk()
    {
        // Arrange
        var userId = 1;
        var user = new User { Id = userId, Name = "Test User" };
        _mockUserService.Setup(s => s.GetByIdAsync(userId))
            .ReturnsAsync(user);
        
        // Act
        var result = await _controller.GetById(userId);
        
        // Assert
        var okResult = Assert.IsType<OkObjectResult>(result.Result);
        var returnedUser = Assert.IsType<User>(okResult.Value);
        Assert.Equal(userId, returnedUser.Id);
    }
    
    [Fact]
    public async Task GetById_WithInvalidId_ReturnsNotFound()
    {
        // Arrange
        var userId = 999;
        _mockUserService.Setup(s => s.GetByIdAsync(userId))
            .ReturnsAsync((User)null);
        
        // Act
        var result = await _controller.GetById(userId);
        
        // Assert
        Assert.IsType<NotFoundResult>(result.Result);
    }
    
    [Fact]
    public async Task Create_WithValidData_ReturnsCreated()
    {
        // Arrange
        var dto = new CreateUserDto { Name = "New User", Email = "new@test.com" };
        var user = new User { Id = 1, Name = dto.Name, Email = dto.Email };
        
        _mockUserService.Setup(s => s.CreateAsync(dto))
            .ReturnsAsync(user);
        
        // Act
        var result = await _controller.Create(dto);
        
        // Assert
        var createdResult = Assert.IsType<CreatedAtActionResult>(result.Result);
        Assert.Equal(nameof(_controller.GetById), createdResult.ActionName);
    }
}
```

## Integration Tests

```csharp
public class UsersApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    
    public UsersApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }
    
    [Fact]
    public async Task GetUsers_ReturnsSuccess()
    {
        // Act
        var response = await _client.GetAsync("/api/users");
        
        // Assert
        response.EnsureSuccessStatusCode();
        var content = await response.Content.ReadAsStringAsync();
        Assert.NotEmpty(content);
    }
    
    [Fact]
    public async Task CreateUser_WithValidData_ReturnsCreated()
    {
        // Arrange
        var user = new CreateUserDto 
        { 
            Name = "Test User", 
            Email = "test@test.com",
            Age = 25
        };
        
        var json = JsonSerializer.Serialize(user);
        var content = new StringContent(json, Encoding.UTF8, "application/json");
        
        // Act
        var response = await _client.PostAsync("/api/users", content);
        
        // Assert
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        Assert.NotNull(response.Headers.Location);
    }
}
```

---

# 📊 خلاصه جامع و جدول تصمیم‌گیری نهایی

## جدول انتخاب معماری

| نیاز شما | REST | GraphQL | gRPC | WebSocket | SSE |
|----------|------|---------|------|-----------|-----|
| CRUD ساده | ✅✅✅ | ✅ | ❌ | ❌ | ❌ |
| Query پیچیده | ✅ | ✅✅✅ | ❌ | ❌ | ❌ |
| Performance بالا | ✅✅ | ✅✅ | ✅✅✅ | ✅✅ | ✅ |
| Real-time دوطرفه | ❌ | ⚠️ | ✅✅ | ✅✅✅ | ❌ |
| Real-time یک‌طرفه | ❌ | ⚠️ | ✅✅ | ✅✅ | ✅✅✅ |
| Browser Support | ✅✅✅ | ✅✅✅ | ⚠️ | ✅✅✅ | ✅✅✅ |
| Mobile Support | ✅✅✅ | ✅✅✅ | ✅✅✅ | ✅✅ | ⚠️ |
| Learning Curve | آسان | متوسط | سخت | متوسط | آسان |
| Caching | ✅✅✅ | ⚠️ | ❌ | ❌ | ❌ |
| Cookie Support | ✅✅✅ | ✅✅✅ | ⚠️ | ⚠️ | ✅✅✅ |

---

## Checklist توسعه API

### ✅ Must Have
- [ ] HTTPS
- [ ] CORS صحیح
- [ ] Cookie امن (HttpOnly, Secure, SameSite)
- [ ] Authentication & Authorization
- [ ] Input Validation
- [ ] Error Handling
- [ ] Logging
- [ ] API Versioning
- [ ] Pagination
- [ ] Health Checks
- [ ] Rate Limiting

### ✅ Should Have
- [ ] Swagger/OpenAPI
- [ ] Response Caching
- [ ] Compression (Gzip, Brotli)
- [ ] Filtering & Sorting
- [ ] Unit Tests
- [ ] Integration Tests
- [ ] Monitoring

### ✅ Nice to Have
- [ ] Webhooks
- [ ] SSE برای notifications
- [ ] GraphQL برای query پیچیده
- [ ] Redis Cache
- [ ] Background Jobs
- [ ] CDN

---

## نکات طلایی 🌟

### Backend (ASP.NET Core)
1. **همیشه async/await استفاده کن**
2. **از Dependency Injection استفاده کن**
3. **Exception handling عمومی داشته باش**
4. **Log همه چیز رو (با Serilog)**
5. **از HttpContext فقط در Controller/Middleware استفاده کن**
6. **Cookie ها رو HttpOnly و Secure کن**
7. **CORS رو درست تنظیم کن (نه AllowAnyOrigin)**
8. **Rate Limiting داشته باش**
9. **Validation رو در چند لایه انجام بده**
10. **Health Check داشته باش**

### Frontend (Angular)
1. **از Signals استفاده کن (Angular 16+)**
2. **Interceptors برای authentication و error handling**
3. **Service ها Standalone باشن**
4. **withCredentials برای Cookie**
5. **Error handling مناسب**
6. **Loading states داشته باش**
7. **Retry logic برای درخواست‌های ناموفق**
8. **Debounce برای search**
9. **Pagination Client-Side**
10. **Form Validation با Reactive Forms**

---

## مثال کامل یک API حرفه‌ای

```csharp
// Program.cs - تنظیمات کامل
var builder = WebApplication.CreateBuilder(args);

// Services
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Database
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

// CORS
builder.Services.AddCors(options =>
{
    options.AddPolicy("Production", builder =>
        builder.WithOrigins("https://myapp.com")
               .AllowCredentials()
               .AllowAnyMethod()
               .AllowAnyHeader());
});

// Authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => { /* JWT config */ });

// Caching
builder.Services.AddResponseCaching();
builder.Services.AddMemoryCache();

// Compression
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<GzipCompressionProvider>();
    options.Providers.Add<BrotliCompressionProvider>();
});

// Health Checks
builder.Services.AddHealthChecks()
    .AddDbContextCheck<ApplicationDbContext>();

// API Versioning
builder.Services.AddApiVersioning();

// Rate Limiting
builder.Services.AddInMemoryRateLimiting();

// Logging
builder.Host.UseSerilog();

var app = builder.Build();

// Middleware Pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseCors("Production");
app.UseResponseCaching();
app.UseResponseCompression();
app.UseMiddleware<GlobalExceptionHandlerMiddleware>();
app.UseMiddleware<RequestLoggingMiddleware>();
app.UseIpRateLimiting();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();
app.MapHealthChecks("/health");

app.Run();
```

---

## ⚡ Performance Tips

### Backend
```csharp
// 1. Async all the way
public async Task<IActionResult> GetUsers()
{
    var users = await _context.Users.ToListAsync();
    return Ok(users);
}

// 2. Select فقط فیلدهای مورد نیاز
var users = await _context.Users
    .Select(u => new { u.Id, u.Name, u.Email })
    .ToListAsync();

// 3. Pagination همیشه
var users = await _context.Users
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();

// 4. AsNoTracking برای Read-Only
var users = await _context.Users
    .AsNoTracking()
    .ToListAsync();

// 5. Response Caching
[ResponseCache(Duration = 60)]
[HttpGet]
public IActionResult Get() { }
```

### Frontend
```typescript
// 1. Debounce برای Search
private searchSubject = new Subject<string>();

ngOnInit() {
  this.searchSubject.pipe(
    debounceTime(300),
    distinctUntilChanged()
  ).subscribe(term => this.search(term));
}

// 2. TrackBy برای *ngFor
@for (user of users(); track user.id) {
  <div>{{ user.name }}</div>
}

// 3. OnPush Change Detection
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})

// 4. Lazy Loading
const routes: Routes = [
  {
    path: 'users',
    loadComponent: () => import('./users/users.component')
  }
];

// 5. Virtual Scrolling برای لیست‌های بزرگ
<cdk-virtual-scroll-viewport>
  <div *cdkVirtualFor="let item of items">
    {{ item }}
  </div>
</cdk-virtual-scroll-viewport>
```

---

این جزوه **واقعا** کامل همه چیزی که نیاز داری رو پوشش می‌ده! 🚀

موارد پوشش داده شده:
✅ همه انواع معماری (REST, GraphQL, gRPC, SOAP, WebSocket, SSE)
✅ CORS عمیق
✅ Webhooks
✅ HttpContext و کلاس‌های ASP.NET Core
✅ Middleware ها
✅ Status Codes و Response Types
✅ API Versioning
✅ Pagination, Filtering, Sorting
✅ Validation (Data Annotations + FluentValidation)
✅ Exception Handling
✅ Swagger/OpenAPI
✅ Health Checks
✅ Logging
✅ Security Best Practices
✅ Testing
✅ Performance Tips
✅ مثال‌های کامل Angular 21
✅ Cookies در همه سناریوها
✅ Headers در هر نوع API

## بخش ۱: مقدمه به APIها و HTTP
- **API چی هست؟** Application Programming Interface – رابطی برای ارتباط نرم‌افزارها. مثلاً اپ فرانت‌اند با بک‌اند حرف می‌زنه. بیشتر APIهای وب بر پایه HTTP هستن.
- **HTTP چیه؟** پروتکل انتقال داده در وب. هر درخواست شامل:
- Method (GET, POST, etc.)
- URL (با query params مثل ?id=1)
- Headers (اطلاعات اضافی مثل Authorization)
- Body (داده اصلی، فقط در بعضی متدها)
- **RESTful API:** سبک محبوب با HTTP methods برای CRUD (Create: POST, Read: GET, Update: PUT/PATCH, Delete: DELETE). Stateless و scalable.
**نکته کلیدی:** GET body نداره (Idempotent – تکرار تغییری نمی‌ده). پارامترها در URL یا headers.
## بخش ۲: HTTP Methods و جزئیات
هر متد رفتار خاص داره. جدول زیر خلاصه می‌کنه (بله/نه یعنی معمولاً):
| Method | Body داره؟ | Header داره؟ | Cookie داره؟ | توضیح |
|--------|------------|--------------|--------------|-------|
| GET | نه | بله | بله | خواندن داده، query در URL. |
| POST | بله | بله | بله | ایجاد جدید، body برای data. |
| PUT | بله | بله | بله | آپدیت کامل. |
| PATCH | بله | بله | بله | آپدیت جزئی. |
| DELETE | معمولاً نه | بله | بله | حذف. |
| HEAD | نه | بله | بله | فقط headers پاسخ. |
| OPTIONS | نه | بله | بله | چک متدهای مجاز (برای CORS). |
| TRACE | نه | بله | بله | debug. |
## بخش ۳: پارامترها و Binding در ASP.NET Core
- انواع پارامتر: Query (در URL), Route (در path), Body (در بدنه), Header, Form.
- در ASP.NET Core: از attributes مثل [FromQuery], [FromBody]. مثال:
```csharp
[HttpGet("users/{id}")]
public IActionResult Get([FromRoute] int id, [FromQuery] string filter, [FromBody] FilterModel? body = null) { ... } // Body در GET ignore می‌شه
```
## بخش ۴: Postman و ابزارهای تست
Postman برای تست API: Headers (مثل Content-Length:0), Body (حالت‌ها: form-data, raw JSON), Params, Auth. مثال: برای POST، body JSON بذار و Content-Type: application/json.
## بخش ۵: WebAssembly (WASM) و SSR از نگاه فرانت‌اند
- **WASM:** کد باینری در مرورگر برای سرعت بالا (مثل Blazor در .NET). با API: از HttpClient برای کال، کوکی‌ها با withCredentials.
- **SSR:** رندر سمت سرور (مثل Next.js). با API: کال مستقیم بدون CORS مسئله. Hybrid: SSR + CSR.
- **از نگاه Angular (v21+):** HttpClient برای کال، interceptors برای auth/cookie. Signals برای reactive، hybrid rendering با Universal.
## بخش ۶: کوکی‌ها و مدیریت
کوکی‌ها: ذخیره در مرورگر، ارسال در headers. انواع: HttpOnly, Secure. در ASP.NET: Response.Cookies.Append. در فرانت: document.cookie.
## بخش ۷: CORS (Cross-Origin Resource Sharing)
تو گفتی CORS رو توضیح بدم، پس بخش جداگانه ساختم.
- **CORS چیه؟** مکانیزم امنیتی مرورگرها برای جلوگیری از درخواست‌های cross-origin (دامنه متفاوت). مثلاً اگر سایت example.com بخواد به api.other.com درخواست بده، بدون CORS اجازه نمی‌ده (برای جلوگیری از حملات مثل CSRF).
- **چطور کار می‌کنه؟**
- **Simple Requests:** (GET/HEAD/POST با headers استاندارد) – سرور باید header Access-Control-Allow-Origin: * (یا دامنه خاص) برگردونه.
- **Preflight Requests:** برای متدهای پیچیده (مثل PUT با custom headers)، مرورگر اول OPTIONS می‌فرسته تا چک کنه (Access-Control-Allow-Methods, Allow-Headers).
- **Headers کلیدی:**
- Access-Control-Allow-Origin: دامنه مجاز (یا *).
- Access-Control-Allow-Credentials: true برای cookie/auth.
- Access-Control-Allow-Methods: GET, POST, etc.
- Access-Control-Max-Age: زمان کش preflight.
- **در ASP.NET Core:** از middleware:
```csharp
app.UseCors(builder => builder.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader());
```
- نکته: * با credentials کار نمی‌کنه – دامنه خاص بذار.
- **در Angular/فرانت:** withCredentials: true در HttpClient برای cookie.
- **مشکلات رایج:** ۴۰۳ Forbidden اگر CORS ست نباشه. برای امنیت، origins رو محدود کن.
- **جایگزین‌ها:** JSONP (قدیمی، insecure)، Proxy (کال از سرور خودت).
## بخش ۸: WebSocket و ارتباطات Real-Time
تو گفتی راجب سوکت صحبت کن، پس عمیق‌تر توضیح می‌دم (قبلاً در انواع API مختصر بود).
- **WebSocket چیه؟** پروتکل full-duplex (دوطرفه همزمان) برای ارتباطات persistent real-time. از HTTP upgrade می‌شه (دست‌دهی اولیه با HTTP، بعد WebSocket).
- **تفاوت با HTTP:** HTTP request-responseه (polling نیاز به چک مداوم)، WebSocket connection باز نگه می‌داره – کم overhead برای real-time.
- **چطور کار می‌کنه؟**
۱. کلاینت درخواست upgrade می‌فرسته (header: Upgrade: websocket).
۲. سرور ۱۰۱ Switching Protocols برمی‌گردونه.
۳. حالا frameها (message, ping/pong برای alive) ردوبدل می‌شن.
- **انواع پیام:** Text (UTF-8), Binary (برای فایل/تصویر), Control (close, ping).
- **در ASP.NET Core:** از WebSockets middleware:
```csharp
if (context.WebSockets.IsWebSocketRequest) {
var webSocket = await context.WebSockets.AcceptWebSocketAsync();
await Echo(webSocket); // handle messages
}
```
- **در Angular:** از WebSocket API:
```typescript
const socket = new WebSocket('wss://example.com');
socket.onmessage = (event) => console.log(event.data);
socket.send('Hello');
```
- **مزایا:** Low latency، bidirectional، efficient برای chat/live data.
- **معایب:** Stateful (سخت scale)، fallback به polling اگر fail بشه.
- **Use Cases:** Chat apps (مثل WhatsApp web), live trading, collaborative editing (Google Docs).
- **امنیت:** همیشه wss:// (با TLS)، authenticate با token در handshake.
- **جایگزین‌ها:** SSE (یک‌طرفه push), Long Polling (HTTP اما باز نگه داشتن).
## بخش ۹: وب‌هوک‌ها (Webhooks)
- Push notifications event-driven. کلاینت URL ثبت می‌کنه، سرور POST می‌فرسته وقتی event رخ می‌ده.
- انواع: Simple, Signed, Retry-enabled.
- در ASP.NET/Angular: Controller برای receive, libraries برای send.
## بخش ۱۰: انواع APIها و Architectures
جدول انواع (با WebSocket و GraphQL):
| نوع | توضیح | پروتکل | Use Cases |
|-----|-------|---------|-----------|
| REST | Resource-based | HTTP | عمومی |
| SOAP | Message-based XML | HTTP/TCP | Enterprise |
| GraphQL | Query language | HTTP | Efficient fetching |
| gRPC | High-perf RPC | HTTP/2 | Microservices |
| WebSocket | Real-time bidirectional | WebSocket | Chat/live |
| SSE | Unidirectional push | HTTP | Updates |
| Webhooks | Event push | HTTP POST | Notifications |
| RPC | Function-like | مختلف | Internal |
- Hybrid: ترکیب REST + WebSocket.
## بخش ۱۱: امنیت APIها
- Auth: API Keys, OAuth, JWT.
- Authorization: RBAC/ABAC.
- تهدیدها: Injection, BOLA, Rate Limiting.
- CORS: بالا توضیح دادم.
- Gateway: برای central security.
## بخش ۱۲: بهترین Practices و مدیریت
- Design: Versioning, Pagination, Error Handling.
- Docs: OpenAPI.
- Testing: Postman.
- Monitoring: Prometheus.
- Lifecycle: Design to Retire.
## بخش ۱۳: روندهای پیشرفته API در ۲۰۲۶
- AI-Driven: APIs با generative models (مثل auto-query optimization).
- Edge Computing: APIs نزدیک کاربر برای latency کم.
- Serverless: FaaS با webhooks.
- Hyper-Personalization: GraphQL با user context.
- Challenges: Scalability, Privacy (GDPR compliance).
