# Swagger / OpenAPI Document Generation

## Overview

goctl (≥ 1.8.2) has a **built-in** `api swagger` subcommand that generates OpenAPI 2.0 (Swagger) JSON/YAML directly from `.api` files. No external plugin required.

> **Important**: For goctl < 1.8.2, the older `goctl-swagger` plugin approach was needed. Since 1.8.2+, always use the built-in command.

## Command Reference

```bash
goctl api swagger [flags]

Flags:
  --api string        .api source file
  --dir string        output directory
  --filename string   output filename (without extension)
  --yaml              generate YAML format instead of JSON
  -h, --help          help for swagger
```

### ✅ Common Usage

```bash
# Generate JSON (default)
goctl api swagger --api api/entry.api --dir docs/swagger --filename aipivot

# Generate YAML
goctl api swagger --api api/entry.api --dir docs/swagger --filename aipivot --yaml
```

### Makefile Integration

```makefile
swagger:
	@mkdir -p docs/swagger
	goctl api swagger --api api/entry.api --dir docs/swagger --filename aipivot
	@echo "Swagger doc generated: docs/swagger/aipivot.json"
```

## `.api` File Swagger Annotations

All swagger metadata is driven by the `.api` file. The following sections document every annotation recognized by `goctl api swagger`.

### info Block — Document Metadata

The `info` block in the entry `.api` file maps to Swagger's top-level fields:

```go
syntax = "v1"

info (
    // Basic info → swagger info object
    title:       "My API"                    // swagger title
    description: "API description..."        // swagger description
    version:     "1.0.0"                     // swagger version

    // Contact info
    contactName:  "Team"                     // info.contact.name
    contactURL:   "https://example.com"      // info.contact.url
    contactEmail: "team@example.com"         // info.contact.email

    // Terms & License
    termsOfService: "https://example.com/tos"
    licenseName:    "MIT"
    licenseURL:     "https://github.com/repo/blob/master/LICENSE"

    // Protocol & Host → swagger top-level
    schemes:  "http,https"                   // comma-separated: http/https/ws/wss
    host:     "localhost:8080"               // API host (no protocol prefix)
    basePath: "/api/v1"                      // base path for all routes

    // Content types
    consumes: "application/json"             // default request content type
    produces: "application/json"             // default response content type

    // Swagger generation options
    useDefinitions: true                     // use $ref definitions for types
    wrapCodeMsg:    true                     // wrap responses in code-msg format
)
```

#### `useDefinitions`

When `true`, response/request types are generated as `$ref` references to a shared `definitions` section, avoiding inline duplication:

```json
{
  "schema": { "$ref": "#/definitions/LoginResponse" }
}
```

#### `wrapCodeMsg`

When `true`, all responses are wrapped in a unified code-msg envelope:

```json
{
  "code": 0,
  "msg": "OK",
  "data": { ... }
}
```

> Note: Boolean syntax (`true`/`false`) requires goctl ≥ 1.8.4. For older versions use string form: `wrapCodeMsg: "true"`.

### @server Block — Route Group Configuration

```go
@server (
    group:    user              // handler/logic directory grouping
    prefix:   /api/v1           // URL prefix for all routes in this block
    tags:     "User Management"  // swagger tags grouping (shown in Swagger UI sidebar)
    authType: apiKey            // security scheme reference (see below)
)
```

- **`tags`**: Maps to Swagger `tags` array on each operation. Routes in the same `tags` appear under the same group in Swagger UI. Takes priority over `summary` for grouping.

### @doc — Operation Summary

```go
service my-api {
    @doc "Create user - register a new user account"
    @handler CreateUserHandler
    post /users (CreateUserRequest) returns (CreateUserResponse)
}
```

- The `@doc` string maps to `summary` on the Swagger operation.
- For multi-line / structured doc:

```go
@doc (
    summary:     "Create user"
    description: "Register a new user account with email verification"
    bizCodeEnumDescription: "1003-User not found<br>1004-Invalid operation"
)
```

### Field Tags — Property Metadata

go-zero field tags are recognized and mapped to Swagger property attributes:

| Tag | Swagger Effect | Example |
|-----|---------------|---------|
| `example=...` | `example` value in schema | `json:"name,example=John"` |
| `optional` | Field not in `required` array | `json:"bio,optional"` |
| `default=...` | `default` value | `json:"role,default=member"` |
| `options=a\|b\|c` | `enum: [a, b, c]` | `json:"role,options=admin\|member"` |
| `range=[min:max]` | `minimum`/`maximum` | `json:"age,range=[1:120]"` |

#### ✅ Complete Field Example

```go
type CreateUserRequest {
    Name     string `json:"name,example=John"`                             // user name
    Email    string `json:"email,example=user@example.com"`                // email address
    Age      int    `json:"age,range=[1:120],example=25"`                  // age
    Role     string `json:"role,default=member,options=admin|member,example=member"` // role
    Bio      string `json:"bio,optional"`                                  // bio (optional)
    Language string `json:"language,options=go|java|python|rust"`          // programming language
}
```

### Path Parameters

Use `path` tag for URL path variables:

```go
type GetUserRequest {
    Id int64 `path:"id"` // auto-converted to {id} in swagger
}

service my-api {
    @handler GetUser
    get /users/:id (GetUserRequest) returns (GetUserResponse)
}
```

### Query Parameters

Use `form` tag for query string parameters:

```go
type ListUsersRequest {
    Page     int    `form:"page,default=1,range=[1:]"`
    PageSize int    `form:"page_size,default=10,range=[1:100]"`
    Keyword  string `form:"keyword,optional"`
}
```

### Security Definitions

Define authentication schemes via JSON in `info`, then reference them with `authType` in `@server`:

```go
info (
    securityDefinitionsFromJson: `{
        "apiKey": {
            "type": "apiKey",
            "name": "Authorization",
            "in": "header"
        },
        "oauth2": {
            "type": "oauth2",
            "authorizationUrl": "https://example.com/oauth/authorize",
            "flow": "implicit",
            "scopes": {
                "read": "read access",
                "write": "write access"
            }
        }
    }`
)

@server (
    authType: apiKey    // all routes in this block use apiKey auth
)
service my-api {
    @handler GetProfile
    get /profile returns (ProfileResponse)
}
```

### Business Error Codes

When `wrapCodeMsg: true`, you can document business error codes:

```go
info (
    wrapCodeMsg: true
    // Global error code descriptions
    bizCodeEnumDescription: "1001-Not logged in<br>1002-No permission"
)

service my-api {
    @doc (
        summary: "Login"
        // Per-endpoint error codes (overrides global)
        bizCodeEnumDescription: "1003-User not found<br>1004-Wrong password"
    )
    @handler Login
    post /login (LoginReq) returns (LoginResp)
}
```

> Note: Per-endpoint `bizCodeEnumDescription` does not work when `useDefinitions: true`, because shared definitions cannot have per-endpoint error variants.

## Complex Type Support

goctl swagger supports:

- Basic types and their arrays/maps
- Nested objects and pointer objects
- Multi-level nesting
- Array of arrays, map of maps

```go
type Level2 {}

type Level1 {
    Value  int     `json:"value,example=1"`
    Child  Level2  `json:"child"`
    Ptr    *Level2 `json:"ptr"`
}

type ComplexReq {
    Matrix     [][]int                         `json:"matrix"`
    NestedMap  map[string]map[string]Level1    `json:"nestedMap"`
    PtrArray   []*Level1                        `json:"ptrArray"`
}
```

## Complete Example

### Multi-file `.api` Structure

```
api/
├── entry.api    # main entry with info + imports
├── comm.api     # shared types
├── auth.api     # auth module types + routes
└── infra.api    # infra module types + routes
```

### entry.api (Main Entry)

```go
syntax = "v1"

info (
    title:          "My Service API"
    description:    "Example multi-module API with swagger annotations"
    version:        "1.0.0"
    schemes:        "http,https"
    host:           "localhost:8080"
    basePath:       "/"
    consumes:       "application/json"
    produces:       "application/json"
    useDefinitions: true
)

import (
    "comm.api"
    "auth.api"
    "infra.api"
)
```

### auth.api (Module with Tags + Examples)

```go
syntax = "v1"

info (
    title:   "Auth API"
    desc:    "Authentication endpoints"
    version: "1.0.0"
)

type (
    // LoginRequest user login payload
    LoginRequest {
        Email    string `json:"email,example=user@example.com"`    // login email
        Password string `json:"password,example=P@ssw0rd123"`     // login password
    }

    // LoginResponse login result
    LoginResponse {
        Code int32     `json:"code,example=0"`   // biz status code, 0 means success
        Msg  string    `json:"msg,example=OK"`   // response message
        Data LoginData `json:"data"`              // login data
    }

    // LoginData login success payload
    LoginData {
        Token    string `json:"token,example=eyJhbGciOi..."`        // JWT access token
        ExpireAt int64  `json:"expireAt,example=1715386400000"`     // token expiry (unix ms)
    }
)

@server (
    group:  auth
    prefix: /api/v1/auth
    tags:   "Authentication"
)
service my-api {
    @doc "User login - authenticate with email and password, returns JWT token"
    @handler LoginHandler
    post /login (LoginRequest) returns (LoginResponse)
}
```

## Best Practices

### ✅ DO

- Put swagger `info` metadata in the **entry** `.api` file (the one passed to `--api`)
- Use `tags` in `@server` to logically group endpoints in Swagger UI
- Add `example` values to every field for meaningful Swagger "Try it out" experience
- Use `options` tag for enum fields — generates `enum` constraints in schema
- Use `useDefinitions: true` to avoid type duplication in generated JSON
- Add `@doc` summary to every endpoint
- Use `//` comments after fields — they become `description` in swagger properties

### ❌ DON'T

- Use the old `goctl-swagger` plugin with goctl ≥ 1.8.2 (it's now built-in)
- Put global swagger metadata in sub-module `.api` files (only `entry.api` info is used)
- Mix `useDefinitions: true` with per-endpoint `bizCodeEnumDescription` (they conflict)
- Forget to add `example` tags — Swagger UI without examples is hard to use
- Hard-code host/basePath — consider environment-specific overrides

## Troubleshooting

### "swagger" subcommand not found

Your goctl version is < 1.8.2. Upgrade:

```bash
go install github.com/zeromicro/go-zero/tools/goctl@latest
goctl --version  # should be >= 1.8.2
```

### Comments not appearing in swagger

- Field comments must be `//` style on the same line or above the field
- Type comments must be `// TypeName description` directly above the type definition
- The comment text maps to `description` in the swagger property

### Boolean values not working in info block

Boolean syntax (`true`/`false` without quotes) requires goctl ≥ 1.8.4. For older versions:

```go
info (
    useDefinitions: "true"   // string form for goctl < 1.8.4
    wrapCodeMsg:    "true"
)
```
