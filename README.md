# Compiler for the C-- language

- [Compiler for the C-- language](#compiler-for-the-c---language)
    - [Specification](#specification)
    - [Lexical Rules](#lexical-rules)
    - [Syntax Rules (Grammar Productions)](#syntax-rules-grammar-productions)
        - [Operator associativity and Precedences](#operator-associativity-and-precedences)
    - [Abstract syntax tree](#abstract-syntax-tree)
    - [Typing Rules (and Internal Datastructure)](#typing-rules-and-internal-datastructure)
        - [Symbol Table](#symbol-table)
        - [Symbol Table Operations](#symbol-table-operations)
        - [Type consistency](#type-consistency)
    - [References](#references)

## Specification
This specification is taken from the [C-- Language Specification](https://www2.cs.arizona.edu/~debray/Teaching/CSc453/DOCS/cminusminusspec.html), and reiterated here convenience.

The extended BNF notation is used,

- Alternatives are separated by vertical bars: i.e., `a | b` stands for "a or b".
- Square brackets indicate optionality: `'[ a ]'` stands for an optional a, i.e., `a | epsilon` (here, epsilon refers to the empty sequence).
- Curly braces indicate repetition: `'{ a }'` stands for `epsilon | a | aa | aaa | ...`

## Lexical Rules

| Production |  | Rules |
|------------|--|-------|
| _letter_ | :== | `a | b | ... | z | A | B | ... | Z` |
| _digit_ | :== | `0 | 1 | ... | 9` | 
| __id__ | :== | _letter_{ _letter_ \| _digit_ \| _ } |
| __intcon__ | :== | _digit_{ _digit_ } |
| __charcon__ | :== | `'ch' | '\n' | '\0'`<sup>1</sup> |
| __stringcon__ | :== | "ch"<sup>2</sup> |
| _Comments_ |  |Comments are as in C, i.e. a sequence of characters preceded by /* and followed by */, and not containing any occurrence of */. |

<sup>1</sup> where _ch_ denotes any printable ASCII character, as specified by `isprint()`, other than __\\__ (backslash) and __'__ (single quote).

<sup>2</sup> where _ch_ denotes any printable ASCII character (as specified by `isprint())` other than " (double quotes) and the newline 

## Syntax Rules (Grammar Productions)

Nonterminals are shown in italics; terminals are shown in boldface, and sometimes enclosed within quotes for clarity.

| Production |  | Rules |
|------------|--|-------|
| _prog_ | : | { _dcl_ '__;__' \| _func_} |
| _dcl_ | : <br> \| <br> \| | _type var\_decl_ { ',' _var\_decl_ } <br> [ __extern__ ] _type_ __id__ '__(__' _parm\_types_ '__)__' { ',' __id__ '__(__' _parm\_types_ '__)__' } <br> [ __extern__ ] __void__ __id__ '__(__' _parm\_types_ '__)__' { ',' __id__ '__(__' _parm\_types_ '__)__' } |
| _var\_decl_ | : | __id__ [ '__[__' __intcon__ '__]__' ] |
| _type_ | : <br> \| | __char__ <br> __int__ |
| _parm\_types_ | : <br> \| | __void__ <br> _type_ __id__ [ '__[__' '__]__' ] {',' _type_ __id__ [ '__[__' '__]__' ]} |
| _func_ | : <br> \| | _type_ __id__ '__(__' _parm\_types_ '__)__' '__{__' { _type_ _var\_decl_ { ',' _var\_decl_ } '__;__' } { _stmt_ } '__}__' <br> __void__ __id__ '__(__' _parm\_types_ '__)__' '__{__' { _type_ _var\_decl_ { ',' _var\_decl_ } '__;__' } { _stmt_ } '__}__' |
| _stmt_ | : <br> \| <br> \| <br> \| <br> \| <br> \| <br> \| <br> \| | __if__ '__(__' _expr_ '__)__' _stmt_ [ __else__ _stmt_ ] <br> __while__ '__(__' _expr_ '__)__' _stmt_ <br> __for__ '__(__' [ _assg_ ] '__;__' [ _expr_ ] '__;__' [ _assg_ ] '__)__' _stmt_ <br> __return__ [ _expr_ ] '__;__' <br> _assg_ '__;__' <br> __id__ '__(__' [ _expr_ { '__,__' _expr_ } ] <br> '__{__' { _stmt_ } '__}__' <br> '__;__'|
| _assg_ | : | __id__ [ '__[__' _expr_ '__]__' ] = _expr_ |
| _expr_ | : <br> \| <br> \| <br> \| <br> \| <br> \| <br> \| <br> \| <br> \| <br> \| | '__-__' _expr_ <br> '__!__' _expr_ <br> _expr_ _binop_ _expr_ <br> _expr_ _relop_ _expr_ <br> _expr_ _logical\_op_ _expr_ <br> __id__ [ '__(__' [ _expr_ { ',' _expr_ } ] '__)__' \| '__[__' _expr_ '__]__' ] <br> '__(__' _expr_ '__)__' <br> __intcon__ <br> __charcon__ <br> __stringcon__ |
| _binop_ | : <br> \| <br> \| <br> \| | __+__ <br> __-__ <br> __*__ <br> __/__ |
| _relop_ | : <br> \| <br> \| <br> \| <br> \| <br> \| | __==__ <br> __!=__ <br> __<=__ <br> __<__ <br> __>=__ <br> __>__ |
| _logical\_op_ | : <br> \| | __&&__ <br> __\|\|__ |

### Operator associativity and Precedences

| Operator | Associativity |
|------------|------|
| !, - (unary) | right to left |
| *, / | left to right |
| +, - (binary) | left to right |
| <, <=, > >= | left to right |
| ==, != | left to right |
| && | left to right |
| \|\| | left to right |

## Abstract syntax tree

We will model the abstract syntax tree as a Haskell data type,

```haskell
-- | Aliases for our literals.
type Intcon = Int32
type Charcon = Word8
type Stringcon = ByteString
type Identifier = String
type Extern = Maybe Bool

-- | The type production
data AType
  = TypeChar 
  | TypeInt

-- | the binop production
data BinOperator
  = Add
  | Sub
  | Mul
  | Div

 -- | The relop production
data RelOperator
  = Equal
  | NotEqual
  | LessThanEqual
  | LessThan
  | GreaterThanEqual
  | GreaterThan

-- | The logical_op production
data LogicalOperator
  = And
  | Or

-- | The expr production
data Expr 
  = Neg Expr
  | Exclamation Expr
  | BinOp BinOperator Expr Expr
  | RelOp RelOperator Expr Expr
  | LogicalOp LogicalOperator Expr Expr
  | BlockExpr Identifier (Maybe [Expr])
  | Parens Expr
  | LitInt Intcon
  | LitChar Charcon
  | LitString Stringcon

-- | The var_decl production
data Variable  
  = VarDecl Identifier Variable
  | VarInt Identifier Intcon Variable
  | NonVar

-- | The parm_types production
data ParameterTypes
  = VoidParameter
  | Parameter AType Identifier ParameterTypes

-- | The assg production
data Assignment 
  = Assign Identifier (Maybe Expr) Expr

-- | The stmt production
data Statement 
  = If Expr Statement Statement
  | While Expr Statement
  | For Assignment Expr Assignment Statement
  | StmtAssignment Assignment
  | StmtExpr Identifier [Expr]
  | Brackets Statement
  | SemiColon

-- | The func production
data Func
  = FuncDef AType Identifier ParameterTypes AType Variable Statement
  | FuncVoidDef Identifier ParameterTypes AType Variable Statement

-- | The dcl production
data Declaration
  = DeclVariableDeclaration AType Variable
  | DeclFunc Extern AType [(Identifier, ParameterTypes)]
  | DeclVoidFunc Extern [(Identifier, ParameterTypes)]

-- | The prog production
data Prog
  = Decl 
  | Func
```

From the AST it will be possible to rebuild the source code (although superfluous newlines and whitespace etc might be stripped).

## Typing Rules (and Internal Datastructure)

For the full information, see the [C-- Language Specification](https://www2.cs.arizona.edu/~debray/Teaching/CSc453/DOCS/cminusminusspec.html) for this part.

We need to keep track of several things as we parse C--.

| Keep track of | Reason |
|---------------|--------|
| Scope | a) An identifier can only be _declared_ at most __once__ globally and __once__ locally <br> b) a function may have at most __one__ prototype i.e. defined at most __once__ <br> c) The formal parameters of a function have __scope local__ to that function |
| Array size | An array __must__ have a non-negative size |
| Prototypes <br> Functions | a) The prototype, if present, must __precede the definition__ of the function <br> b) The __types of the prototype must match the types of its definition__ along with parameters etc <br> c) If a __function takes no parameters__, its __prototype__ must indicate this by using the __keyword void__ in place of the formal parameters <br> d) A function whose prototype is preceded by the __keyword extern__ must __not be defined__ in the program being processed |
| Identifiers | a) An identifier can occur at most __once__ in the list of formal parameters in a function definition <br> b) If an identifier is declared to have scope local to a function, then all uses of that identifier within that function refer to this local entity; if not, but it's defined globally, then it will refer to the global instance |
| Variables | Variables must be __declared before they are used__ |

### Symbol Table

We have some simple requiresments for our table,

- Insertion is only done once
- Lookup is done many times
    - We need fast lookups!

With the above in mind we settle on using a a `Map`, which in Haskell is based on _size balanced_ binrary trees. This means we have worst case times of O(_log n_) lookup and O(_log n_) insertion, while offering a nice and simple interface to the datastructure.

First an overview of all the components are presented, in the table below, and then each symbol type is explained further using Haskell code examples of the (possible) implementations of these.

| | Symbol Name | Type | Scope | Attributes | Location |
|--|-------------|------|-------|------------|---------|
| __Type name__ | `SymbolName` | `SymbolType` <br>  | `SymbolScope` <br>  | `SymbolAttribute` | `SymbolLocation` |
| __Explanation__ | The name of the symbol,<br> e.g. `foobar`, `i`, `sum`. | What type of symbol are we are dealing with, a function, a declaration, an identifier etc. | What our scope is: either global or local to a specific item (e.g. function). More specifically, a function will keep a nested `SymbolTable` which contains the items inside the scope of the function. | The symbol attributes contain additional information about a symbol, such as function parameters, return types etc. | The location of the symbol, such as file and line:column number |

__Symbol Table__ is defined as,
```haskell
-- | The symbol table data is a product type of all the fields in the symbol table.
data SymbolTableData = SymbolTableData SymbolType SymbolScope SymbolAttribute

-- | An alias for an optional symbol table parent, `Just SymbolTable`, or none (i.e. 
-- the global scope), `Nothing`.
type SymbolTableParent = Maybe SymbolTable

-- | The symbol table consist of an optional parent symbol table, and a Map containing
-- the symbol names as key and the data as values.
data SymbolTable = SymbolTable SymbolTableParent (Map SymbolName SymbolTableData)
```

Our symbol table then becomes a tree of scopes. For example, the following code,

```c
int main(void) {
  int sumValue, 
  sum(...);
  storeResult(sumValue);
  processFile();
}

int sum(int count, int *value) {
  // ...
}

void storeResult(int value) {
  // ...
}

int processFile() {
  readFile();
  // ...
}
```

would give a the following tree,

```
                Global Symbol Table (main)
                  /      |       \
                 /       |        \
  Symbol Table (sum)     |      Symbol Table (processFile)
                         |                 \
          Symbol Table (storeResult)        \
                                     Symbol Table (readFile)
```

__Symbol Name__ is defined as,
```haskell
-- | The symbol name is simply a string representing the name.
newtype SymbolName = SymbolName String
```

__Type__ is defined as,
```haskell
-- | The possible types our symbols can have.
data SymbolType 
  = Prototype 
  | ExternPrototype
  | Function 
  | Identifier
```

__Scope__ is defined as,
```haskell
-- | A scope is either global or local to a specific item, i.e. function or block. The local scope
-- keeps a track of its own symbol table.
data SymbolScope 
  = Global 
  | Local SymbolTable
```

__Attributes__ are defined as,
```haskell
-- | The basic types of our literals (reiterated here from the AST section).
data ValueType 
  = ValueInt Int32 -- int is 32 bit
  | ValueChar Word8 -- a char is 8 bits
  | ValueString ByteString -- a string is an array of chars (`ByteString` is a list of `Word8`)
  | ValueIntArray (Array i Int32) –- an array of ints
  | ValueReturnVoid -- the void return value

-- | The identifier name is simply a string representing the name.
newtype IdentifierName = IdentifierName String

-- | Function parameters, can either be none (`VoidParam`) or a recursive amount (`Param`). 
-- E.g. a function with 2 parameters will look like:
--     `Param IdentifierName ValueType (Param IdentifierName ValueType EmptyParam)`.
data Parameter 
  = Param IdentifierName ValueType Parameter 
  | EmptyParam

-- | The attributes a symbol can have.
data SymbolAttribute 
  = PrototypeAttr ValueType Parameter -- a return type and parameter information
  | FunctionAttr ValueType Parameter -- a return type and parameter information
  | IdentifierAttr ValueType -- the type of the identifier
```

__Location__ is defined as,
```haskell
-- | Our location information is simply a record of the filename, line number and column number.
data SymbolLocation = SymbolLocation 
  { locFilename :: String
  , locLineNumber :: Int
  , locColumnNumber :: Int
  }
```

NOTE: To the unfamiliar, `newtypes` in Haskell only exist at compile time and are erased at runtime only where they become identical to their content. That is `ArraySize Int` is just an `Int` at runtime. This gives us no runtime overhead, but allows us more compile time safety.

### Symbol Table Operations

The interface of the symbol table looks as follows,

- `enterScope :: SymbolTable` starts a new nested scope
- `findSymbol scope s :: SymbolTable -> SymbolName -> Maybe SymbolTableData` looks for a symbol, `s` going up the symboltable, `scope` and returns the `SymbolTableData` for the symbol if it exists, else returns `Nothing`
- `checkScope scope s :: SymbolTable -> SymbolName -> Bool` checks if `s` is in the immediate scope (to rule out duplicate declarations)
- `addSymbol scope s :: SymbolTable -> SymbolName -> SymbolTable` adds a symbol, `s`,  to the symbol table, `scope`,  and returns the updated table
- `exitScope :: SymbolTable` exits the scope and returns the parent scope

### Type consistency

We denote compatibility here with 🔛,

1. __int__ 🔛 __int__, __char__ 🔛 __char__
2. __int__ 🔛 __char__, __char__ 🔛 __int__
3. Array of __int__ 🔛 Array of __int__, and same for __char__
4. Everything else is not compatible

## References

- [Symbol table design (Compiler Construction)](https://www.slideshare.net/Tech_MX/symbol-table-design-compiler-construction)
