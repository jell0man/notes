Install Go
```bash
curl -sS https://webi.sh/golang | sh
source ~/.config/envman/PATH.env
```
#### Proverbs
```
Don't communicate by sharing memory, share memory by communicating.

Concurrency is not parallelism.

Channels orchestrate; mutexes serialize.

The bigger the interface, the weaker the abstraction.

Make the zero value useful.

interface{} says nothing.

Gofmt's style is no one's favorite, yet gofmt is everyone's favorite.

A little copying is better than a little dependency.

Syscall must always be guarded with build tags.

Cgo must always be guarded with build tags.

Cgo is not Go.

With the unsafe package there are no guarantees.

Clear is better than clever.

Reflection is never clear.

Errors are values.

Don't just check errors, handle them gracefully.

Design the architecture, name the components, document the details.

Documentation is for users.

Don't panic.
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
// 2. BOOLEAN SHORTHAND
// ==========================================

// Booleans don't need to be compared to true/false explicitly.
// if ok       is the same as    if ok == true
// if !ok      is the same as    if ok == false

// Go way:
if ok {
	fmt.Println("success")
}

// Negation:
if !ok {
	fmt.Println("something went wrong")
}

// ==========================================
// 3. IF WITH INITIAL STATEMENT
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
// 4. SWITCH STATEMENTS
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
// 4. RANGE
// ==========================================

// range lets you iterate over slices, maps, strings, and channels
// without managing an index manually.

// Over a slice — gives you (index, value):
for i, val := range mySlice {
	fmt.Printf("index: %d, value: %v\n", i, val)
}

// Don't need the index? Use blank identifier:
for _, val := range mySlice {
	fmt.Println(val)
}

// Don't need the value? Just use the index:
for i := range mySlice {
	fmt.Println(i)
}

// Over a map — gives you (key, value):
for key, value := range myMap {
	fmt.Printf("%s: %v\n", key, value)
}

// Over a string — gives you (index, rune):
for i, ch := range "hello" {
	fmt.Printf("index: %d, char: %c\n", i, ch)
}

// Example: Load emails into a buffered channel
func addEmailsToQueue(emails []string) chan string {
	emailsToSend := make(chan string, len(emails))
	for _, email := range emails {
		emailsToSend <- email
	}
	return emailsToSend
}

// ==========================================
// 5. FIZZBUZZ (CLASSIC EXAMPLE)
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
primes := [6]int{2, 3, 5, 7, 11, 13}   // Initialized literal

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

// WARNING: append() modifies the underlying array AND returns a new slice.
// ALWAYS append a slice to ITSELF. Never append to a different slice variable.
someSlice = append(otherSlice, element) // DON'T do this!
someSlice = append(someSlice, element)  // DO this.

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
// 5. SLICE OF SLICES (2D Slices / Matrices)
// ==========================================

// Declare and build a 2D slice by appending rows:
rows := [][]int{}
rows = append(rows, []int{1, 2, 3})
rows = append(rows, []int{4, 5, 6})
// [[1 2 3] [4 5 6]]

// Or build one programmatically with make:
func createMatrix(rows, cols int) [][]int {
	matrix := make([][]int, rows)
	for i := 0; i < rows; i++ {
		matrix[i] = make([]int, cols) // Use = not := (matrix[i] already exists)
		for j := 0; j < cols; j++ {
			matrix[i][j] = i * j
		}
	}
	return matrix
}

// ==========================================
// 6. VARIADIC FUNCTIONS & SPREAD
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
// 7. PRACTICAL EXAMPLES
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

// Iterating through a string for password validation
func isValidPassword(password string) bool {
	uppercaseCount := 0
	digitCount := 0
	for i := 0; i < len(password); i++ {
		char := password[i]
		if char >= 'A' && char <= 'Z' {
			uppercaseCount++
		} else if char >= '0' && char <= '9' {
			digitCount++
		}
	}
	return len(password) >= 5 && len(password) <= 12 && uppercaseCount > 0 && digitCount > 0
}

// Tagging and filtering messages using slices, structs, and first-class functions
type sms struct {
	id      string
	content string
	tags    []string
}

func tagMessages(messages []sms, tagger func(sms) []string) []sms {
	for i := 0; i < len(messages); i++ {
		messages[i].tags = tagger(messages[i])
	}
	return messages
}

func tagger(msg sms) []string {
	tags := []string{}
	lowerContent := strings.ToLower(msg.content)

	if strings.Contains(lowerContent, "urgent") {
		tags = append(tags, "Urgent")
	}
	if strings.Contains(lowerContent, "sale") {
		tags = append(tags, "Promo")
	}
	return tags
}
```
#### Maps
```go
// ==========================================
// 1. CREATING MAPS
// ==========================================

// Maps provide key -> value mapping. Zero value of a map is nil.

// Using make():
ages := make(map[string]int)
ages["John"] = 37
ages["Mary"] = 24
ages["Mary"] = 21 // Overwrites 24

// Using a literal:
ages := map[string]int{
	"John": 37,
	"Mary": 21,
}

// ==========================================
// 2. MUTATIONS
// ==========================================

m[key] = elem         // Insert or update
elem = m[key]         // Get (returns zero value if key missing)
delete(m, key)        // Delete
elem, ok := m[key]    // Check existence (ok = true if found)

// ==========================================
// 3. KEY TYPES
// ==========================================

// Values can be any type. Keys must be comparable:
// boolean, numeric, string, pointer, channel, interface, struct, array
// NOT: slices, maps, or functions

// ==========================================
// 4. COMMA-OK PATTERN
// ==========================================

// Combine existence check with if-initial-statement (the "Go Way").
// Ties directly into the conditional pattern from the Conditionals section.

names := map[string]int{}
missingNames := []string{}

if _, ok := names["Denna"]; !ok {
	missingNames = append(missingNames, "Denna")
}

// Example: Delete a user only if they exist and are scheduled for deletion
func deleteIfNecessary(users map[string]user, name string) (deleted bool, err error) {
	user, ok := users[name]
	if !ok {
		return false, errors.New("not found")
	}
	if !user.scheduledForDeletion {
		return false, nil
	}
	delete(users, name)
	return true, nil
}

// Example: Increment counts only for valid users
func updateCounts(messagedUsers []string, validUsers map[string]int) {
	for _, name := range messagedUsers {
		if _, ok := validUsers[name]; ok {
			validUsers[name]++
		}
	}
}

// ==========================================
// 5. NESTED MAPS
// ==========================================

// Maps can contain maps. You must initialize inner maps before using them!
// map[string]map[string]int
// map[rune]map[string]int

// Example: Group name counts by first letter
func getNameCounts(names []string) map[rune]map[string]int {
	nameCounts := make(map[rune]map[string]int)
	for _, name := range names {
		firstLetter := []rune(name)[0]

		if _, ok := nameCounts[firstLetter]; !ok {
			nameCounts[firstLetter] = make(map[string]int) // Init inner map!
		}
		nameCounts[firstLetter][name]++
	}
	return nameCounts
}

// ==========================================
// 6. SETS (Using Empty Structs)
// ==========================================

// Go has no set type. Use a map with empty struct values (0 bytes each).
// Ties back to empty structs from the Structs section.

func countDistinctWords(messages []string) int {
	distinctWords := make(map[string]struct{})
	for _, message := range messages {
		for _, word := range strings.Fields(strings.ToLower(message)) {
			distinctWords[word] = struct{}{} // Zero-byte value, just marks presence
		}
	}
	return len(distinctWords)
}

// ==========================================
// 7. PRACTICAL EXAMPLE
// ==========================================

// Map names to phone numbers from two parallel slices
type user struct {
	name        string
	phoneNumber int
}

func getUserMap(names []string, phoneNumbers []int) (map[string]user, error) {
	if len(names) != len(phoneNumbers) {
		return nil, errors.New("invalid sizes")
	}
	users := make(map[string]user)
	for i := 0; i < len(names); i++ {
		users[names[i]] = user{
			name:        names[i],
			phoneNumber: phoneNumbers[i],
		}
	}
	return users, nil
}
```
#### Pointers
```go
// ==========================================
// 1. POINTER BASICS
// ==========================================

// A pointer stores the memory address of another variable.
// *T  — declares a pointer to type T
// &x  — gets the address of x (creates a pointer)
// *p  — dereferences a pointer (gets the value it points to)

myString := "hello"
myStringPtr := &myString // myStringPtr holds the memory address of myString

fmt.Println(myStringPtr)  // 0x140c050 (some memory address)
fmt.Println(*myStringPtr) // "hello" (dereferenced — the actual value)

// Dereference to read or write through the pointer:
*myStringPtr = "world"
fmt.Println(myString) // "world" — the original variable changed!

// ==========================================
// 2. PASS BY REFERENCE
// ==========================================

// Recall: Go functions receive COPIES of arguments by default.
// Pointers let you modify the original variable from inside a function.

func increment(x *int) {
	*x++ // Modify the original value through the pointer
}
func main() {
	x := 5
	increment(&x) // Pass the address of x
	fmt.Println(x) // 6 — x was modified!
}

// Example: Censor profanity in-place (no return needed)
func removeProfanity(message *string) {
	messageVal := *message
	messageVal = strings.ReplaceAll(messageVal, "fubb", "****")
	messageVal = strings.ReplaceAll(messageVal, "shiz", "****")
	messageVal = strings.ReplaceAll(messageVal, "witch", "*****")
	*message = messageVal
}

// ==========================================
// 3. NIL POINTERS
// ==========================================

// A pointer that points to nothing is nil.
// Dereferencing a nil pointer = runtime panic (crash).
// Solution: Check for nil and return early.

var p *int
fmt.Println(p) // <nil>

if message == nil {
	return // Bail out before dereferencing
}

// ==========================================
// 4. POINTER RECEIVERS (Methods)
// ==========================================

// A method receiver can be a pointer, letting the method modify the struct.
// Pointer receivers are more common than value receivers.

type car struct {
	color string
}

// Pointer receiver — modifies the original struct:
func (c *car) setColor(color string) {
	c.color = color
}

// Value receiver — modifies a COPY, original is untouched:
// func (c car) setColor(color string) {
//     c.color = color // This changes nothing outside the method!
// }

func main() {
	c := car{color: "white"}
	c.setColor("blue")
	fmt.Println(c.color) // "blue" (pointer receiver modified the original)
}

// ==========================================
// 5. STRUCT POINTER SHORTHAND
// ==========================================

// Go automatically dereferences struct pointers for field access.
// No need to manually dereference with (*p).Field.

// These are equivalent:
(*analytics).MessagesTotal // Explicit dereference (verbose)
analytics.MessagesTotal    // Shorthand (the Go way)

// WRONG: This tries to dereference the field, not the struct!
// *analytics.MessagesTotal

// ==========================================
// 6. POINTER PERFORMANCE
// ==========================================

// Rule: Use pointers when you need a shared reference to a value.
// Don't use pointers just for "performance" unless you've profiled.
//
// Stack vs Heap:
// - Local non-pointer variables live on the stack (fast).
// - Pointers can cause values to "escape" to the heap (slower).
// - Copying small structs is often faster than pointer indirection.

// ==========================================
// 7. PRACTICAL EXAMPLE
// ==========================================

type customer struct {
	id      int
	balance float64
}

type transactionType string
const (
	transactionDeposit    transactionType = "deposit"
	transactionWithdrawal transactionType = "withdrawal"
)

type transaction struct {
	customerID      int
	amount          float64
	transactionType transactionType
}

// Pointer to customer lets us update the balance in-place.
func updateBalance(c *customer, t transaction) error {
	switch t.transactionType {
	case transactionDeposit:
		c.balance += t.amount
		return nil
	case transactionWithdrawal:
		if c.balance < t.amount {
			return errors.New("insufficient funds")
		}
		c.balance -= t.amount
		return nil
	default:
		return errors.New("unknown transaction type")
	}
}
```
#### Packages and Modules
```go
// ==========================================
// 1. PACKAGES
// ==========================================

// A "main" package compiles into an executable. Entry point is main().
package main

// Any other name is a library package (no executable, just importable code).
package textio

// By convention, package name matches the last element of its import path.
package parser // from github.com/wagslane/parser

// Exported vs Unexported:
// Capitalized = exported (accessible outside the package)
// Lowercase   = unexported (private to the package)
func Reverse(s string) string {} // Exported — other packages can call this
func helper() {}                 // Unexported — only usable inside this package

// ==========================================
// 2. MODULES
// ==========================================

// A module is a collection of Go packages released together.
// A repository usually contains one module.
// A module is defined by a go.mod file at the root.
//
// Import path = module path + subdirectory:
// github.com/google/go-cmp/cmp  →  "cmp" is the package
// Standard library packages have no module path prefix (just "fmt", "errors", etc.)

// ==========================================
// 3. YOUR FIRST PROGRAM
// ==========================================

// Initialize a module:
// mkdir hellogo && cd hellogo
// go mod init example.com/username/hellogo

// Write main.go:
package main

import "fmt"

func main() {
	fmt.Println("hello world")
}

// go run main.go   — Compile and run in one shot (dev/testing)
// go build         — Compile into a static executable (./hellogo)
// go install       — Compile and install to $GOPATH/bin (run from anywhere)

// ==========================================
// 4. CREATING A LIBRARY PACKAGE
// ==========================================

// Create a separate module for reusable code:
// mkdir mystrings && cd mystrings
// go mod init example.com/username/mystrings

// mystrings.go — package name matches directory name by convention:
package mystrings

func Reverse(s string) string {
	result := ""
	for _, v := range s {
		result = string(v) + result
	}
	return result
}
// No main.go or func main() — this is a library, not an executable.
// go build compiles it to the local cache but produces no binary.

// Import it from hellogo/main.go:
package main

import (
	"fmt"
	"example.com/username/mystrings"
)
func main() {
	fmt.Println(mystrings.Reverse("hello world"))
}

// For LOCAL packages, add a replace directive to hellogo/go.mod:
// replace example.com/username/mystrings v0.0.0 => ../mystrings
// require example.com/username/mystrings v0.0.0
//
// NOTE: "replace" is a dev shortcut, not for production. Use remote packages instead.

// ==========================================
// 5. REMOTE PACKAGES
// ==========================================

// go get downloads and installs a remote package:
// go get github.com/wagslane/go-tinytime

package main

import (
	"fmt"
	"time"
	tinytime "github.com/wagslane/go-tinytime" // Aliased import
)
func main() {
	tt := tinytime.New(1585750374)
	tt = tt.Add(time.Hour * 48)
	fmt.Println("1585750374 converted to a tinytime is:", tt)
}

// Workflow:
// go mod init example.com/username/datetest  — Initialize module
// go get github.com/wagslane/go-tinytime     — Download remote package
// go build                                   — Compile
// ./datetest                                  — Run

// ==========================================
// 6. CLEAN PACKAGE DESIGN
// ==========================================

// 1. Hide internal logic — only export what consumers need.
// 2. Don't change APIs — keep exported function signatures stable across versions.
// 3. Don't export from main — main is an app, not a library.
// 4. Packages shouldn't know about dependents — no references to the apps that use them.
```
#### Concurrency
```go
// ==========================================
// WHAT'S A CHANNEL?
// ==========================================

// A channel is just a pipe between goroutines.
// One goroutine shoves values in one end, another pulls them out the other.
//
//  goroutine A ─── ch <- 42 ──>  [  CHANNEL  ]  ──> v := <-ch  ── goroutine B
//

// ==========================================
// 1. GOROUTINES
// ==========================================

// Concurrency = multiple tasks making progress at the same time.
// A goroutine is a lightweight concurrent "thread" managed by the Go runtime.
// Launch one by putting 'go' in front of a function call.

go doSomething()

// Example: "Email sent" prints BEFORE "Email received" because the
// anonymous goroutine runs concurrently with a 250ms delay.
func sendEmail(message string) {
	go func() {
		time.Sleep(time.Millisecond * 250)
		fmt.Printf("Email received: '%s'\n", message)
	}()
	fmt.Printf("Email sent: '%s'\n", message)
}

// ==========================================
// 2. CHANNELS
// ==========================================

// Channels are typed, thread-safe queues that let goroutines talk to each other.
// One goroutine shoves values in, another pulls them out. FIFO.
// They are reference types (like maps and slices).

ch := make(chan int) // Create a channel that carries ints

// The <- operator sends and receives. Data flows in the direction of the arrow.
ch <- 69     // Send 69 into the channel
v := <-ch    // Receive a value from the channel into v

// Both send and receive BLOCK until the other side is ready.

// Example: Send from a goroutine, receive in main
func send(ch chan int) {
	ch <- 99
}
func main() {
	ch := make(chan int)
	go send(ch)
	fmt.Println(<-ch) // 99
}

// ==========================================
// 3. BUFFERED CHANNELS
// ==========================================

// Pass a capacity as the second argument to make().
// Sends don't block until the buffer is full.
// Receives don't block until the buffer is empty.

ch := make(chan int, 100) // Buffer holds 100 values before sending blocks

// Example: Pre-load a work queue for other goroutines to consume
func addEmailsToQueue(emails []string) chan string {
	emailChannel := make(chan string, len(emails))
	for i := 0; i < len(emails); i++ {
		emailChannel <- emails[i]
	}
	return emailChannel
}

// ==========================================
// 4. CLOSING CHANNELS
// ==========================================

// Only SENDERS close channels. Sending on a closed channel = panic!
ch := make(chan int)
close(ch)

// Check if a channel is closed with comma-ok:
v, ok := <-ch // ok = false if channel is empty AND closed

// Receive from a closed channel returns the zero value immediately:
var ch = make(chan int, 100)
close(ch)
fmt.Println(<-ch) // 0

// ==========================================
// 5. RANGING OVER CHANNELS
// ==========================================

// range on a channel blocks until each value arrives, stops when channel closes.
for item := range ch {
	// item is the next value received from the channel
}

// Example: Generate fibonacci numbers concurrently
func concurrentFib(n int) []int {
	intChan := make(chan int)
	fibSlice := []int{}

	go fibonacci(n, intChan) // Generate in a separate goroutine

	for item := range intChan { // Blocks until channel closes
		fibSlice = append(fibSlice, item)
	}
	return fibSlice
}

func fibonacci(n int, ch chan int) {
	x, y := 0, 1
	for i := 0; i < n; i++ {
		ch <- x
		x, y = y, x+y
	}
	close(ch) // Signals to receiver that generation is done
}

// ==========================================
// 6. SELECT
// ==========================================

// Listen to multiple channels at once. Like a switch but for channels.
// The first channel with a value ready fires. If multiple are ready, one is chosen randomly.

select {
case i, ok := <-chInts:
	if ok {
		fmt.Println(i)
	}
case s, ok := <-chStrings:
	if ok {
		fmt.Println(s)
	}
}

// Example: Log messages from two channels until one closes
func logMessages(chEmails, chSms chan string) {
	for {
		select {
		case email, ok := <-chEmails:
			if !ok {
				return
			}
			logEmail(email)
		case sms, ok := <-chSms:
			if !ok {
				return
			}
			logSms(sms)
		}
	}
}

// Default Case: Runs immediately if no channel is ready. Prevents blocking.
select {
case v := <-ch:
	// use v
default:
	// receiving from ch would block, do something else
}

// Ignoring the value — just care that something arrived:
select {
case <-ch:
	// event received, value discarded
default:
	// nothing ready
}

// ==========================================
// 7. DIRECTIONAL CHANNELS
// ==========================================

// Restrict a channel to read-only or write-only in a function signature.
// Catches misuse at compile time.

func readOnly(ch <-chan int) {
	// Can only receive from ch
}

func writeOnly(ch chan<- int) {
	// Can only send to ch
}

// ==========================================
// 8. TIME UTILITIES
// ==========================================

time.Sleep(500 * time.Millisecond) // Block current goroutine for a duration
time.Tick(500 * time.Millisecond)  // Returns a channel that sends on an interval
time.After(2 * time.Second)        // Returns a channel that sends once after a delay

// ==========================================
// 9. NIL & CLOSED CHANNEL GOTCHAS
// ==========================================

// A declared but uninitialized channel is nil (just like slices):
var ch chan string // ch is nil

// Send to nil channel    → blocks forever
// Receive from nil channel → blocks forever
// Send to closed channel  → PANIC
// Receive from closed channel → returns zero value immediately
```
#### Mutexes
```go
// ==========================================
// 1. WHAT'S A MUTEX?
// ==========================================

// Mutex = "mutual exclusion".
// It locks shared data so only one goroutine can access it at a time.
// Without it, concurrent goroutines reading/writing the same data = race condition.

sync.Mutex // The type
mu.Lock()  // Lock — blocks if already locked by another goroutine
mu.Unlock() // Unlock — lets the next goroutine in

// Best practice: Use defer to guarantee the unlock always happens.
func protected() {
	mu.Lock()
	defer mu.Unlock()
	// Everything below is protected.
	// Any other call to mu.Lock() will block until this function returns.
}

// ==========================================
// 2. PROTECTING SHARED DATA
// ==========================================

// Maps are NOT thread-safe. Concurrent reads/writes without a mutex = crash.
// Embed a mutex in your struct to protect access.

type safeCounter struct {
	counts map[string]int
	mu     *sync.Mutex
}

func (sc safeCounter) inc(key string) {
	sc.mu.Lock()
	defer sc.mu.Unlock()
	sc.slowIncrement(key)
}

func (sc safeCounter) val(key string) int {
	sc.mu.Lock()
	defer sc.mu.Unlock()
	return sc.slowVal(key)
}

// ==========================================
// 3. RW MUTEX (Read/Write Mutex)
// ==========================================

// sync.RWMutex — optimized for read-heavy workloads.
// Writers: Only 1 at a time (exclusive lock, same as regular mutex).
// Readers: Unlimited at the same time (shared lock, much faster).

sync.RWMutex // The type
mu.RLock()   // Read-lock — multiple goroutines can hold this simultaneously
mu.RUnlock() // Read-unlock

// Use RLock/RUnlock for methods that only READ:
func (sc safeCounter) val(key string) int {
	sc.mu.RLock()
	defer sc.mu.RUnlock()
	return sc.counts[key]
}

// Use Lock/Unlock for methods that WRITE:
func (sc safeCounter) inc(key string) {
	sc.mu.Lock()
	defer sc.mu.Unlock()
	sc.counts[key]++
}
```
#### Generics
```go
// ==========================================
// 1. TYPE PARAMETERS
// ==========================================

// Go has no classes, making code reuse kinda inconvenient.
// Generics (Go 1.18+) let you use variables to refer to specific types.
// Took forever because Go is meant to be simplistic...

// T is the type parameter. 'any' means it accepts all types.
func splitAnySlice[T any](s []T) ([]T, []T) {
	mid := len(s) / 2
	return s[:mid], s[mid:]
}

// Example: Get the last item from any slice
func getLast[T any](s []T) T {
	var zero T
	if len(s) == 0 {
		return zero // Return the zero value of whatever T is
	}
	return s[len(s)-1]
}

// Generics reduce repetitive code.
// Used more often in libraries and packages than application code.
// Naming: T is convention, but any name works.

// ==========================================
// 2. CONSTRAINTS
// ==========================================

// Sometimes your generic function needs to DO something with the type
// (not just pass it around). Constraints restrict which types are allowed.

// Custom constraint using an interface:
type stringer interface {
	String() string
}

func concat[T stringer](vals []T) string {
	result := ""
	for _, val := range vals {
		result += val.String() // Only works because stringer guarantees .String()
	}
	return result
}

// ==========================================
// 3. TYPE LISTS (Type-Set Interfaces)
// ==========================================

// Instead of listing methods, list the concrete types that are allowed.
// Uses the ~ operator to include types with the same underlying type.

// Ordered matches any type that supports <, <=, >, >=
type Ordered interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64 |
		~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
		~float32 | ~float64 |
		~string
}

// Because T is constrained by Ordered, the compiler knows < is valid.
func Min[T Ordered](a, b T) T {
	if a < b {
		return a
	}
	return b
}

// ==========================================
// 4. PARAMETRIC CONSTRAINTS
// ==========================================

// Constraint interfaces can accept type parameters too.
type biller[C customer] interface {
	Charge(C) bill
	Name() string
}
```
#### Enums
```go
// ==========================================
// 1. IOTA (Pseudo-Enums)
// ==========================================

// Go has no real enums, sum types, or tagged unions.
// The type system is more C than Rust — simplicity over expressiveness.
// Iota is the closest thing. If you squint really hard, it kinda looks like an enum.

type sendingChannel int

const (
	Email sendingChannel = iota // 0
	SMS                         // 1
	Phone                       // 2
)

// Iota auto-increments starting at 0 for each constant in a const block.
// But there's no exhaustiveness check — nothing stops you from assigning
// any random int to a sendingChannel. It's convention, not enforcement.

// ==========================================
// 2. ERROR WRAPPING
// ==========================================

// Errors are just values. Wrap them with fmt.Errorf and %w to add context
// as they bubble up the call stack. This builds a chain you can inspect later.

user, err := getUser()
if err != nil {
	return fmt.Errorf("failed to get user: %w", err)
}

// The developer CAN just ignore the error and use the data anyway,
// even if it's invalid (probably nil or empty struct). Don't be that guy.

// Pattern: Check each step, wrap and return immediately on failure.
func (a *analytics) handleEmailBounce(em email) error {
	if err := em.recipient.updateStatus(em.status); err != nil {
		return fmt.Errorf("error updating user status: %w", err)
	}
	if err := a.track(em.status); err != nil {
		return fmt.Errorf("error tracking user bounce: %w", err)
	}
	return nil
}
```
