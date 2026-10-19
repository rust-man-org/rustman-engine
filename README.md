# rustman-engine

A tiny, deliberately simple scripting language for API pre-request and test
scripts, interpreted in pure Rust. No JavaScript, no QuickJS/V8, no `pm.*`
API — just a small lexer, parser, and tree-walking evaluator.

It is built for [Rustman](https://github.com/rust-man-org/rustman), where
scripts are meant to be *generated* — by an AI, from a plain-English
description of what the user wants. So the language is kept small on purpose:
a model gets it right on the first try, and scripts cannot loop, cannot
define functions, and cannot touch the network or the filesystem. They can
only ask the host for things through the built-ins below.

## A script

```rust
let token = env("access_token")
if token != "" {
    set_header("Authorization", "Bearer " + token)
}
test("status is 200", response.status == 200)
test("no stack trace leaked", !contains(response.text(), "panicked at"))
```

## Embedding

```rust
use rustman_engine::{run, HostInput};

let outcome = run(source, HostInput {
    env_vars: vec![("access_token".into(), "abc".into())],
    ..Default::default()
})?;

for effect in outcome.effects {
    // apply to the host
}
```

`run` returns the effects a script asked for. The engine never touches host
state itself — the host applies `RunOutcome::effects`, which keeps this crate
a pure function of its input and easy to test.

### Effects

| Effect                  | Produced by   | Host meaning                               |
| ----------------------- | ------------- | ------------------------------------------ |
| `SetEnv(key, value)`    | `set_env`     | write an environment variable              |
| `SetHeader(key, value)` | `set_header`  | set a request header                       |
| `SetBody(text)`         | `set_body`    | replace the outgoing request body          |
| `Test { name, passed }` | `test`        | record an assertion result                 |
| `Log(text)`             | `print`       | a console line for debugging               |

### Host input

| Field      | What it carries                                            |
| ---------- | ---------------------------------------------------------- |
| `env_vars` | environment variables                                      |
| `headers`  | request headers (pre-request) or response headers (tests)  |
| `cookies`  | cookies                                                    |
| `body`     | the outgoing request body (available in both slots)        |
| `url`      | the outgoing request URL (available in both slots)         |
| `response` | `{ status, body }` — `None` for pre-request scripts        |

## The language

Statements: `let`, `if`/`else`, and expressions. Comments start with `//`.
Semicolons are optional.

```rust
let status = response.status
if status >= 200 && status < 300 {
    test("ok range", true)
} else {
    test("ok range", false)
}
```

- **Values**: `null`, `true`/`false`, numbers, strings, arrays, objects.
- **Operators**: `+` `-` `==` `!=` `<` `>` `<=` `>=` `&&` `||` `!`, field
  access with `.`, and calls. `+` adds numbers and concatenates anything else.
- **Truthiness**: `null`, `0`, `""`, `[]`, and `false` are falsy; objects are
  always truthy.
- **No loops, no user-defined functions, no regex.** Deliberate limits, not a
  roadmap accident: every addition also has to be explained in an LLM prompt.

## Built-ins

| Function                          | Notes                                                              |
| --------------------------------- | ------------------------------------------------------------------ |
| `env(key)`                        | missing variable returns `""`, not `null`                          |
| `set_env(key, value)`             | value may be any type; objects/arrays are JSON-stringified         |
| `header(key)`                     | case-insensitive lookup; missing returns `""`                      |
| `headers()`                       | every header as an object                                          |
| `set_header(key, value)`          |                                                                    |
| `cookie(key)`                     | case-insensitive; missing returns `""`                             |
| `body()`                          | the outgoing request body                                          |
| `set_body(value)`                 | pre-request scripts only                                           |
| `url()`                           | the outgoing request URL                                           |
| `test(name, condition)`           | passes when the condition is truthy                                |
| `print(value)`                    | debug log; objects/arrays print as real JSON                       |
| `contains(haystack, needle)`      | substring on a string, membership on an array, key test on an object |
| `base64_encode(s)` / `base64_decode(s)` | decode returns `null` on invalid input                       |
| `jwt_decode(token)`               | decodes header/payload only, no signature check; accepts `"Bearer "` prefix |
| `json_parse(s)`                   | invalid JSON returns `null`                                        |
| `json_stringify(v)`               | real quoted JSON, unlike `print`'s readable format                 |
| `aes_encrypt(plaintext, key)` / `aes_decrypt(ciphertext, key)` | AES-256-GCM, key is SHA-256 of the key string, output is `base64(nonce ∥ ciphertext)` |

### The `response` object

| Expression                | Meaning                                        |
| ------------------------- | ---------------------------------------------- |
| `response.status`         | HTTP status code (number)                      |
| `response.text()`         | the raw response body                          |
| `response.json()`         | parsed body (returns `null` on invalid JSON)   |
| `response.contains(x)`    | shorthand for `contains(response.text(), x)`   |

`haystack.contains(needle)` is the same function as `contains(haystack, needle)`
spelled the other way.

## Semantics worth knowing

- Missing `env`/`header`/`cookie` lookups return `""`, so `if x != ""` is the
  natural existence check.
- Reading a missing field returns `null`; `contains` on `null`, a number, or a
  bool is `false` rather than an error, so checking something absent never
  aborts a run.
- Matching in `contains` is case-sensitive (there is no `lower()` to opt back
  in, and an exact match can always be loosened by a shorter needle).
- Numbers compare by value, and equality is strict across types.
- Scripts are interpreted directly from the AST — no bytecode, no JIT. Scripts
  are a few lines, so this is plenty fast.

## Development

```sh
cargo test
```

| File                      | Role                                    |
| ------------------------- | --------------------------------------- |
| `src/lexer.rs`            | tokenizer                               |
| `src/parser.rs`           | recursive-descent parser (tokens → AST) |
| `src/ast.rs`              | AST node definitions                    |
| `src/interpreter.rs`      | tree-walking evaluator, built-ins       |
| `src/builtins.rs`         | JSON/JWT/base64/AES helpers             |
| `src/value.rs`            | runtime values                          |
| `tests/integration.rs`    | end-to-end script tests                 |

The full grammar, built-in reference, and a copy-pasteable LLM prompt that
writes scripts in this language live at
<https://rust-man-org.github.io/rustman/scripting.html>.

## License

MIT — see [LICENSE](LICENSE).
