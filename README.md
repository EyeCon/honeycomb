# 🍯 Honeycomb

Honeycomb is a parser combinator library written in pure Nim. It's designed to be simple, straightforward, and easy to expand, while relying on zero dependencies from outside of Nim's standard library.

Honeycomb was heavily inspired by the excellent Python library [parsy](https://github.com/python-parsy/parsy), as well as the existing but unmaintained [combparser](https://github.com/PMunch/combparser).

```nim
let
  parser  = ((s("Hello") | s("Greetings")) << c(',') << whitespace) & (regex(r"\w+") << c("!."))
  result1 = parser.parse("Hello, world!")
  result2 = parser.parse("Greetings, peasants.")

assert result1.kind  == success
assert result1.value == @["Hello", "world"]

assert result2.kind  == success
assert result2.value == @["Greetings", "peasants"]
```

Honeycomb supports the following key features:

- Predefined parsers and parser constructors for numerous basic parsing needs
- An extensive library of combinators with which to combine them
- Support for manually defining custom parsers / combinators
- Forward-declared parsers to support mutually recursive parser definitions

Once you've installed Honeycomb, you can use it in your project by importing it.

```nim
import honeycomb
```

## Usage

### Core Parser Constructors

Honeycomb provides several basic parser constructors:

```nim
# Parse a literal string
let parser = s("Hello")
let result = parser.parse("Hello, world!")
assert result.kind == success
assert result.value == "Hello"

# Parse a literal character
let charParser = c('H')
let charResult = charParser.parse("Hello")
assert charResult.value == 'H'

# Parse any character from a string, set, or range
let strSetParser = c("HG")       # Match 'H' or 'G'
let setParser = c({'G', 'H'})    # Match 'G' or 'H'
let rangeParser = c('G'..'H')    # Match 'G' through 'H'

# Parse using regular expressions
let regexParser = regex(r"H\w+")
let regexResult = regexParser.parse("Hello, world!")
assert regexResult.value == "Hello"

# No-op parser (always succeeds, consumes no input)
let nopParser = nop[string]()
let nopResult = nopParser.parse("Hello, world!")
assert nopResult.value == ""
assert nopResult.tail == "Hello, world!"
```

### Predefined Parsers

Several common parsers are provided out of the box:

```nim
# End of input
eof.parse("")  # succeeds
eof.parse("x")  # fails

# Any single character
anyChar.parse("Hello")  # succeeds with 'H'
anyChar.parse("")       # fails

# Whitespace (one or more)
whitespace.parse("  \t\n  text")  # succeeds with "  \t\n  "

# Letter, digit, alphanumeric
letter.parse("Hello")        # succeeds with 'H'
digit.parse("123")           # succeeds with '1'
alphanumeric.parse("a1")     # succeeds with 'a'
```

### Parser Combinators

Combinators allow you to combine parsers in powerful ways:

#### Sequencing

```nim
# Chain parsers together (& operator or chain())
let seqParser1 = s("Hello") & c(',') & whitespace & regex(r"\w+") & c("!")
let seqParser2 = chain(s("Hello"), c(','), whitespace, regex(r"\w+"), c('!'))
let result = seqParser1.parse("Hello, world!")
assert result.value == @["Hello", ",", " ", "world", "!"]

# Keep only right result (>> operator or then())
let thenParser1 = s("Hello, ") >> whitespace >> regex(r"\w+")
let thenParser2 = s("Hello, ").then(whitespace).then(regex(r"\w+"))
let thenResult = thenParser1.parse("Hello, world!")
assert thenResult.value == "world"

# Keep only left result (<< operator or skip())
let skipParser1 = s("Hello") << c(',') << whitespace << regex(r"\w+")
let skipParser2 = s("Hello").skip(c(',').asString).skip(whitespace).skip(regex(r"\w+"))
let skipResult = skipParser1.parse("Hello, world!")
assert skipResult.value == "Hello"
```

#### Alternatives

```nim
# Try one parser or another (| operator or oneOf())
let altParser1 = s("Hello") | s("Greetings")
let altParser2 = oneOf(s("Hello"), s("Greetings"))
let altResult = altParser1.parse("Hello, world!")
assert altResult.value == "Hello"
```

#### Repetition

```nim
# Exact number of times (* operator or times())
let exactParser1 = s("Hello ") * 3
let exactParser2 = s("Hello ").times(3)
let exactResult = exactParser1.parse("Hello Hello Hello ")
assert exactResult.value == @["Hello ", "Hello ", "Hello "]

# Range of times
let rangeParser = s("Hello ").times(3..5)
let rangeResult = rangeParser.parse("Hello Hello Hello Hello ")
assert rangeResult.value.len == 4

# At least n times
let atLeastParser = s("Hello ").atLeast(3)
let atLeastResult = atLeastParser.parse("Hello Hello Hello Hello ")
assert atLeastResult.value.len == 4

# At most n times
let atMostParser = s("Hello ").atMost(3) << eof
let atMostResult = atMostParser.parse("Hello Hello ")
assert atMostResult.value == @["Hello ", "Hello "]

# Zero or more times
let manyParser = s("Hello ").many()
let manyResult1 = manyParser.parse("Hello Hello Hello ")
let manyResult2 = manyParser.parse("")
assert manyResult2.value == newSeq[string]()

# Optional (once or none)
let optParser = s("Hello").optional()
let optResult1 = optParser.parse("Hello")
let optResult2 = optParser.parse("")
assert optResult2.value == ""
```

#### Lookahead

```nim
# Negative lookahead (! operator or negla())
# Succeeds if the given parser would fail, consumes no input
let neglaParser1 = !eof
let neglaParser2 = eof.negla
let neglaResult = neglaParser1.parse("Hello, world!")
assert neglaResult.kind == success
assert neglaResult.value == ""
assert neglaResult.tail == "Hello, world!"
```

#### Transformation

```nim
# Map over result
let mapParser = digit.atLeast(1).join.map(parseInt)
let mapResult = mapParser.parse("127")
assert mapResult.value == 127

# Map over each element in sequence
let mapEachParser = digit.asString.atLeast(1).mapEach(parseInt)
let mapEachResult = mapEachParser.parse("127")
assert mapEachResult.value == @[1, 2, 7]

# Replace with constant
let constParser = digit.atLeast(1).result("number found")
let constResult = constParser.parse("123")
assert constResult.value == "number found"

# Filter sequence elements
let filterParser = digit.asString.atLeast(1).mapEach(parseInt).filter(x => x >= 5)
let filterResult = filterParser.parse("0918273654")
assert filterResult.value == @[9, 8, 7, 6, 5]

# Validate with custom condition
let numParser = digit.atLeast(1).map(a => a.join().parseInt)
let validParser = numParser.validate(a => a < 500, "number smaller than 500")
let validResult = validParser.parse("323")
assert validResult.value == 323

let invalidResult = validParser.parse("874")
assert invalidResult.error == "[1:1] Expected number smaller than 500"
```

#### Sequence Operations

```nim
# Flatten nested sequences
let flattenParser = (digit & digit & digit).atLeast(1).flatten().join
let flattenResult = flattenParser.parse("127")
assert flattenResult.value == "127"

# Join sequence into string
let joinParser = s("Hello ").times(3).join
let joinResult = joinParser.parse("Hello Hello Hello ")
assert joinResult.value == "Hello Hello Hello "

# Join with delimiter
let joinDelimParser = (s("Hello") << whitespace).times(3).join(", ")
let joinDelimResult = joinDelimParser.parse("Hello Hello Hello ")
assert joinDelimResult.value == "Hello, Hello, Hello"
```

### Forward Declarations

For mutually recursive parsers:

```nim
var parser1 = fwdcl[string]()
let parser2 = (s("Hello, ") & parser1 & c('!'))

parser1.become(s("world"))

let result = parser2.parse("Hello, world!")
assert result.value == @["Hello, ", "world", "!"]
```

### Custom Parsers

Create your own parsers with `createParser`:

```nim
let parser = createParser(string):
  if input.len < 10:
    return fail(input, @["at least 10 characters"], input)
  return succeed(input, input[0..9], input[10..^1])

let result = parser.parse("Hello, world!")
assert result.value == "Hello, wor"
```

### Error Handling

```nim
let parser = s("Hello, world!")
let result = parser.parse("Greetings")

# Check result
if result.kind == failure:
  echo result.error  # "[1:1] Expected 'Hello, world!'"

# Or raise exception on failure
try:
  result.raiseIfFailed
except ParseError as e:
  echo e.msg
```

### Integration Example: Receipt Parser

```nim
let
  receiptEntry = (regex(r"[\w\s]+") << s(": ")) &
                 (digit.atLeast(1) << c('.')).join &
                 digit.times(2).join << c('\n').optional()
  receiptParser = receiptEntry.map(x => (x[0], parseFloat("$1.$2" % x[1..^1]))).atLeast(1)
  testReceipt = "Milk: 4.00\nEggs: 15.99\nCool robot: 69.99"
let result = receiptParser.parse(testReceipt)

assert result.value == @[("Milk", 4.00), ("Eggs", 15.99), ("Cool robot", 69.99)]
```

For a more in-depth conceptual look at parser combinators in general, you can try these resources:

- [Antoine Leblanc: Parser Combinators Walkthrough](https://hasura.io/blog/parser-combinators-walkthrough/) (article, Haskell)<br>Gives a very thorough and in-depth explanation of the basics of parser combinators.
- [Stephen Gutekanst: Zig, Parser Combinators - and Why They're Awesome](https://serokell.io/blog/parser-combinators-in-elixir) (article, Zig)<br>A more detailed look at implementing some basic parser combinators.
- [Graham Hutton et al.: Monadic Parser Combinators](https://www.cs.nott.ac.uk/~pszgmh/monparsing.pdf) (whitepaper, Haskell)<br>An analysis of some of the type theory behind parser combinators.
- [Scott Wlaschin: Understanding Parser Combinators](https://fsharpforfunandprofit.com/series/understanding-parser-combinators/) (article series, F#)<br>A multipart series starting from the basics and ending with the implementation of a full JSON parser.
- [Yassine Elouafi: Introduction to Parser Combinators](https://gist.github.com/yelouafi/556e5159e869952335e01f6b473c4ec1) (article, Javascript)<br>A straightforward look at implementing simple parser combinators.
- [Computerphile: Functional Parsing](https://www.youtube.com/watch?v=dDtZLm7HIJs) (video, Haskell)<br>A high-level overview of the basics of parser combinators in video format.
- [Li Haoyi: Easy Parsing with Parser Combinators](https://www.lihaoyi.com/post/EasyParsingwithParserCombinators.html) (article, Scala)<br>A detailed explanation of how to use parser combinators in some more complex contexts.

Honeycomb has an extensive and expanding suite of unit tests, which can be found in [`tests/test.nim`](./tests/test.nim). You can run the tests with:
```bash
nimble test
```

To generate a local copy of the documentation from Honeycomb's code, you can use the following command. Note that the `docs` folder is intentionally `.gitignore`d and should not be committed; when your changes are merged into the `master` branch, an automated process will regenerate the documentation on the `docs` branch.
```bash
nimble gendocs
```
