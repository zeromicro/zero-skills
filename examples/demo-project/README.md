# Demo Project - GitHub Copilot with go-zero

This demo project demonstrates how to use GitHub Copilot + ai-context to develop go-zero applications.

## Quick Start

### Prerequisites

- Go 1.23+
- Git
- VS Code with GitHub Copilot extension
- goctl (will be installed automatically)

### Setup Demo Project

```bash
# Run setup script
cd /Users/kevin/Develop/go/zero-skills/examples/demo-project
./setup-demo.sh
```

The script will automatically:
1. ✅ Check and install dependencies (Go, goctl)
2. ✅ Create demo workspace directory
3. ✅ Configure GitHub Copilot (add ai-context submodule)
4. ✅ Generate go-zero API project using goctl
5. ✅ Create sample API definition (user management)
6. ✅ Generate complete project structure

### Verify Configuration

```bash
cd demo-workspace
./verify-copilot.sh
```

You should see:
```
✓ ai-context submodule exists
✓ copilot-instructions.md symlink exists
✓ Configuration file contains go-zero content
✓ go-zero project structure is correct
✓ All checks passed!
```

## Project Structure

```
demo-workspace/
├── .github/
│   ├── ai-context/              # ai-context submodule
│   └── copilot-instructions.md  # -> ai-context/00-instructions.md
├── userapidemo/
│   ├── etc/
│   │   └── user-api.yaml        # Configuration file
│   ├── internal/
│   │   ├── config/              # Config definitions
│   │   ├── handler/             # HTTP handler layer
│   │   │   ├── createuserhandler.go
│   │   │   ├── getuserhandler.go
│   │   │   └── listusershandler.go
│   │   ├── logic/               # Business logic layer
│   │   │   ├── createuserlogic.go
│   │   │   ├── getuserlogic.go
│   │   │   └── listuserslogic.go
│   │   ├── svc/                 # Service context
│   │   │   └── servicecontext.go
│   │   └── types/               # Type definitions
│   │       └── types.go
│   ├── user.api                 # API definition file
│   ├── userapidemo.go           # Main entry point
│   └── README.md                # Project documentation
└── verify-copilot.sh            # Verification script
```

## Testing GitHub Copilot

### Test Scenario 1: Implement CreateUser Business Logic

1. Open the project in VS Code:
   ```bash
   cd demo-workspace/userapidemo
   code .
   ```

2. Open file: `internal/logic/createuserlogic.go`

3. In the `CreateUser` method, try typing the following comment:
   ```go
   // Validate username is not empty
   ```

4. **Expected behavior**:
   - ✅ Copilot suggests go-zero error handling patterns
   - ✅ Uses `errorx` or standard error returns
   - ✅ Follows Logic layer responsibilities
   - ✅ Correctly uses `req` and `types` definitions

5. **Example implementation** (Copilot might suggest):
   ```go
   func (l *CreateUserLogic) CreateUser(req *types.CreateUserRequest) (*types.CreateUserResponse, error) {
       // Validate username is not empty
       if req.Username == "" {
           return nil, errors.New("username is required")
       }

       // Validate email format
       if req.Email == "" {
           return nil, errors.New("email is required")
       }

       // TODO: Save to database
       user := types.User{
           Id:       1,
           Username: req.Username,
           Email:    req.Email,
           CreateAt: time.Now().Format("2006-01-02 15:04:05"),
       }

       return &types.CreateUserResponse{
           User: user,
       }, nil
   }
   ```

### Test Scenario 2: Add Middleware

1. Create new file: `internal/middleware/auth.go`

2. Type:
   ```go
   package middleware

   import "net/http"

   // JWT authentication middleware
   ```

3. **Expected behavior**:
   - ✅ Copilot suggests go-zero middleware pattern
   - ✅ Returns `func(http.HandlerFunc) http.HandlerFunc`
   - ✅ Proper error response handling
   - ✅ Uses `httpx` utilities

### Test Scenario 3: Add Database Operations

1. Create comment:
   ```go
   // TODO: Add MySQL connection and user table operations
   ```

2. **Expected behavior**:
   - ✅ Copilot suggests using `sqlx` or `go-zero/core/stores/sqlx`
   - ✅ Suggests using `goctl model` for code generation
   - ✅ Provides correct database configuration patterns

## Comparison Test

### Copilot WITH ai-context

```go
// Input: Implement user creation
// Copilot suggests:
func (l *CreateUserLogic) CreateUser(req *types.CreateUserRequest) (*types.CreateUserResponse, error) {
    // ✅ Correct parameter validation
    // ✅ Uses go-zero error handling
    // ✅ Return type matches types definitions
    // ✅ Follows business logic in Logic layer principle
}
```

### Copilot WITHOUT ai-context

```go
// Input: Implement user creation
// Copilot might suggest:
func CreateUser(w http.ResponseWriter, r *http.Request) {
    // ❌ Business logic directly in handler
    // ❌ Uses generic HTTP patterns, not go-zero compliant
    // ❌ Error handling might use http.Error
}
```

## Verification

### 1. Check Copilot Configuration

```bash
# Confirm Copilot loaded the configuration
cat demo-workspace/.github/copilot-instructions.md | head -20

# Should see go-zero related instructions
```

### 2. Test Code Suggestion Quality

When implementing business logic, observe Copilot suggestions:
- Does it follow three-layer architecture?
- Does it use correct error handling?
- Does it understand go-zero utilities?

### 3. Run the Project

```bash
cd demo-workspace/userapidemo

# Run the service
go run userapidemo.go -f etc/user-api.yaml

# Test API (in another terminal)
curl http://localhost:8888/api/users
```

## Common Issues

### Q: Copilot not using go-zero patterns?

**A:** Check:
1. Is VS Code opened in the correct project directory?
2. Does `.github/copilot-instructions.md` file exist?
3. Restart VS Code to reload Copilot configuration

### Q: Symlinks don't work on Windows?

**A:** On Windows, run with administrator privileges:
```cmd
mklink .github\copilot-instructions.md .github\ai-context\00-instructions.md
```

Or copy the file directly:
```cmd
copy .github\ai-context\00-instructions.md .github\copilot-instructions.md
```

### Q: goctl command not found?

**A:** Install goctl:
```bash
go install github.com/zeromicro/go-zero/tools/goctl@latest
```

## Cleanup

Delete the demo project:
```bash
rm -rf demo-workspace
```

## More Resources

- [ai-context](https://github.com/zeromicro/ai-context) - GitHub Copilot instructions
- [zero-skills](https://github.com/zeromicro/zero-skills) - go-zero knowledge base
- [go-zero Documentation](https://go-zero.dev)
- [Claude Code Guide](../../getting-started/claude-code-guide.md)
