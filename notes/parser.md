# Parser

The parser job is to take an input ( source code ) and turn it into an Abstract Syntax Tree or another data structure that represent the source code.

Here is a small example of what a parser would do:

```js
if (3 * 5 > 10) {
  return "hello";
} else {
  return "goodbye";
}
```

The parser would turn this into a tree that could looks like this:

```js
{
  type: "if-statement",
  condition: {
    type: "operator-expression",
    operator: ">",
    left: {
      type: "operator-expression",
      operator: "*",
      left: { type: "integer-literal", value: 3 },
      right: { type: "integer-literal", value: 5 }
    },
    right: { type: "integer-literal", value: 10 }
  },
  consequence: {
    type: "return-statement",
    returnValue: { type: "string-literal", value: "hello" }
  },
  alternative: {
    type: "return-statement",
    returnValue: { type: "string-literal", value: "goodbye" }
  }
}
```

There are many type of parsers but the parser we are writting for the monkey programming language is a Recursive Descent Parser. More precisely it's a Top Down Operator Precedence. ( Sometime called Pratt Parser ).

## Parsing let statements

A let statement bind a value to a given name. `let x = 5;` bind the value `5` to the variable `x`.

We'll call the left part of the let statement an identifier and the right part and expression.

```
let <identifier> = <expression>;
```

The expression could be just a value like `5` or a more complex expression like `5 + 5`, or even a function. For now, let's focus on the simple value.

### Difference between statements and expression.

Expression produce values while statement don't. The statement `let x = 5;` doesn't produce a value, while `5` does. ( The value it procuces is `5` ). The statement `return 5` doesn't produce a value, but the expression `add(5, 5)` does. This distinction changes depending of who you ask but this is what we will do in monkey.

## The AST

With that in mind, we can start to define our AST. It need two nodes: `statements` and `expressions`.

```go
package ast

type Node interface {
	TokenLiteral() string
}

type Statement interface {
	Node
	statementNode()
}

type Expression interface {
	Node
	expressionNode()
}
```

Every node in the AST will need to implement the `Node` interface. Meaning it has to provide a `TokenLiteral` method that return the literal value of the token that created the node. Token literral wil be used for debugging and testing only.

The `Statement` interface embeds the `Node` interface and add a `statementNode` method. This method is just a dummy. It is not required but it helps guide the Go compiler. The same goes for the `Expression` interface.

The first node will be the `Program`. This is the root node of our AST. It contains a series of statements in `Program.Statements` which is a slice of nodes that implement the `Statement` interface.

```go
// ast/ast.go

type Program struct {
    Statements []Statement
}

func (p *Program) TokenLiteral() string {
    if len(p.Statements) > 0 {
        return p.Statements[0].TokenLiteral()
    } else {
        return ""
    }
}
```

Next, we'll create the `LetStatement` node. Here is a let statement. `let x = 5;`. The node will need a Name field, for the name of the variable. It will also need a field that point to the `expression` on the right side of the equal sign. It needs to be able to point to any `expression`. It also need to keep track of the `token` the node is associated with.

```go
// ast/ast.go

import "monkey/token"

// [...]

type LetStatement struct {
	Token token.Token // the token.LET token
	Name  *Identifier
	Value Expression
}

func (ls *LetStatement) statementNode()       {}
func (ls *LetStatement) TokenLiteral() string { return ls.Token.Literal }

type Identifier struct {
	Token token.Token // the token.IDENTIFIER token
	Value string
}

func (i *Identifier) expressionNode()      {}
func (i *Identifier) TokenLiteral() string { return i.Token.Literal }
```

The `LetStatement` node has a `Token` field that hold the `token.LET` token. It also has a `Name` field that hold a pointer to an `Identifier` node. The `Value` field hold an `Expression` node interface. ( Hence why this is not a pointer to a specific node ( i think ) ).

The `Identifier` Node implement the `Expression` interface. This is to keep things simple because in some case, the `Identifier` could produce a value. i.e. `let x = identifierProducingAValue;

Now that we have the basic building blocks of our AST, we can start working on our parser.

The parser will have three fields. The first is a pointer to an instance of our lexer on which we call `NextToken()` repeatedly to get the next token in the inpit. The two other fields are `curToken` and `peekToken`. They are the same as `position` and `readPosition` in the lexer. We need both in order to decide what to do next. For example take a single line only containing `5`. Then `curToken` is a `token.Int` and we need `peekToken` to know if we are at the end of the line or at the start of an arithmetic expression.

```go
// parser/parser.go

package parser

import (
    "monkey/ast"
    "monkey/lexer"
    "monkey/token"
)

type Parser struct {
    l *lexer.Lexer

    curToken  token.Token
    peekToken token.Token
}

func New(l *lexer.Lexer) *Parser {
    p := &Parser{l: l}

    // Read two tokens, so curToken and peekToken are both set
    p.nextToken()
    p.nextToken()

    return p
}

func (p *Parser) nextToken() {
    p.curToken = p.peekToken
    p.peekToken = p.l.NextToken()
}
```

Next we add the `ParseProgram` method. This method construct the root node our ast. `*ast.Program`. It then iterrate over each token in the input until it finds a `token.EOF` token. For each token it call `parseStatement` and append the result to the `Program.Statements` slice. It then returns the `*ast.Program`

```go
// parser/parser.go

func (p *Parser) ParseProgram() *ast.Program {
    program := &ast.Program{}
    program.Statements = []ast.Statement{}

    for p.curToken.Type != token.EOF {
        stmt := p.parseStatement()
        if stmt != nil {
            program.Statements = append(program.Statements, stmt)
        }
        p.nextToken()
    }

    return program
}
```

The `parseStatement` method is a helper method that switch over the `p.curToken.Type` and decide what parsing method to call. For now, we only have the `let` statement so we only need to call `parseLetStatement`.

```go
// parser/parser.go

func (p *Parser) parseStatement() ast.Statement {
    switch p.curToken.Type {
    case token.LET:
        return p.parseLetStatement()
    default:
        return nil
    }
}
```

The `parseLetStatement` method first check if the next token is an `IDENTIFIER` token. If not, it return `nil`. It then check if the next token is an `ASSIGN` token. If not, it return `nil`. It then would call `parseExpression` which we don't have yet to parse the expression on the right side of the equal sign. For now, we just skip the expression and call `nextToken` until we find a `SEMICOLON` token. Then we return the statement. To know what the helpers method like `expectPeek` does, look at the code in the repository.

```go
// parser/parser.go

func (p *Parser) parseLetStatement() *ast.LetStatement {
    stmt := &ast.LetStatement{Token: p.curToken}

    if !p.expectPeek(token.IDENTIFIER) {
        return nil
    }

    stmt.Name = &ast.Identifier{Token: p.curToken, Value: p.curToken.Literal}

    if !p.expectPeek(token.ASSIGN) {
        return nil
    }

    // for now we just skip the expression and call nextToken until we find a semi colon.

    for !p.curTokenIs(token.SEMICOLON) {
        p.nextToken()
    }

    return stmt
}
```
