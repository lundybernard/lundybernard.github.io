---
layout: post
title:  "Configuration is a User Interface"
date:   2026-01-23 12:00:00 -0700
---

Configuration management is part of your User Interface.

Whether you are creating an application for users, a library for devs, 
or a microservice, how your software handles configuration 
is a critical concern and needs to cater to your users' needs.


## What is Configuration?
Configuration in this context is specifically user-controllable settings which
are used to change the behavior of the software at runtime. 
This includes things like logging levels, feature flags, 
user preferences like dark/light mode. 
It excludes things like plugins, user-defined business logic/rules, and data
which the software will process.
`llm_model="gpt-oss"` is a config setting, the model file itself is not.


## How will your users set configuration options?
That's the critical question because the answer depends on who your users are
and their needs.
In a GUI or web-app it is common practice to include a 'settings' menu.
Config files are a common solution for many applications.
There are many other ways to configure software like:
  * command line arguments
  * environment variables
  * configuration store systems (Consul, etcd, Kubernetes ConfigMaps, etc.)

Often, but not always, software libraries receive their config settings from
the code which utilizes them.

It's good to keep GUI settings in mind when thinking about config options,
if it is easy to include on a settings page, its probably a good config option.


## What is not configuration?
There is no hard and fast rule, which is why it is helpful to think of config
as part of the UI.
What starts as a simple configuration can quickly grow into large complex
collections of settings. 
Things like workflows, Rulesets, and Schemas, kubernetes manifests, 
and logstash pipelines are beyond the scope of standard configuration.
Toggles for debug logging and darkmode are a separate concern 
from data-processing rules which require their own DSL.


## Practical Advice:
### Think of config as part of your UI
When someone edits your config file, they’re not “tinkering with internals.” 
They’re using an interface you designed—whether you meant to or not.
So configuration deserves the same care as any other UI element,
it should:
* be easy to understand.
* be hard to misuse.
* fail with helpful error messages.
* be stable across versions.
The best config UX assumes the user:
* is tired
* is in a hurry
* is operating the tool from a different context than the author
* will copy/paste examples
* will try to override settings in env vars

### Prefer “strings at the edges” (parse inside the program)
Configuration values enter the system as strings.
Your application turns them into the typed values that it needs.
Don’t make your users debug type mechanics. 
Own handle parsing and validation in code.

This helps you craft helpful error messages like:
“TIMEOUT must look like 30s, 5m, or 250ms”
“THRESHOLD must be between 0 and 1”

This supports configuration using Environment variables, and is important for
users running software in containers, on cloud infrastructure, 
and other contexts ENV variables are preferred.

### Be wary of Lists
Lists and key:value maps require special consideration.
It often makes sense to include lists of options in a configuration.
A short list in a CLI arg, or .yaml file can be simple and useful...
But lists are difficult to represent as strings, in Environment variables,
and require extra care when converting them to types.
Long lists (more than you want to type out by hand) probably represent Data
not configuration.

### Interpolation is business logic (don’t hide logic in config)
Interpolation and templating in config starts innocent:
“Let me reuse a base URL”
“Let me reference DATA_DIR”
“Let me compute a path”
Then it grows to include conditionals, defaults, string functions, 
environment lookups, precedence rules, edge cases around escaping...
It becomes its own language (DSL)

Configuration should declare inputs.
If values need to be derived, do the derivation in code where it can be tested.

### For complex user logic, use a separate system
If you require advanced user-defined logic, keep it separate from your runtime
config.  Build the appropriate testing and documentation around it, and avoid
conflating simple settings like darkmode and quiet output with programatic logic.

keep the standard config small and boring (good!)
move complex, user-defined behavior into a separate lane:
* plugins
* a dedicated rule file format with tooling
* a database-backed rules UI
* a sandboxed expression system with tests

This separation protects your config UI 
from becoming a fragile all-purpose control panel.

### Don’t confuse configuration with data
This is one of the most expensive mistakes in “research tools that became real systems.”
Configuration answers: How should the tool run?
Data answers: What should the tool process?
If your config file starts containing lots of records, big lists, 
or evolving rule sets, it’s probably not config anymore—it’s data.

Config is not a substitute database.


## Conclusion
Configuration is part of your user interface,
treat it like you would any other UI: 
keep it small, understandable, and hard to misuse.

In practice that means designing for the real world: 
multiple config sources (file, CLI, env vars),
users copying examples, and “future you” debugging a run at 2am. 
Favor simple, string-shaped inputs at the boundaries, 
validate and parse inside the program, and resist the temptation to smuggle
computation into configuration through interpolation.

And when your users truly need complex logic 
don’t force it into the same settings system. 
Give it a separate lane with the right tooling, validation, and tests.

Boring configuration is a feature. 
It makes your tool easier to run, easier to deploy, and easier to trust. 
