---
name: reforger-wiki-rest-api
description: "Trigger: RestCallback, RestContext, GET request, POST request, HTTP header. RestApi scripting — context creation, GET/POST requests, callback handling, and lifetime rules."
disable-model-invocation: true
user-invocable: false
license: MIT
metadata:
  author: arga-reforger-team
  version: "1.1.0"
  triggers:
    - "RestCallback"
    - "RestContext"
    - "GET request"
    - "POST request"
    - "HTTP header"
---

## Activation Contract

Load this skill when writing Enforce Script code that communicates with an external web API via `RestApi`, `RestContext`, or `RestCallback`.

## Hard Rules

**CORRECTED against Doxygen (`_rest_context_8c_source.html`, `_rest_callback_8c_source.html`)** — several claims below were wrong in the previous version of this skill:

**Scope and limitations**
- `RestContext` actually exposes `GET`, `POST`, `PUT`, and `DELETE` (plus obsolete `FILE`/`FILE_now` marked `[Obsolete("Not supported, will be removed!")]`). PUT and DELETE ARE available — the previous version of this skill said only GET/POST existed, which was wrong.
- Custom HTTP headers ARE supported via `context.SetHeaders(string definition)` — the previous version of this skill said headers were not supported, which was wrong.
- `context.SetTimeout(int timeoutS)` sets the request timeout.
- Accepted data size is below 1 MB. Larger responses will fail.
- REST requests are asynchronous — they do NOT block the main game thread.

**Context**
- A `RestContext` represents a base URL. Create it once and reuse it for multiple requests to the same host.
- Obtain a context via `GetGame().GetRestApi().GetContext("https://example.com/")`.
- The URL passed to `GetContext` is the root; append the endpoint path in the `GET`/`POST`/`PUT`/`DELETE` call.

**Callback — use the CURRENT API, not the obsolete one**
- `RestCallback` (real class) exposes `SetOnSuccess(RestCallbackFunc onSuccess)` and `SetOnError(RestCallbackFunc onError)` to register plain function callbacks, plus `GetRestResult()` (`ERestResult`), `GetHttpCode()` (`HttpCode`), and `GetData()` (`string`) to read the result after the callback fires.
- The override-based pattern `override void OnSuccess(string data, int dataSize)` / `OnError(int errorCode)` / `OnTimeout()` still compiles but each one is marked `[Obsolete(...)]` in the engine source, explicitly pointing to `SetOnSuccess()`/`SetOnError()` instead — do not use them in new code.
- The scripter is responsible for callback object lifetime. Hold the callback in a `ref` member variable for the duration of the request — if it gets garbage-collected the callback will fire on a null reference.
- `GetData()` may return a large string; guard `Print()` calls if you don't want to flood the log.

**Making requests**
- `context.GET(callback, "endpoint")` — sends HTTP GET to `<base>/<endpoint>`. `GET_now(request)` is a synchronous variant returning `string` directly (blocks — use sparingly).
- `context.GET(callback, "endpoint?param=value")` — query string is appended directly to the endpoint string.
- `context.POST(callback, "endpoint", body)`, `context.PUT(callback, "endpoint", body)`, `context.DELETE(callback, "endpoint", body)` — all take the same `(callback, request, data)` shape; each has a blocking `_now` variant too.

**Error handling**
- Error codes are processed asynchronously. Never assume a request completed synchronously.
- Read the outcome via `callback.GetRestResult()` (`ERestResult`) and `callback.GetHttpCode()` (`HttpCode`) after `SetOnSuccess`/`SetOnError` fires — not via the obsolete `errorCode` int parameter.

## Key APIs / Patterns

```enforce
// Callback class — current (non-obsolete) API
class MyRestCallback : RestCallback
{
    void MyRestCallback()
    {
        SetOnSuccess(OnRequestSuccess);
        SetOnError(OnRequestError);
    }

    // RestCallbackFunc signature is void(RestCallback cb = null) — verified in OnlineTypes.c
    void OnRequestSuccess(RestCallback cb = null)
    {
        PrintFormat("Success: HTTP %1", GetHttpCode());
        string data = GetData();
        if (!data.IsEmpty())
            Print(data);
    }

    void OnRequestError(RestCallback cb = null)
    {
        PrintFormat("Error: result=%1 http=%2", GetRestResult(), GetHttpCode());
    }
}

// Holder class — keeps callback alive
class ApiClient
{
    protected ref MyRestCallback m_Callback;

    void SendRequest()
    {
        if (!m_Callback)
            m_Callback = new MyRestCallback();

        RestContext ctx = GetGame().GetRestApi().GetContext("https://httpbin.org/");
        ctx.SetHeaders("Content-Type: application/json");

        // Simple GET
        ctx.GET(m_Callback, "get");

        // GET with query string
        ctx.GET(m_Callback, "get?x=10&y=5");

        // PUT / DELETE are real methods too
        ctx.PUT(m_Callback, "put", "{}");
        ctx.DELETE(m_Callback, "delete", "");
    }
}
```

**Calling from game code**
```enforce
class MyComponent : ScriptComponent
{
    protected ref ApiClient m_ApiClient;

    override void EOnPostInit(IEntity owner)
    {
        m_ApiClient = new ApiClient();
        m_ApiClient.SendRequest();
    }
}
```

## References

- PDF: `REST API Usage – Arma Reforger - Bohemia Interactive Community.pdf`
- Doxygen (source of truth for this skill's corrections): `_rest_context_8c_source.html` (`RestContext` real API: GET/GET_now/POST/POST_now/PUT/PUT_now/DELETE/DELETE_now/SetHeaders/SetTimeout, plus obsolete FILE/FILE_now/reset), `_rest_callback_8c_source.html` (`RestCallback` real API: SetOnSuccess/SetOnError/GetRestResult/GetHttpCode/GetData, plus obsolete OnSuccess/OnError/OnTimeout), `_online_types_8c_source.html` (`RestCallbackFunc` signature).
- Wiki: `https://community.bistudio.com/wiki/Arma_Reforger:REST_API_Usage`
- See also: `reforger-wiki-json-api-struct` (JsonApiStruct as REST callback payload handler)
