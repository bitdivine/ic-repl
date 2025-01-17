# Canister REPL

```
ic-repl [--replica [local|ic|url] | --offline [--format [ascii|png]]] --config <dhall config> [script file]
```

## Commands

```
<command> := 
 | import <id> = <text> (as <text>)?         // bind canister URI to <id>, with optional did file
 | export <text>                             // export current environment variables
 | load <text>                               // load and run a script file
 | config <text>                             // set config for random value generator in dhall format
 | let <id> = <exp>                          // bind <exp> to a variable <id>
 | <exp>                                     // show the value of <exp>
 | assert <exp> <binop> <exp>                // assertion
 | identity <id> (<text> | record { slot_index = <nat>; key_id = <text> })?   // switch to identity <id>, with optional pem file or HSM config
 | function <id> ( <id>,* ) { <command>;* }  // define a function
<exp> := 
 | <candid val>                                    // any candid value
 | <var> <transformer>*                            // variable with optional transformers
 | fail <exp>                                      // convert error message as text
 | call (as <name>)? <name> . <name> ( <exp>,* )   // call a canister method, and store the result as a single value
 | encode (<name> . <name>)? ( <exp>,* )           // encode candid arguments as a blob value. canister.__init_args represents init args
 | decode (as <name> . <name>)? <exp>              // decode blob as candid values
 | <id> ( <exp>,* )                                // function application
<var> := 
 | <id>                  // variable name 
 | _                     // previous eval of exp is bind to `_`
<transformer> :=
 | ?                     // select opt value
 | . <name>              // select field name from record or variant value
 | [ <nat> ]             // select index from vec, record, or variant value
 | . <id> ( <exp>,* )    // transform (map, filter, fold) a collection value
<binop> := 
 | ==                    // structural equality
 | ~=                    // equal under candid subtyping; for text value, we check if the right side is contained in the left side
 | !=                    // not equal
```

## Functions

Similar to most shell languages, functions in ic-repl is dynamically scoped and untyped.
You cannot define recursive functions, as there is no control flow in the language.

We also provide some built-in functions:
* `account(principal)`: convert principal to account id.
* `neuron_account(principal, nonce)`: convert (principal, nonce) to account in the governance canister.
* `metadata(principal, path)`: fetch the HTTP endpoint of `canister/<principal>/<path>`.
* `file(path)`: load external file as a blob value.
* `gzip(blob)`: gzip a blob value.
* `stringify(exp1, exp2, exp3, ...)`: convert all expressions to string and concat. Only supports primitive types.
* `output(path, content)`: append text content to file path.
* `wasm_profiling(path)/wasm_profiling(path, vec { func_name })`: load Wasm module, instrument the code and store as a blob value. Calling profiled canister binds the cost to variable `__cost_{id}` or `__cost__`. The second argument of func_names is optional. If provided, it will only count and trace the provided functions.
* `flamegraph(canister_id, title, filename)`: generate flamegraph for the last update call to canister_id, with title and write to `{filename}.svg`. The cost of the update call is returned.
* `concat(e1, e2)`: concatenate two vec/record/text together.
* `add/sub/mul/div(e1, e2)`: addition/subtraction/multiplication/division of two integer/float numbers. If one of the arguments is float32/float64, the result is float64; otherwise, the result is integer. You can use type annotation to get the integer part of the float number. For example `div((mul(div(1, 3.0), 1000) : nat), 100.0)` returns `3.33`.

## Object methods

For `vec`, `record` or `text` value, we provide some built-in methods for value transformation:
* v.map(func): transform each item `v[i]` with `func(v[i])`.
* v.filter(func): filter out item `v[i]` if `func(v[i])` returns `false` or has an error.
* v.fold(init, func): combine all items in `v` by repeatedly applying `func(...func(func(init, v[0]), v[1])..., v[n-1])`.
* v.size(): count the size of `v`.

For `record` value, `v[i]` is represented as `record { key; value }` sorted by field id.

For `text` value, `v[i]` is represented as a `text` value containing a single character.

## Type casting

Type annotations in `ic-repl` is more permissible (not following the subtyping rules) than the Candid library to allow piping results from different canister calls.
* `("text" : blob)` becomes `blob "text"` and vice versa. Converting `blob` to `text` can get an error if the blob is not utf8 compatible.
* `(service "aaaaa-aa" : principal)` becomes `principal "aaaaa-aa"`. You can convert among `service`, `principal` and `func`.
* `((((1.99 : nat8) : int) : float32) : nat32)` becomes `(1 : nat32)`. When converting from float to integer, we only return the integer part of the float.
* Type annotations for `record`, `variant` is left unimplemented. With candid interface embedded in the canister metadata, annotating composite types is almost never needed.

## Examples

### test.sh
```
#!/usr/bin/ic-repl -r ic
// assume we already installed the greet canister
import greet = "rrkah-fqaaa-aaaaa-aaaaq-cai";
call greet.greet("test");
let result = _;
assert _ == "Hello, test!";
identity alice;
call "rrkah-fqaaa-aaaaa-aaaaq-cai".greet("test");
assert _ == result;
```

### nns.sh
```
#!/usr/bin/ic-repl -r ic
// nns and ledger canisters are auto-imported if connected to the mainnet
call nns.get_pending_proposals()
identity private "./private.pem";
call ledger.account_balance(record { account = account(private) });

function transfer(to, amount, memo) {
  call ledger.transfer(
    record {
      to = to;
      fee = record { e8s = 10_000 };
      memo = memo;
      from_subaccount = null;
      created_at_time = null;
      amount = record { e8s = amount };
    },
  );
};
function stake(amount, memo) {
  let _ = transfer(neuron_account(private, memo), amount, memo);
  call nns.claim_or_refresh_neuron_from_account(
    record { controller = opt private; memo = memo }
  );
  _.result?.NeuronId
};
let neuron_id = stake(100_000_000, 42);
```

### install.sh
```
#!/usr/bin/ic-repl
function deploy(wasm) {
  let id = call ic.provisional_create_canister_with_cycles(record { settings = null; amount = null });
  call ic.install_code(
    record {
      arg = encode ();
      wasm_module = wasm;
      mode = variant { install };
      canister_id = id.canister_id;
    },
  );
  id
};

identity alice;
let id = deploy(file("greet.wasm"));
let status = call ic.canister_status(id);
assert status.settings ~= record { controllers = vec { alice } };
assert status.module_hash? == blob "...";
let canister = id.canister_id;
call canister.greet("test");
```

### wallet.sh
```
#!/usr/bin/ic-repl
import wallet = "${WALLET_ID:-rwlgt-iiaaa-aaaaa-aaaaa-cai}" as "wallet.did";
identity default "~/.config/dfx/identity/default/identity.pem";
call wallet.wallet_create_canister(
  record {
    cycles = ${CYCLE:-1_000_000};
    settings = record {
      controllers = null;
      freezing_threshold = null;
      memory_allocation = null;
      compute_allocation = null;
    };
  },
);
let id = _.Ok.canister_id;
call as wallet ic.install_code(
  record {
    arg = encode ();
    wasm_module = file("${WASM_FILE}");
    mode = variant { install };
    canister_id = id;
  },
);
call id.greet("test");
```

### profiling.sh
```
#!/usr/bin/ic-repl
import "install.sh";

let file = "result.md";
output(file, "# profiling result\n\n");
output(file, "|generate|get|put|\n|--:|--:|--:|\n");

let cid = deploy(gzip(wasm_profiling("hashmap.wasm")));
call cid.__toggle_tracing();   // Disable flamegraph tracing
call cid.generate(50000);
output(file, stringify(__cost__, "|"));

call cid.__toggle_tracing();   // Enable flamegraph tracing
call cid.batch_get(50);
flamegraph(cid, "hashmap.get(50)", "get");
output(file, stringify("[", __cost__, "](get.svg)|"));

let put = call cid.batch_put(50);
flamegraph(cid, "hashmap.put(50)", "put.svg");
output(file, stringify("[", __cost_put, "](put.svg)|\n"));
```

## Relative paths

Several commands and functions are taking arguments from the file system. We have different definitions for
relative paths, depending on whether you are reading or writing the file.

* For reading files, e.g., `import`, `load`, `identity`, `file`, `wasm_profiling`, relative paths are based on where the current script is located;
* For writing files, e.g., `export`, `output`, `flamegraph`, relative paths are based on the current directory when the script is run.

The rationale for the difference is that we can have an easier time to control where the output files are located, as scripts can spread out in different directories.

## Derived forms

* `call as proxy_canister target_canister.method(args)` is a shorthand for
```
let _ = call proxy_canister.wallet_call(
  record {
    args = encode target_canister.method(args);
    cycles = 0;
    method_name = "method";
    canister = principal "target_canister";
  }
);
decode as target_canister.method _.Ok.return
```

## Contributing

Please follow the guidelines in the [CONTRIBUTING.md](.github/CONTRIBUTING.md) document.

## Issues

* Acess to service init type (get from either Wasm or http endpoint)
* `IDLValue::Blob` for efficient blob serialization
* Autocompletion within Candid value
* Robust support for `~=`, requires inferring principal types
* Loop detection for `load`
* Assert upgrade correctness
