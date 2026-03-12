Install Golang
```bash
curl -sS https://webi.sh/golang | sh
source ~/.config/envman/PATH.env
```

Variables
```go
// 1. VARIABLE DECLARATION
// -----------------------
var sadNum int = 42    // Explicit declaration 
happyNum := 42         // Short declaration - the GOAT

// 2. DATA TYPES
// -------------
// Use defaults unless performance/memory is an issue.
bool               // true/false
string             // UTF-8 text
int                // Integer (platform-dependent size)
uint               // Unsigned integer (non-negative)
byte               // uint8 alias (8-bit integer)
rune               // int32 alias (Unicode code point/emoji)
float64            // Decimal (64-bit precision)
complex128         // Complex numbers (real + imaginary)

// Sized Types (Memory specific):
// int8, int16, int32, int64
// uint8, uint16, uint32, uint64, uintptr
// float32, float64
// complex64, complex128

// 3. FORMATTED STRINGS (fmt)
// --------------------------
// Functions:
fmt.Printf()       // Prints formatted string to os.Stdout
fmt.Sprintf()      // Returns formatted string as a variable

// Formatting Verbs:
// %v : The "Catch-all" (Default representation)
// %s : String
// %d : Integer (Decimal)
// %t : Boolean
// %f : Float (Use %.2f to round to 2 decimals)
// %T : Type of the value

// 4. PRACTICAL EXAMPLE
// --------------------
name := "Gemini"
openRate := 98.765

// Interpolating multiple types into one string
msg := fmt.Sprintf("Hi %s, your open rate is %.1f percent\n", name, openRate)
fmt.Print(msg) // Output: Hi Gemini, your open rate is 98.8 percent
```

Conditionals
```go
// 1. BASIC IF / ELSE IF / ELSE
// ----------------------------
height := 5

if height > 6 {
    fmt.Println("You are super tall!")
} else if height > 4 {
    fmt.Println("You are tall enough!")
} else {
    fmt.Println("You are not tall enough!")
}

// 2. IF WITH INITIAL STATEMENT
// ----------------------------
// Syntax: if INITIAL_STATEMENT; CONDITION { }
// Variables declared here are scoped strictly to the if/else blocks.

// Instead of declaring beforehand:
// length := len(email)
// if length < 10 { ... }

// Do this (The "Go Way"):
if length := len(email); length < 10 {
    fmt.Printf("Email must be at least 10 chars, is %d\n", length)
} 
// 'length' no longer exists in memory after this block!

// 3. SWITCH STATEMENTS
// --------------------
// Cleaner alternative to long if-else chains for exact matches.
// NOTE: 'break' is implicit in Go! It stops automatically after a match.

func getCreator(os string) string {
    var creator string
    
    switch os {
    case "linux":
        creator = "Linus Torvalds"
    case "windows":
        creator = "Bill Gates"
    case "mac":
        creator = "A Steve"
    default:
        creator = "Unknown"
    }
    
    return creator
}
```

Functions
```go
// ==========================================
// 1. BASIC SYNTAX & PARAMETERS
// ==========================================

// Basic signature: func name(params) returnType
func calculateOffset(baseAddress int, offset int) int {
	return baseAddress + offset
}

// Condensed Parameters:
// If consecutive parameters share a type, only declare it at the very end.
func configPayload(size, threadCount int, target string, active bool) {
	// size and threadCount are both ints
}

// ==========================================
// 2. RETURN VALUES
// ==========================================

// Multiple Returns: Very common in Go, heavily used for error handling.
func scanTarget() (string, error) {
	return "192.168.1.5", nil
}

// Ignoring Returns: Use the blank identifier (_) if you don't need a value.
// Go will not compile if you declare a variable but don't use it.
func main() {
	ip, _ := scanTarget() // We only care about the IP, so we toss the error
}

// Named Return Values & "Naked" Returns:
// You can name what you are returning directly in the function signature.
func getAdminCredentials() (username, password string) {
	// username and password are automatically initialized as "" (zero values)
	username = "admin"
	password = "supersecretpassword"

	// "Naked Return": Automatically returns the named variables above.
	// Best practice: Only use in very small functions, otherwise it hurts readability.
	return
}

// Explicit Returns (and Overwriting Named Returns):
// Even if you named the returns, you can still return values explicitly.
func getKeys() (pubKey, privKey string) {
	return "public_rsa", "private_rsa"   // be careful!
}

// ==========================================
// 3. CONTROL FLOW & VARIABLE PASSING
// ==========================================

// Pass by Value: Variables are passed as COPIES by default.
func incrementAttempts(attempts int) {
	attempts++ // This modifies the local copy, NOT the original variable!
}

// Early Returns (Guard Clauses):
// Avoid deep if/else nesting. Check for errors/conditions early and return immediately.
func exploitVulnerability(isOpen bool) (bool, error) {
	if !isOpen {
		return false, errors.New("target port is closed") // Early return out of the function
	}
	// Proceed with the rest of the complex logic here
	return true, nil
}

// Defer:
// Postpones execution of a statement until the surrounding function hits a return.
// This is heavily used for cleanup (closing sockets, files, database connections).
func connectToC2() {
	// defer connection.Close()
	// No matter how many return statements exist below this line,
	// connection.Close() is guaranteed to run at the very end.
}

// ==========================================
// 4. ADVANCED FUNCTIONS (First-Class Citizens)
// ==========================================

// Functions as Values: You can pass a function into another function as an argument.
func applyEvasion(payload string, evasionTech func(string) string) string {
	return evasionTech(payload) // Executes the function that was passed in
}

// Anonymous Functions:
// Functions without a name, usually defined inline for one-off uses.
func runOneOff() {
	// Passing an anonymous function directly into applyEvasion
	obfuscated := applyEvasion("shellcode", func(p string) string {
		return "obfuscated_" + p
	})
}

// ==========================================
// 5. SCOPE, CLOSURES & CURRYING
// ==========================================

// Block Scope: 
// Variables declared inside { } brackets only exist inside those brackets.

// Closures:
// A function that "traps" or references variables from outside its own body.
func sessionGenerator() func() int {
	sessionID := 0 // The inner function "remembers" this variable forever
	
	return func() int {
		sessionID++
		return sessionID
	}
}

// Currying (Partial Application):
// Breaking down a multi-argument function into a chain of single-argument functions.
// Useful for locking in a specific configuration early on.
func baseEncoder(algorithm string) func(string) string {
	// We lock in the "algorithm" parameter now...
	return func(data string) string {
		// ...and use it later when the returned inner function is actually called with "data"
		return algorithm + "_encoded_" + data
	}
}
```

Structs
```go
// Structs represent structured data
type car struct {
	brand      string
	mileage    int
}

// Nested Structs
type car struct {
	frontWheel  wheel   // we declare frontWheel belongs wo wheel structs
}

type wheel struct {
	radius   int
}

myCar := car{}                  // assigning myCar as a car struct
myCar.frontWheel.radius = 5     // . operator to access struct fields

// Anonymous Structs -- In general, prefer named structs
myCar := struct {
  brand string
  model string
} {
  brand: "Toyota",
  model: "Camry",
}

// Embedded Structs - A KIND OF data-only inheritance
type car struct {
  brand string
  model string
}

type truck struct {
  // "car" is embedded, so the definition of a
  // "truck" now also additionally contains all
  // of the fields of the car struct
  car
  bedSize int
}
```