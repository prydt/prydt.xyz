+++
title = "Configuration Languages Should be Strongly Typed"
date = 2026-03-05
+++

Nothing in an undergraduate education in computer science prepared me for the amount of YAML I would have to stare at as a software engineer. Whether its to configure k8s, docker, CI, my eyes start to glaze over. The indentation guides on my favorite hipster IDE blend together as I try to figure out whether the snippet of config I copy-and-pasted from another codebase is correctly aligned. 

This sucks.

Much of the infastructure that powers our world is build on a foundation of oceans of JSON and mountains of YAML. 

I think the configuration languages we use should be more like programming languages. In specific, they should have a strong type system with the ability to effectively convey constraints, and require schemas.

Here's an example. 
```yaml
server:
  port: "8080"         # string or int?
  timeout: 30          # seconds? milliseconds?
  ssl: "yes"           # bool? string? "yes"/"true"/1?
  max_connections: -1  # valid? sentinel for "unlimited"?
  log_level: "verbose" # what are the valid options?
  retries: 3.7         # fractional retries make no sense
```

Now while this is just a random bit of YAML I produced, we can already start to see the limitations of YAMLs types and how it is conveyed to the programmer.

Let's see how we can do better using something like Zod, a TypeScript schema validation library.
```typescript
const ServerConfig = z.object({
  port: z.number().int().min(1).max(65535),
  timeout: z.number().positive().describe("Timeout in milliseconds"),
  ssl: z.boolean(),
  maxConnections: z.number().int().positive().or(z.literal(Infinity)),
  logLevel: z.enum(["debug", "info", "warn", "error"]),
  retries: z.number().int().nonnegative(),
});
```

This is an example of the sorts of constraints which would be really great to be able to express in a configuration language. Here, we can tell that:
- port must be within the range [1, 65535]
- logLevel must be one of the options instead of an arbitrary string

and so on.

Strong types let us convey constraints clearly and regardless of whether we convey it to the user, all of our programs have constraints. This is why a schema is also important. Regardless of whether we provide a schema to the user, there is always an *implicit schema*.

This schema-on-read is built in to the way that our program parses (and validates... we are validating user-provided data, right?) the config file. 

Now take a look at this config file:
```dhall
let LogLevel = < Debug | Info | Warn | Error >

let MaxConnections = < Unlimited | Limit : Natural >

let ServerConfig =
  { port : Natural
  , timeout_ms : Natural
  , ssl : Bool
  , log_level : LogLevel
  , max_connections : MaxConnections
  , retries : Natural
  }

let config : ServerConfig =
  { port = 8080
  , timeout_ms = 5000
  , ssl = True
  , log_level = LogLevel.Info
  , max_connections = MaxConnections.Limit 100
  , retries = 3
  }

in { server = config }
```
This is a configuration language called [Dhall](https://dhall-lang.org/). You can think of it like "JSON + functions + types + imports." This Dhall file actually compiles into
```yaml
server:
  log_level: Info
  max_connections: 100
  port: 8080
  retries: 3
  ssl: true
  timeout_ms: 5000
```

but it is so much clearer.


To conclude, types and schemas are really our best friends for dealing with all sorts of problems we can run into when writing configuration state in brittle formats such as JSON or YAML. 

There are many other interesting higher level configuration languages to try like Dhall, Apple's [Pkl](https://pkl-lang.org/index.html), and Google's [Starlark](https://starlark-lang.org/). If jumping in the deep end is impractical, even just combining an existing JSON config file with a [JSON Schema](https://json-schema.org/learn/getting-started-step-by-step) is a great first step, and provides real benefits.

-- prydt

## Further reading
- [https://beza1e1.tuxen.de/config_levels.html](https://beza1e1.tuxen.de/config_levels.html)
- [https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/)
