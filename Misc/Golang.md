Install Golang
```bash
curl -sS https://webi.sh/golang | sh
source ~/.config/envman/PATH.env
```
#### Variables
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
#### Conditionals
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
#### Functions
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
#### Structs
```go
// --- GO STRUCTS REFERENCE ---

// 1. BASIC STRUCTS
// Represent structured data with named fields.
type Car struct {
	Brand   string
	Mileage int
}

// 2. NESTED STRUCTS
// Structs that contain other structs as fields.
type Wheel struct {
	Radius int
}
type NestedCar struct {
	FrontWheel Wheel // FrontWheel is of type Wheel
}

/* Usage:
	myCar := NestedCar{}
	myCar.FrontWheel.Radius = 5 // Use the dot operator to access fields
*/

// 3. ANONYMOUS STRUCTS
// Declared and instantiated simultaneously.
// Note: In general, prefer named structs for reusability.
/* Usage:
	myCar := struct {
		Brand string
		Model string
	}{
		Brand: "Toyota",
		Model: "Camry",
	}
*/

// 4. EMBEDDED STRUCTS
// Go's version of data-only inheritance (composition).
type BaseCar struct {
	Brand string
	Model string
}

type Truck struct {
	BaseCar         // Embedded struct: Truck inherits all fields of BaseCar
	BedSize int
}

/* Usage:
	t := Truck{}
	t.Brand = "Ford" // Fields from BaseCar are promoted and accessible directly
*/

// 5. STRUCT METHODS
// Functions that attach to a specific type via a "receiver".

// Example A: Simple value receiver
type Rect struct {
	Width  int
	Height int
}

// 'r' is the receiver. It receives a copy of the Rect struct.
func (r Rect) Area() int {        
	return r.Width * r.Height          
}

/* Usage:
	r := Rect{Width: 5, Height: 10}
	fmt.Println(r.Area())
*/

// Example B: Method with arguments and multiple return values
type User struct {
	MessageCharLimit int
}

func (u User) SendMessage(message string, messageLength int) (string, bool) {
	if messageLength <= u.MessageCharLimit {
		return message, true
	}
	return "", false
}

// 6. MEMORY LAYOUT
// Go allocates structs in contiguous blocks of memory.
// Field ordering matters! The compiler adds "padding" to align memory.
// Best practice: Order fields from largest to smallest data type to save space.

type Stats struct { // Efficient ordering: 2 bytes + 1 byte + 1 byte = 4 bytes total
	Reach    uint16
	NumPosts uint8
	NumLikes uint8
}

/* Debugging memory layout:
	import "reflect"
	
	typ := reflect.TypeOf(Stats{})
	fmt.Printf("Struct is %d bytes\n", typ.Size())
*/

// 7. EMPTY STRUCTS
// Used as a unary value. They consume exactly 0 bytes of memory!
// Highly useful for channel signals or creating sets from maps.

/* Usage:
	// Anonymous empty struct
	empty1 := struct{}{}        
	
	// Named empty struct type
	type EmptyStruct struct{}  
	empty2 := EmptyStruct{}
*/
```
#### Interfaces
Allow you to focus on what a type does rather than how it's built. Help you define behaviors that different types can share.

```go
// ==========================================
// 1. DEFINING INTERFACES
// ==========================================

// An interface defines a set of method signatures (a "behavior contract").
// Any type that implements ALL methods automatically satisfies the interface.
// No "implements" keyword — it's implicit.

// Design Rules:
// - Keep interfaces small (1-3 methods is ideal).
// - Define interfaces where they're USED, not where they're implemented.
// - They are NOT classes. Think "what can it do?" not "what is it?"

type shape interface {
	area() float64
	perimeter() float64
}

// ==========================================
// 2. IMPLEMENTING AN INTERFACE
// ==========================================

// A type satisfies an interface just by having all the right methods.
// No explicit declaration needed — if it has the methods, it qualifies.

type rect struct {
	width, height float64
}
func (r rect) area() float64 {
	return r.width * r.height
}
func (r rect) perimeter() float64 {
	return 2 * (r.width + r.height)
}

type circle struct {
	radius float64
}
func (c circle) area() float64 {
	return math.Pi * c.radius * c.radius
}
func (c circle) perimeter() float64 {
	return 2 * math.Pi * c.radius
}

// Both rect and circle now satisfy the shape interface.
func printShapeDetails(s shape) {
	fmt.Printf("Area: %.2f, Perimeter: %.2f\n", s.area(), s.perimeter())
}

// ==========================================
// 3. THE EMPTY INTERFACE
// ==========================================

// interface{} (or 'any' in Go 1.18+) has zero methods.
// Every type satisfies it. Use when you need to accept anything.

func logAnything(val interface{}) {
	fmt.Printf("Value: %v, Type: %T\n", val, val)
}

// ==========================================
// 4. TYPE ASSERTIONS
// ==========================================

// When a value is stored in an interface, you can only call the interface's methods.
// The concrete type's fields are hidden behind the interface wrapper.
// A type assertion "unwraps" the interface to get the original type back.
//
// Syntax: concreteVal, ok := interfaceVal.(ConcreteType)
// ok = true  → Correct guess. concreteVal is the fully unwrapped type.
// ok = false → Wrong guess. concreteVal is a useless zero-value.
// WARNING: Skipping the comma-ok and guessing wrong = runtime panic!

func printShapeInfo(s shape) {
	// s.radius won't compile — shape doesn't promise a radius field.
	// Assert s back to circle first to access .radius.
	c, ok := s.(circle)
	if ok {
		fmt.Printf("Circle with radius: %v\n", c.radius)
		return
	}
	r, ok := s.(rect)
	if ok {
		fmt.Printf("Rect %v x %v\n", r.width, r.height)
		return
	}
}

// ==========================================
// 5. TYPE SWITCHES
// ==========================================

// Clean alternative to chaining type assertions.
// Uses the special .(type) syntax — only valid inside a switch.

func describeValue(val interface{}) {
	switch v := val.(type) {
	case int:
		fmt.Printf("Integer: %d\n", v)
	case string:
		fmt.Printf("String: %s\n", v)
	case bool:
		fmt.Printf("Boolean: %t\n", v)
	default:
		fmt.Printf("Unknown type: %T\n", v)
	}
}

// ==========================================
// 6. PRACTICAL EXAMPLE
// ==========================================

// Notification system: multiple types share a common behavior.

// Declaring the interface
type notification interface {
	importance() int
}

// Structs and their methods — each one satisfies notification
type directMessage struct {
	senderUsername string
	messageContent string
	priorityLevel  int
	isUrgent       bool
}
func (d directMessage) importance() int {
	if d.isUrgent {
		return 50
	}
	return d.priorityLevel
}

type groupMessage struct {
	groupName      string
	messageContent string
	priorityLevel  int
}
func (g groupMessage) importance() int {
	return g.priorityLevel
}

type systemAlert struct {
	alertCode      string
	messageContent string
}
func (s systemAlert) importance() int {
	return 100
}

// Type switch to handle each notification type cleanly
func processNotification(n notification) (string, int) {
	switch v := n.(type) {
	case directMessage:
		return v.senderUsername, v.importance()
	case groupMessage:
		return v.groupName, v.importance()
	case systemAlert:
		return v.alertCode, v.importance()
	default:
		return "", 0
	}
}
```
#### Errors
