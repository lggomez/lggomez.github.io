{
    "title": "Fixing and extending jwt-go without forking",
    "description": "How to keep using jwt-go without forking or deprecating it in your project",
    "summary": "How to keep using jwt-go without forking or deprecating it in your project",
    "tags": ["Golang", "JWT"],
    "date": "2021-01-15T18:56:44-03:00",
    "categories": ["Programming", "Backend", "Tutorial", "Security", "Bug", "Open Source"],
    "type": "publication",
    "weight": 0,
    "publishdate": null,
    "expirydate": null,
    "aliases": null,
    "slug": "fixing-jwt-go-without-forking",
    "url": ""
}

Drama isn't usual in the go community as far as I'm concerned and I wouldn't call this incident itself a drama but it reminds me of instances that happen more than often on other technology stacks __[citation omitted]__.

Security utilities are no joke, and no one in their sane mind would go about writing a custom RSA implementation in C++ for, say, their bank employer. People dedicate their entire careers writing this stuff and any modern platform or programming language has a security toolset for developing applications, and, otherwise, the open source solutions which are battle-tested by the community offer more guarantee that a one-man solution. Which brings us to today's entry.

Go contains several official packages related to security but, to my surprise when I looked around, there was no canonical JWT implementation. I needed to do some token verification and introspection, and I decided to go with https://github.com/dgrijalva/jwt-go as it seems to be one of the most popular packages  with an API that looked pretty ergonomic for my use case.

However, as of today it still has a security flaw: https://github.com/dgrijalva/jwt-go/issues/428
The repo is not maintained anymore (altough is stable aside from that issue) and the owner is nowhere to be found, or neither wants to be found. In the context of dependency management, this problem brings up some questions:
* Is it viable to use a fork at the cost of fragmentation?
* What happens to versions which were already audited by some process? a fork and new owner implies a full re-audit of the dependency?
* Why is google in the [list](https://pkg.go.dev/github.com/dgrijalva/jwt-go?tab=importedby) of users of this package and didn't bother to do a fork, or better yet, an official package like in the case of oauth?

Really interesting questions but for which I didn't have the answers, or the time to find them as I wanted to finish my implementation. Moving to another, more interesting question, how much of the package would I need to copy or re-implement in order to fix this without doing a complete fork or having to move on to another package?

## Not so tight, not so loose coupling 
The basic workflow goes like this:

```go
import (
    jwtgo "https://github.com/dgrijalva/jwt-go"
)

claims := jwtgo.MapClaims{}

// Traverse JWKS and validate/match by kid (ideally)
for _, key := range publicKeys {
    token, err := jwtgo.ParseWithClaims(rawStringJWT, &claims, func(t *jwtgo.Token) (interface{}, error) {
        // Revalidate signing method to prevent bypass attacks
        // ...
        
        return key.PublicKey, nil // PublicKey is my *rsa.PublicKey instance containing the valid key to verify the token
    })
}
```

The issue is in the `jwtgo.MapClaims` type, which contains the bug, but wait: the `ParseWithClaims` method expects a `Claims` parameter, which is an interface:

```go
type Claims interface {
	Valid() error
}
```

## Extending 
The So, knowing this, we can implement our custom type, namely a copy of the fixed implementation of the jwtgo.MapClaims type (for reference, see https://github.com/dgrijalva/jwt-go/pull/385). This type complies with Claims interface and can be extended to advanced validations, like required and optional claims:

```go
type MyFixedClaims = jwtgoclaims.MapClaims // alias to subpackage containing the fixed claims implementation
// for clarity and easier replacement later on if the author
// is alive and decides to merge the pull request

type ClaimValue struct {
	Value string
	Required bool
}

type ClaimsVerificationValues map[string]ClaimValue

func VerifyMyClaims(myClaims MyFixedClaims, verifier ClaimsVerificationValues) error {
	for k, v := range verifier {
		if err := verifyValue(myClaims, v.Value, k, v.Required); err != nil {
			return err
		}
	}
}

func verifyValue(c MyFixedClaims, expected, key string, required bool) error {
	val, found := c[key]
	actual, ok := val.(string)
	if !(found && ok) && required {
		return fmt.Errorf("%v verification failed", key)
	}
	
	
	if required {
		if !ok || actual == "" {
			return fmt.Errorf("%v verification failed", key)
		}
	} else if actual == "" {
		return nil
	}
	
	if actual != expected {
		return fmt.Errorf("%v verification failed", key)
	}
	
	return nil
}
```

I'd like to think there is a nicer way of doing the last method, so I will leave that up to the reader.

## Wrapping up 
With that, our initial code piece would end up like this:

```go
import (
    jwtgo "https://github.com/dgrijalva/jwt-go"
)

claims := MyFixedClaims{}

verificationValues := ClaimsVerificationValues{
	"customclaim": ClaimValue{Value:"foo", Required: true},
	"konamicode": ClaimValue{Value:"↑ ↑ ↓ ↓ ← → ← → B A", Required: false}
}

// Traverse JWKS and validate/match by kid (ideally)
for _, key := range publicKeys {
    token, err := jwtgo.ParseWithClaims(rawStringJWT, &claims, func(t *jwtgo.Token) (interface{}, error) {
        // Revalidate signing method to prevent bypass attacks
        // ...
        
        if err := VerifyMyClaims(claims, verificationValues); err != nil {
        	return nil, err
        }
        
        // ...
        
        return key.PublicKey, nil // PublicKey is my *rsa.PublicKey instance containing the valid key to verify the token
    })
}
```

Drama is avoided. Pending refactor is all that remains.

**PS:** Be nice and don´t forget to include the licenses belonging to the code you fork when applicable.
