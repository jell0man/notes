Install Golang
```bash
curl -sS https://webi.sh/golang | sh
source ~/.config/envman/PATH.env
```

Variable Declaration
```go
// Sad variable declaration
var sadNum int
sadNum = 42

// GOATed variable declaration
happyNum := 42
```

Types
```go
// Use defaults unless performance is an issue
// Defaults
bool       // boolean
string     // string 
int        // integer
uint       // unsigned integers - non-negative/no decimal
byte       // uint8 - 8 bit integer
rune       // int32 - thing unicode
float64    // decimals - used 64 bits of memory
complex128 // complex numbers - real and imaginary

// Integers, uints, floats, and complex numbers all have type sizes.
int  int8  int16  int32  int64
uint uint8 uint16 uint32 uint64 uintptr
float32 float64
complex64 complex128
```