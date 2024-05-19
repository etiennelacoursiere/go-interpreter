# Lexer

The first step is to transform the source code in something easier to work with. This mean, taking in the source code and transforming first into token, then into a AST ( Abstract syntax tree ).

The transformation from source code to token is called "Lexical analysis" or "Lexing" and is done by a Lexer. ( Also called Tokenizer or Scanner )

Token are small data structure that are easily recognizable and then fed to the parser.

Example:
Input: `let x = 5 + 5`
output from the Lexer:

```
[
	LET,
	IDENTIFIER("X"),
	EQUAL_SIGN,
	INTEGER(5),
	PLUS_SIGN,
	INTEGER(5),
	SEMICOLON
]
```

First, the Lexer is instantiated and takes in a input: `l := lexer.New("let x = 5 + 5")`

The function `New` of the Lexer package instantiate a new Lexer struct with the input and also calls the `readChar` method.

```go
package lexer

import "monkey/token"

type Lexer struct {
	input        string
	position     int  // current position in input (points to current char)
	readPosition int  // current reading position in input (after current char)
	ch           byte // current char under examination
}

func New(input string) *Lexer {
	l := &Lexer{input: input}
	l.readChar()

	return l
}
```

The readChar function first check if it has reach the end of the input. If so, it sets the `char` field of the lexer to 0 which is te ASCII character for "NUL". If it haven't reach the end of the input, it sets `char` to the next character by accessing `l.input[l.readPosition]`. Then it will set `position` to `readPosition` and increment `readPossition` . `readPosition` always points to the next position its going to read from next and `position` always points to the position it last read. Notes that uninitialized `int` in Go are set to 0, hence the call to `readChar` inside the `New` function to set the `readPosition` to 1.

```go
func (l *Lexer) readChar() {
	if l.readPosition >= len(l.input) {
		l.ch = 0
	} else {
		l.ch = l.input[l.readPosition]
	}

	l.position = l.readPosition
	l.readPosition += 1
}
```

Then, we need to call the `NextToken` method of the lexer until we reach the end of the file.

```go
l := lexer.New(line)

for tok := l.NextToken(); tok.Type != token.EOF; tok = l.NextToken() {
	// print to output
}
```

The `NexToken` function is essentially a big switch statement on the current `ch` value. This is where we transform parts of the input into tokens. Let say the current `ch` value is `;` . One condition of the switch statement will match on it and return a `Token` struct for that value.

```go
func (l *Lexer) NextToken() token.Token {
	var tok token.Token // instanciate a token

	// skipWhiteSpace is self explanatory. This function look at
	// the current ch and if it is  any sort of whitespaces it
	// skips to the next position by calling  l.readChar

	l.skipWhitespace()

	switch l.ch {
	case ';':
		tok = newToken(token.SEMICOLON, l.ch)
	case '(':
		tok = newToken(token.LPAREN, l.ch)
	case ')':
		tok = newToken(token.RPAREN, l.ch)
	case ',':
		tok = newToken(token.COMMA, l.ch)
	case '+':
		tok = newToken(token.PLUS, l.ch)
	case '{':
		tok = newToken(token.LBRACE, l.ch)
	case '}':
		tok = newToken(token.RBRACE, l.ch)

	// And so on.
	}

	// At the end, we need to call readChar to advances our positions
	l.readChar()

	// and we return the Token.
	return tok
}

```

The `newToken` function is just a little helpers to create token.

```go
func newToken(tokenType token.TokenType, ch byte) token.Token {
	return token.Token{Type: tokenType, Literal: string(ch)}
}
```

`Token` is a small module where we declare all of our possible token. Its struct has a `Type` field of `TokenType` which is a string, my guess as of why it is declared like this is if we somehow would want to change the tokenType it would be easier. To be fair, i'm not really sure why it was done that way.. It's second field `Litteral` is a string.

```go
type TokenType string

type Token struct {
	Type    TokenType
	Literal string
}

// Tokens
const (
	ILLEGAL = "ILLEGAL"
	EOF     = "EOF"

	// Identifiers + literals
	IDENTIFIER = "IDENTIFIER" // add, foobar, x, y, ...
	INT   = "INT"   // 1343456

	// Operators
	ASSIGN   = "="
	PLUS     = "+"
	MINUS    = "-"
	BANG     = "!"
	ASTERISK = "*"
	SLASH    = "/"
	LT       = "<"
	GT       = ">"
	EQ       = "=="
	NOT_EQ   = "!="

	// Delimiters
	COMMA     = ","
	SEMICOLON = ";"

	LPAREN = "("
	RPAREN = ")"
	LBRACE = "{"
	RBRACE = "}"

	// Keywords
	FUNCTION = "FUNCTION"
	LET      = "LET"
	IF       = "IF"
	ELSE     = "ELSE"
	TRUE     = "TRUE"
	FALSE    = "FALSE"
	RETURN   = "RETURN"
)
```

So far, we can read single char but what about keywords, variables and numbers? Let's add those. First we need a way to detect them. To do that, we'll need to write to helper functions `isLetter` and `isDigit` that takes in the `ch` we are looking at and return a boolean.

```go
// in the lexer module

func isLetter(ch byte) bool {
	return 'a' <= ch && ch <= 'z' || 'A' <= ch && ch <= 'Z' || ch == '_'
}

func isDigit(ch byte) bool {
	return '0' <= ch && ch <= '9'
}
```

If our `ch` is a letter or a number, we need to iterate on the input until it's not. Let's write the functions to do just that.

```go
// our function receive isNumber or isLetter
func (l *Lexer) readType(isType func(byte) bool) string {
	// we keep our starting position.
	position := l.position

   // we advance our readPosition until isType return false.
	for isType(l.ch) {
		l.readChar()
	}

   // this is a slice. This mean we return the content of
    // l.input from position ton l.position
	return l.input[position:l.position]
}
```

Let's put this all together by adding a default case in our `NexToken` function

```go
func (l *Lexer) NextToken() token.Token {
	var tok token.Token

	l.skipWhitespace()

	switch l.ch {
	// ... other cases below
	default:
		if isLetter(l.ch) {
			tok.Literal = l.readType(isLetter)
			tok.Type = token.LookupIdent(tok.Literal)

			return tok
		} else if isDigit(l.ch) {
			tok.Literal = l.readType(isDigit)
			tok.Type = token.INT

			return tok
		} else {
			tok = newToken(token.ILLEGAL, l.ch)
		}
	}

	l.readChar()

	return tok
}
```

`LookupIdent` is a function on the token module that look if the string passed is a keyword if so it return the `KEYWORD` token otherwise it return `IDENTIFIER`

```go
// token module
var keywords = map[string]TokenType{
	"fn":     FUNCTION,
	"let":    LET,
	"true":   TRUE,
	"false":  FALSE,
	"if":     IF,
	"else":   ELSE,
	"return": RETURN,
}

func LookupIdent(ident string) TokenType {
	if tok, ok := keywords[ident]; ok {
		return tok
	}

	return IDENTIFIER
}
```

We're almost finish. The last thing we need to detect is == and !=

To do that, we'll need too peek ahead in our input. Let's write a function to do this.

```go
func (l *Lexer) peekChar() byte {
	if l.readPosition >= len(l.input) {
		return 0
	} else {
		return l.input[l.readPosition]
	}
}
```

This function looks a lot like readChar, but unlike it, it does not advance the position and readPosition it only look ahead and return the nextChar.

Let's add this to our NextToken function.

```go
func (l *Lexer) NextToken() token.Token {
	var tok token.Token

	l.skipWhitespace()

	switch l.ch {
	case '=':
		if l.peekChar() == '=' {
			l.readChar()
			tok = token.Token{Type: token.EQ, Literal: "=="}
		} else {
			tok = newToken(token.ASSIGN, l.ch)
		}
	// ... other cases
	case '!':
		if l.peekChar() == '=' {
			l.readChar()
			tok = token.Token{Type: token.NOT_EQ, Literal: "!="}
		} else {
			tok = newToken(token.BANG, l.ch)
		}
	// ... other cases including default:
}
```

with that, we are done with our Lexer!

To use it from the command line, we'll need a REPL. Here's one. It's pretty self explanatory. ( I am too lazy to explain. If future me read this and doesn't understand. SORRY)

```go
package repl

import (
	"bufio"
	"fmt"
	"io"
	"monkey/lexer"
	"monkey/token"
)

const PROMPT = ">> "

func Start(in io.Reader, out io.Writer) {
	scanner := bufio.NewScanner(in)

	for {
		fmt.Fprintf(out, PROMPT)
		scanned := scanner.Scan()
		if !scanned {
			return
		}

		line := scanner.Text()
		l := lexer.New(line)

		for tok := l.NextToken(); tok.Type != token.EOF; tok = l.NextToken() {
			fmt.Fprintf(out, "%+v\n", tok)
		}
	}
}
```

and lastly, let's finaly create our `main.go`

```go
package main

import (
	"fmt"
	"monkey/repl"
	"os"
	"os/user"
)

func main() {
	user, err := user.Current()

	if err != nil {
		panic(err)
	}

	fmt.Printf(
		"Hello %s! This is the Monkey programming language!\n",
		user.Username,
	)

	fmt.Printf("Feel free to type in commands\n")

	repl.Start(os.Stdin, os.Stdout)
}
```

With that, we can now run `go run main`

```
λ go run main.go
Hello etiennelacoursiere! This is the Monkey programming language!
Feel free to type in commands
>> let add = fn(x, y) {x + y}
{Type:LET Literal:let}
{Type:IDENTIFIER Literal:add}
{Type:= Literal:=}
{Type:FUNCTION Literal:fn}
{Type:( Literal:(}
{Type:IDENTIFIER Literal:x}
{Type:, Literal:,}
{Type:IDENTIFIER Literal:y}
{Type:) Literal:)}
{Type:{ Literal:{}
{Type:IDENTIFIER Literal:x}
{Type:+ Literal:+}
{Type:IDENTIFIER Literal:y}
{Type:} Literal:}}
```
