Install Go
```bash
curl -sS https://webi.sh/golang | sh
source ~/.config/envman/PATH.env
```
#### Variables
```go
// ==========================================
// 1. VARIABLE DECLARATION
// ==========================================

var sadNum int = 42 // Explicit declaration
happyNum := 42      // Short declaration — the GOAT

// ==========================================
// 2. DATA TYPES
// ==========================================

// Use defaults unless performance/memory is an issue.
bool       // true/false
string     // UTF-8 text
int        // Integer (platform-dependent size)
uint       // Unsigned integer (non-negative)
byte       // uint8 alias (8-bit integer)
rune       // int32 alias (Unicode code point/emoji)
float64    // Decimal (64-bit precision)
complex128 // Complex numbers (real + imaginary)

// Sized Types (Memory specific):
// int8, int16, int32, int64
// uint8, uint16, uint32, uint64, uintptr
// float32, float64
// complex64, complex128

// ==========================================
// 3. FORMATTED STRINGS (fmt)
// ==========================================

// Functions:
fmt.Printf()  // Prints formatted string to os.Stdout
fmt.Sprintf() // Returns formatted string as a variable
fmt.Println() // Prints with a newline, no formatting verbs

// Formatting Verbs:
// %v : The "Catch-all" (Default representation)
// %s : String
// %d : Integer (Decimal)
// %t : Boolean
// %f : Float (Use %.2f to round to 2 decimals)
// %T : Type of the value

// ==========================================
// 4. PRACTICAL EXAMPLE
// ==========================================

name := "Gemini"
openRate := 98.765

// Interpolating multiple types into one string
msg := fmt.Sprintf("Hi %s, your open rate is %.1f percent\n", name, openRate)
fmt.Println(msg) // Output: Hi Gemini, your open rate is 98.8 percent
```
#### Conditionals
```go
// ==========================================
// 1. BASIC IF / ELSE IF / ELSE
// ==========================================

height := 5

if height > 6 {
	fmt.Println("You are super tall!")
} else if height > 4 {
	fmt.Println("You are tall enough!")
} else {
	fmt.Println("You are not tall enough!")
}

// ==========================================
// 2. IF WITH INITIAL STATEMENT
// ==========================================

// Syntax: if INITIAL_STATEMENT; CONDITION { }
// Variables declared here are scoped strictly to the if/else blocks.

// Instead of this:
// length := len(email)
// if length < 10 { ... }

// Do this (The "Go Way"):
if length := len(email); length < 10 {
	fmt.Printf("Email must be at least 10 chars, is %d\n", length)
}
// 'length' no longer exists in memory after this block!

// ==========================================
// 3. SWITCH STATEMENTS
// ==========================================

// Cleaner alternative to long if-else chains for exact matches.
// NOTE: 'break' is implicit in Go! It stops automatically after a match.

func getCreator(os string) string {
	switch os {
	case "linux":
		return "Linus Torvalds"
	case "windows":
		return "Bill Gates"
	case "mac":
		return "A Steve"
	default:
		return "Unknown"
	}
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
// ==========================================
// 1. BASIC STRUCTS
// ==========================================

// Represent structured data with named fields.
type Car struct {
	Brand   string
	Mileage int
}

/* Usage:
	myCar := Car{Brand: "Toyota", Mileage: 50000}
	fmt.Println(myCar.Brand) // Access fields with dot operator
*/

// ==========================================
// 2. NESTED STRUCTS
// ==========================================

// Structs that contain other structs as fields.
type Wheel struct {
	Radius int
}
type NestedCar struct {
	FrontWheel Wheel
}

/* Usage:
	myCar := NestedCar{}
	myCar.FrontWheel.Radius = 5 // Chain the dot operator to dig in
*/

// ==========================================
// 3. ANONYMOUS STRUCTS
// ==========================================

// Declared and instantiated at the same time. No reusability.
// Prefer named structs unless it's truly a one-off.

/* Usage:
	myCar := struct {
		Brand string
		Model string
	}{
		Brand: "Toyota",
		Model: "Camry",
	}
*/

// ==========================================
// 4. EMBEDDED STRUCTS
// ==========================================

// Go's version of inheritance (composition over inheritance).
// The embedded struct's fields get "promoted" — accessible directly.

type BaseCar struct {
	Brand string
	Model string
}
type Truck struct {
	BaseCar // Embedded: Truck gets all of BaseCar's fields
	BedSize int
}

/* Usage:
	t := Truck{}
	t.Brand = "Ford"  // Promoted — no need for t.BaseCar.Brand
*/

// ==========================================
// 5. STRUCT METHODS
// ==========================================

// Functions that attach to a type via a "receiver".
// The receiver goes between func and the method name.

// Value Receiver: Gets a COPY of the struct (can't modify the original).
type Rect struct {
	Width  int
	Height int
}
func (r Rect) Area() int {
	return r.Width * r.Height
}

/* Usage:
	r := Rect{Width: 5, Height: 10}
	fmt.Println(r.Area()) // 50
*/

// Methods can take arguments and return multiple values, just like functions.
type User struct {
	MessageCharLimit int
}
func (u User) SendMessage(message string, messageLength int) (string, bool) {
	if messageLength <= u.MessageCharLimit {
		return message, true
	}
	return "", false
}

// ==========================================
// 6. MEMORY LAYOUT
// ==========================================

// Go allocates structs in contiguous blocks of memory.
// The compiler adds "padding" between fields to align them.
// Best practice: Order fields from largest to smallest type to minimize padding.

type Stats struct { // 2 bytes + 1 byte + 1 byte = 4 bytes total
	Reach    uint16
	NumPosts uint8
	NumLikes uint8
}

/* Debugging memory layout:
	typ := reflect.TypeOf(Stats{})
	fmt.Printf("Struct is %d bytes\n", typ.Size())
*/

// ==========================================
// 7. EMPTY STRUCTS
// ==========================================

// Consume exactly 0 bytes of memory.
// Useful for channel signals or creating sets from maps.

/* Usage:
	empty := struct{}{}

	// Set pattern: map with empty struct values
	seen := map[string]struct{}{}
	seen["hello"] = struct{}{}
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
An Error is any type that implements the simple built-in [error interface](https://blog.golang.org/error-handling-and-go):

```go
// ==========================================
// 1. THE ERROR INTERFACE
// ==========================================

// An error in Go is just any type that implements this built-in interface:
type error interface {
	Error() string
}

// A nil error = success. A non-nil error = failure.
// Go has no try/catch. You handle errors explicitly every single time.

// ==========================================
// 2. THE BASIC PATTERN
// ==========================================

// Functions return an error as their last return value.
// You check it immediately. Every. Single. Time.
// Convention: return zero values for everything else when returning a non-nil error.

func sendSMSToCouple(msgToCustomer, msgToSpouse string) (int, error) {
	cost, err := sendSMS(msgToCustomer)
	if err != nil {
		return 0, err
	}
	costSpouse, err := sendSMS(msgToSpouse)
	if err != nil {
		return 0, err
	}
	return costSpouse + cost, nil
}

// ==========================================
// 3. CREATING ERRORS
// ==========================================

// Quick Errors: Use errors.New() for simple error messages.
// https://pkg.go.dev/errors#New
func divide(x, y float64) (float64, error) {
	if y == 0 {
		return 0, errors.New("no dividing by 0")
	}
	return x / y, nil
}

// Formatted Errors: Use fmt.Errorf() when you need to interpolate values.
func withdraw(balance, amount float64) (float64, error) {
	if amount > balance {
		return 0, fmt.Errorf("insufficient funds: tried %.2f, have %.2f", amount, balance)
	}
	return balance - amount, nil
}

// ==========================================
// 4. CUSTOM ERROR TYPES
// ==========================================

// Any type that implements Error() string satisfies the error interface.
// Useful when callers need to inspect the error for extra context.

type userError struct {
	name string
}
func (e userError) Error() string {
	return fmt.Sprintf("%v has a problem with their account", e.name)
}

/* Usage:
	func processUser(name string) error {
		return userError{name: name}
	}
*/

// ==========================================
// 5. PANIC & RECOVER
// ==========================================

// panic() crashes the program and prints a stack trace.
// General Rule: DON'T USE IT. It's not for normal error handling.
// It yeets control up the call stack until it hits a recover() or the program dies.

func enrichUser(userID string) User {
	user, err := getUser(userID)
	if err != nil {
		panic(err) // Nuclear option. Almost always wrong.
	}
	return user
}

// recover() catches a panic inside a deferred function. Still ugly.
func main() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("recovered from panic:", r)
		}
	}()
	enrichUser("123") // panics, but defer/recover catches it
}

// If you want a clean exit for something truly unrecoverable, use log.Fatal().

// ==========================================
// 6. PRACTICAL EXAMPLE
// ==========================================

func validateStatus(status string) error {
	if len(status) == 0 {
		return errors.New("status cannot be empty")
	}
	if len(status) > 140 {
		return errors.New("status exceeds 140 characters")
	}
	return nil
}
```
#### Loops
```go
// ==========================================
// 1. BASIC FOR LOOP
// ==========================================

// Go only has one loop keyword: for
// Syntax: for INITIAL; CONDITION; AFTER { }
// INITIAL   : Runs once at the start
// CONDITION : Checked before each iteration
// AFTER     : Runs after each iteration

func bulkSend(numMessages int) float64 {
	totalCost := 0.0
	for i := 0; i < numMessages; i++ {
		totalCost += 1 + (float64(i) * 0.01)
	}
	return totalCost
}

// ==========================================
// 2. OMITTING LOOP COMPONENTS
// ==========================================

// You can drop any part of the for loop.

// No condition (use break to exit manually):
func maxMessages(thresh int) int {
	counter := 0
	for i := 0; ; i++ {
		if thresh >= (100 + i) {
			thresh -= (100 + i)
			counter += 1
		} else {
			break
		}
	}
	return counter
}

// No anything — Go has no while keyword. This is how you do it:
for {
	// runs forever until break or return
}

// "While-style" — just a condition, no init or after:
for x < 100 {
	x += 10
}

// ==========================================
// 3. CONTINUE & BREAK
// ==========================================

// continue : Skips the rest of this iteration, jumps to next one.
// break    : Exits the loop entirely.

func printPrimes(max int) {
	for n := 2; n < (max + 1); n++ {
		if n == 2 {
			fmt.Println(n)
			continue // Skip even-number check for 2
		} else if n%2 == 0 {
			continue // Skip all other even numbers
		}

		isPrime := true

		for i := 3; i*i <= n; i += 2 {
			if n%i == 0 {
				isPrime = false
				break // Found a divisor, stop checking
			}
		}

		if isPrime {
			fmt.Println(n)
		}
	}
}

// ==========================================
// 4. FIZZBUZZ (CLASSIC EXAMPLE)
// ==========================================

func fizzbuzz() {
	for i := 1; i < 101; i++ {
		if (i%3 == 0) && (i%5 == 0) {
			fmt.Println("fizzbuzz")
		} else if i%3 == 0 {
			fmt.Println("fizz")
		} else if i%5 == 0 {
			fmt.Println("buzz")
		} else {
			fmt.Println(i)
		}
	}
}
```
#### Slices
```go
// ==========================================
// 1. ARRAYS (Fixed Size)
// ==========================================

// Arrays have a fixed length set at declaration. Rarely used directly.
var myInts [10]int                     // Array of 10 integers (all zero-valued)
primes := [6]int{2, 3, 5, 7, 11, 13}  // Initialized literal

// ==========================================
// 2. SLICES (Dynamic Size)
// ==========================================

// Slices are what you'll actually use. They wrap an underlying array
// and can grow or shrink as needed.

// Create from an array:
primes := [6]int{2, 3, 5, 7, 11, 13}
mySlice := primes[1:4] // {3, 5, 7}

// Slice syntax:
// arrayname[lowIndex:highIndex]  — low is inclusive, high is exclusive
// arrayname[lowIndex:]           — from low to end
// arrayname[:highIndex]          — from start to high
// arrayname[:]                   — the whole thing

// Empty slice:
mySlice := []int{}

// Make — create a slice with a specific length (and optional capacity):
// func make([]T, len, cap) []T
mySlice := make([]int, 5, 10) // length 5, capacity 10
mySlice := make([]int, 5)     // capacity defaults to length

// Length vs Capacity:
len(mySlice) // Number of elements the slice contains
cap(mySlice) // Number of elements in the underlying array (rarely matters)

// ==========================================
// 3. APPEND
// ==========================================

// Add elements to a slice. Returns a NEW slice.
// func append(slice []Type, elems ...Type) []Type

slice = append(slice, oneThing)
slice = append(slice, firstThing, secondThing)
slice = append(slice, anotherSlice...) // Spread a slice into append

// Example: Filter costs by day, appending matches to a new slice
type cost struct {
	day   int
	value float64
}
func getDayCosts(costs []cost, day int) []float64 {
	dayCosts := []float64{}
	for i := 0; i < len(costs); i++ {
		cost := costs[i]
		if cost.day == day {
			dayCosts = append(dayCosts, cost.value)
		}
	}
	return dayCosts
}

// ==========================================
// 4. RANGE
// ==========================================

// Cleaner way to iterate over slices (and maps, strings, channels).
// Gives you both the index and the element each iteration.
for INDEX, ELEMENT := range SLICE {}

// Example: Find the index of the first bad word in a message
func indexOfFirstBadWord(msg []string, badWords []string) int {
	for i, word := range msg {
		for _, badWord := range badWords {
			if word == badWord {
				return i
			}
		}
	}
	return -1
}

// ==========================================
// 5. VARIADIC FUNCTIONS & SPREAD
// ==========================================

// Variadic: Accept any number of final arguments using ... in the signature.
// Under the hood, the variadic parameter is just a slice.
func concat(strs ...string) string {
	final := ""
	for i := 0; i < len(strs); i++ {
		final += strs[i]
	}
	return final
}

// Spread Operator: Pass a slice into a variadic function with ...
func printStrings(strings ...string) {
	for i := 0; i < len(strings); i++ {
		fmt.Println(strings[i])
	}
}
func main() {
	names := []string{"bob", "sue", "alice"}
	printStrings(names...) // Spread the slice into individual arguments
}

// ==========================================
// 6. PRACTICAL EXAMPLE
// ==========================================

// Create a slice of costs from a slice of messages
func getMessageCosts(messages []string) []float64 {
	messageCosts := make([]float64, len(messages))
	for i := 0; i < len(messages); i++ {
		cost := float64(len(messages[i])) * 0.01
		messageCosts[i] = cost
	}
	return messageCosts
}

// Slice of Slices -- effectively creating a matrix
rows := [][]int{} // declare 2D slice
rows = append(rows, []int{1, 2, 3})
rows = append(rows, []int{4, 5, 6})
fmt.Println(rows)
// [[1 2 3] [4 5 6]]

// example
func createMatrix(rows, cols int) [][]int {
	// 1. Declare as a 2D slice ([][]int)
	matrix := make([][]int, rows)
	for i := 0; i < rows; i++ {
		// 2. Use standard assignment (=) instead of short declaration (:=)
		matrix[i] = make([]int, cols)
		for j := 0; j < cols; j++ {
			matrix[i][j] = i * j
		}
	}	
	return matrix
}

// tricky slices -- The append() function changes the underlying array of its parameter AND returns a new slice. Ussing append on anything other than itself is a BAD idea.
someSlice = append(otherSlice, element) // don't do this
```