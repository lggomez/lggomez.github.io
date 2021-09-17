{
    "title": "The otherworldly landscape of enums in go",
    "description": "The state of enums in go and how to create a self validating type generator",
    "summary": "The state of enums in go and how to create a self validating type generator",
    "tags": ["Golang", "Enums", "Scaffolding"],
    "date": "2021-03-07T02:43:15-03:00",
    "lastmod": "2021-03-07T02:43:15-03:00",
    "categories": ["Programming", "Backend", "Intermediate"],
    "type": "",
    "weight": 0,
    "publishdate": null,
    "expirydate": null,
    "aliases": null,
    "slug": "the-otherworldly-landscape-of-enums-in-go",
    "url": ""
}

Hi there. Let's talk about enumerations (commonly named enums) in golang.  

In other languages, enums commonly work as a closed type (usually aliasing or composing the int and/or string primitives) to provide the user some additional flexibilites that constants do not provide:
* Aliasing: we can think of enums as constants, aliased on a type level 
* Immutability: some languages work on numeric enums as numeric masks; howevers, their values are not altered during operations
* Extensibility: enums can (and have to be) easily extendable to the desired amount of values (at least with no extra burden on the developer on coding and compile time). After all, we can also think of them as a static collection of constants
* Value-safety: that is, an enum value is unique but can be unboxed into its primitive value
* Typed operations: as a bonus, language defined support provides free, out of the box method and operations like value checking and traversal
Golang offers varied, but in my opinion, limited ways to implement enums. 

Let's see how messy we can get, depending on __how many__ of the above guarantees we want.

## The naive way: constants 
Start with the basics: after all, enum types are a group of constants of an integral/primitive type bound by another type (generically speaking). So, I wanna have my weekdays:

```go
const (
    Sunday = "Sunday"
    Monday = "Monday"
    Tuesday = "Tuesday"
    Wednesday = "Tuesday"
    Thursday = "Thursday"
    Friday = "Friday"
    Saturday = "Saturday"
)
```

## The idiomatic way: constant aliases, a.k.a iota enums 
This is the classic and idiomatic way offered by the language to implement basic enumerations. In fact, the native time package offers a Weekday enum out of the box:

```go
type Weekday int
const (
    Sunday Weekday = iota
    Monday
    Tuesday
    Wednesday
    Thursday
    Friday
    Saturday
)

// String returns the English name of the day ("Sunday", "Monday", ...).
func (d Weekday) String() string {
	//...
}
```

A couple (well, several...) things to note here:
* The "enum" here already feels half baked, as it is a constant collection (given a syntactic  shortcut) tied to a definition of a type alias
* Also, one must implement its Stringer interface if they want to give it a string representation (the language does not support infering it from the label)
* **iota** is a constant equal to zero. It is convention to use it as the first value, and the subsequent entries of the const block will have autoincremental values
* Under this convention, values lower than iota are assumed invalid. Consumers must adhere to this convention so this is already a leaky leanguage feature IMO, which depends on the good faith of the developers
* Given the above, do we validate values greater than iota upon type conversion?
* The only way to enumerate is to explicitly make a switch between the values

## Hot topic 
Since I want to avoid copypasting code from users without context, here is a very interesting [StackOverflow](https://stackoverflow.com/questions/14426366/what-is-an-idiomatic-way-of-representing-enums-in-go) thread in which users go wild with their own ways to represent enumerations beyond the capabilities of the standard approach
And it also it a popular issue in the official issue tracker, having several proposals standing:
* proposal: spec: add typed enum support - https://github.com/golang/go/issues/19814
* proposal: Go 2: enums as an extension to types - https://github.com/golang/go/issues/28987
* proposal: Go 2: exhaustive switching for enum type-safety - https://github.com/golang/go/issues/36387
* ...and the list goes on: https://github.com/golang/go/issues?q=enum+label%3ALanguageChange+label%3AProposal+

And and even bigger list of enum generators, helpers, and implementations: https://github.com/search?l=&q=enum+language%3AGo&type=repositories
This includes mine somewhere, which I will start to discuss below.

## Adding the sugar ourselves with our custom type 
Having said that, what do I expect from an enum type to be compatible with in a large application or general use case?
1. Immutability
2. Ability to be traversable
3. Enum fields compatible with several base and (un)marshal interfaces, including runtime validation:
	a. Stringer
	b. json.Marshaler, json.Unmarshaler
	c. text.Marshaler, text.Unmarshaler
	d. json.Marshaler, json.Unmarshaler
	e. gob.GobEncoder, gob.GobDecoder
	f. driver.Valuer, sql.Scanner
	g. bson.Marshaler, bson.Unmarshaler (from go.mongodb.org/mongo-driver/bson package)
4. Ability to perform type-safe comparisons at runtime against strings and instances of the same enum type

All of this sounds difficult to reutilize without any sort of code generation, so of course `go generate` was involved in the making. For this case, we will create a string enum type.

### Creating a core enum type 
The core enum type will have the job of keeping a registry of the exposed enumerations as a map of lookup maps. This is for two reasons:
1. This enum type is designed to work on a package level, so ideally more than one enumeration will be expected here and have to be resolved
2. Value lookups will be performed to validate incoming values un unmarshal operations

```go
package enum

import (
	"strings"
)

// Base value index for internal validations
var (
	enumIndex = map[string]valueIndex{}
)

type valueIndex map[string]struct{}

// stringEnumValue is a type designed to be embeddable in other structs so
// they can expose type safe string enumerations and all of their
// corresponding marshaling and unmarshaling operations
//
// This type is NOT meant to be consumed directly
type stringEnumValue struct {
	value string
	key   string
}

func fromValue(value string, ignoreCase bool, key string) (stringEnumValue, bool) {
	if index, ok := enumIndex[key]; ok {
		if ignoreCase {
			value = strings.ToUpper(value)
		}
		for v := range index {
			if ignoreCase {
				v = strings.ToUpper(v)
			}
			if v == value {
				return stringEnumValue{v, key}, true
			}
		}
	}
	return stringEnumValue{}, false
}

// Equals does a case sensitive comparison against a string value
func (e stringEnumValue) Equals(s string) bool {
	return e.value == s
}

// Equals does a case insensitive comparison against a string value
func (e stringEnumValue) EqualsIgnoreCase(s string) bool {
	return strings.ToUpper(e.value) == strings.ToUpper(s)
}

// IsEmpty returns whether the value is empty.
// Such values are permitted in order to support empty values on compile and (un)marshaling time without pointer fields
func (e stringEnumValue) IsEmpty() bool { return e.value == "" }

// IsUndefined returns whether the value is manually initialized, thus being undefined (MyEnumType{}).
// Such values are permitted in order to support empty values on compile and (un)marshaling time without pointer fields
func (e stringEnumValue) IsUndefined() bool { return e.IsEmpty() && e.key == "" }
```

### Codecs: handling validation upon marshalling 
Here we implement the support for all the marshallers/serializers we want on the enum support, validating that the incoming value is indeed one of the enumeration constants, otherwise returning a marshalling validation error.

```go
package enum

import (
	"database/sql/driver"
	"encoding/json"
	"errors"
	"fmt"
	"go.mongodb.org/mongo-driver/bson"
)

// codecs: marshal/unmarshal methods for several native interfaces:
// 	- Stringer
// 	- json.Marshaler, json.Unmarshaler
// 	- text.Marshaler, text.Unmarshaler
// 	- bson.Marshaler, bson.Unmarshaler
// 	- json.Marshaler, json.Unmarshaler
// 	- gob.GobEncoder, gob.GobDecoder
// 	- driver.Valuer, sql.Scanner
const (
	JSONNull string = "null"
)

// Validate and assign value to unmarshal into target enum instance.
//
// Since we know the specific type by the key injected in e.key, we
// check against the enum index to validate the incoming value
func (e *stringEnumValue) validateValueByKey(value string) error {
	if _, ok := enumIndex[e.key][value]; !ok && (value != "") {
		return fmt.Errorf("stringEnumValue: value '%v' is not allowed", value)
	}
	return nil
}

// Stringer implementation
func (e stringEnumValue) String() string { return e.value }

// MarshalJSON returns the stringEnumValue value as JSON
func (e stringEnumValue) MarshalJSON() ([]byte, error) {
	data, err := json.Marshal(e.value)
	return data, err
}

// UnmarshalJSON sets the stringEnumValue value from JSON
func (e *stringEnumValue) UnmarshalJSON(data []byte) error {
	var value string
	if err := json.Unmarshal(data, &value); err != nil {
		return err
	}
	if err := e.validateValueByKey(value); err != nil {
		return err
	}
	e.value = value
	return nil
}

// UnmarshalText parses a text representation into a date types
func (e *stringEnumValue) UnmarshalText(text []byte) error {
	value := string(text)
	if err := e.validateValueByKey(value); err != nil {
		return err
	}
	e.value = value
	return nil
}

// MarshalText serializes this date types to string
func (e stringEnumValue) MarshalText() ([]byte, error) {
	data := []byte(e.String())
	return data, nil
}

// Scan scans a stringEnumValue value from database driver types.
func (e *stringEnumValue) Scan(raw interface{}) error {
	switch v := raw.(type) {
	case []byte:
		return e.UnmarshalText(v)
	case string:
		return e.UnmarshalText([]byte(v))
	default:
		return fmt.Errorf("cannot sql.Scan() enum.stringEnumValue from: %#v", v)
	}
}

// Value converts stringEnumValue to a primitive value ready to written to a database.
func (e stringEnumValue) Value() (driver.Value, error) {
	return driver.Value(e.String()), nil
}

// MarshalBSON implements the bson.Marshaler interface.
func (e stringEnumValue) MarshalBSON() ([]byte, error) {
	return bson.Marshal(bson.M{"data": e.String()})
}

// UnmarshalBSON implements the bson.Unmarshaler interface.
func (e *stringEnumValue) UnmarshalBSON(data []byte) error {
	var m bson.M
	if err := bson.Unmarshal(data, &m); err != nil {
		return err
	}
	if data, ok := m["data"].(string); ok {
		if err := e.validateValueByKey(data); err != nil {
			return err
		}
		e.value = data
		return nil
	}
	return errors.New("couldn't unmarshal bson bytes string as enum.stringEnumValue")
}

// GobEncode implements the gob.GobEncoder interface.
func (e stringEnumValue) GobEncode() ([]byte, error) {
	return e.MarshalBinary()
}

// GobDecode implements the gob.GobDecoder interface.
func (e *stringEnumValue) GobDecode(data []byte) error {
	return e.UnmarshalBinary(data)
}

// MarshalBinary implements the encoding.BinaryMarshaler interface.
func (e stringEnumValue) MarshalBinary() ([]byte, error) {
	return []byte(e.value), nil
}

// UnmarshalBinary implements the encoding.BinaryUnmarshaler interface.
func (e *stringEnumValue) UnmarshalBinary(data []byte) error {
	value := string(data)
	if err := e.validateValueByKey(value); err != nil {
		return err
	}
	e.value = value
	return nil
}
```

### Implementing the enumeration 
On the other side, we have the concrete enum type we want. This type is mostly a public facade with 3 tasks:
1. Registering itself into the enumeration index of the local package on init()
2. Proxying calls to the codecs method implementations
3. Define its iterator and public types, values
Note that to achieve immutability the public members have to be func() Weekday types rather than just Weekday, as anyone outside the package could just take its reference and mutate it otherwise

```go
package enum

import (
	"fmt"
)

/**
 *
 * init: register enum to the internal index
 *
 **/
//nolint:gochecknoinits // enum has to register itself for unmarshaling at runtime
func init() {
	if _, ok := enumIndex[weekdayKey]; ok {
		panic(fmt.Sprintf("enum: enumeration %s is already registered", weekdayKey))
	}
	enumIndex[weekdayKey] = weekdayValueIndex
}

/**
 *
 * Type aliases and declarations
 *
 **/
type Weekday struct {
	stringEnumValue
}

func WeekdayFromValue(value string, ignoreCase bool) (Weekday, bool) {
	result, found := fromValue(value, ignoreCase, weekdayKey)
	return Weekday{result}, found
}

type weekdayList []func() Weekday

// weekdayEnum is a type and memory safe iterable enumeration of Weekday values
type weekdayEnum struct {
	weekdayList
}

func (e weekdayEnum) ForEach(f func(int, Weekday)) {
	for i, e := range e.weekdayList {
		f(i, e())
	}
}

func (e weekdayEnum) Len() int {
	return len(e.weekdayList)
}

/**
 *
 * Private value index, key
 *
 **/
var (
	weekdayValueIndex = valueIndex{
		"Friday":    {},
		"Monday":    {},
		"Saturday":  {},
		"Sunday":    {},
		"Thursday":  {},
		"Tuesday":   {},
		"Wednesday": {},
	}
	weekdayKey = "Weekday"
)

/**
 *
 * Public enumeration
 *
 **/
var (
	WeekdayFRIDAY    = func() Weekday { return Weekday{stringEnumValue{"Friday", weekdayKey}} }
	WeekdayMONDAY    = func() Weekday { return Weekday{stringEnumValue{"Monday", weekdayKey}} }
	WeekdaySATURDAY  = func() Weekday { return Weekday{stringEnumValue{"Saturday", weekdayKey}} }
	WeekdaySUNDAY    = func() Weekday { return Weekday{stringEnumValue{"Sunday", weekdayKey}} }
	WeekdayTHURSDAY  = func() Weekday { return Weekday{stringEnumValue{"Thursday", weekdayKey}} }
	WeekdayTUESDAY   = func() Weekday { return Weekday{stringEnumValue{"Tuesday", weekdayKey}} }
	WeekdayWEDNESDAY = func() Weekday { return Weekday{stringEnumValue{"Wednesday", weekdayKey}} }
	EnumWeekday = weekdayEnum{weekdayList{
		WeekdayFRIDAY,
		WeekdayMONDAY,
		WeekdaySATURDAY,
		WeekdaySUNDAY,
		WeekdayTHURSDAY,
		WeekdayTUESDAY,
		WeekdayWEDNESDAY,
	}}
)

/**
 *
 * Proxy methods for enum unmarshaling
 *
 **/
func (e *Weekday) UnmarshalJSON(data []byte) error {
	e.key = weekdayKey
	return e.stringEnumValue.UnmarshalJSON(data)
}
func (e *Weekday) UnmarshalText(text []byte) error {
	e.key = weekdayKey
	return e.stringEnumValue.UnmarshalText(text)
}
func (e *Weekday) UnmarshalBSON(data []byte) error {
	e.key = weekdayKey
	return e.stringEnumValue.UnmarshalBSON(data)
}
func (e *Weekday) UnmarshalBinary(data []byte) error {
	e.key = weekdayKey
	return e.stringEnumValue.UnmarshalBinary(data)
}
func (e *Weekday) GobDecode(data []byte) error {
	e.key = weekdayKey
	return e.stringEnumValue.GobDecode(data)
}
func (e *Weekday) Scan(raw interface{}) error {
	e.key = weekdayKey
	return e.stringEnumValue.Scan(raw)
}
```

### Using the generator 
I won't bore you with the details of code generation, and for that end I already made one for this enum implementation avilable at [github.com/lggomez/go-enum](https://github.com/lggomez/go-enum) documentation and examples included.
