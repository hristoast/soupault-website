<h1 id="post-title">Soupault 2.2.0 release</h1>

<p>Date: <time id="post-date">2020-11-29</time> </p>

<p id="post-excerpt">
Soupault 2.2.0 is <a href="https://files.baturin.org/software/soupault/2.2.0">available for download</a>.
It brings support for settings shared between plugins, makes it easier to work with tables from Lua code,
adds some date/time manipulation functions, and fixes a few bugs. This release also removes support
for 32-bit Windows.
</p>

## Bug fixes

First, I fixed a TOML syntax error in default configs generated by `soupault --init`. The issue was a stray semicolon after `keep_doctype = true`.
That issue only affected soupault 2.1.0.

Second, integer-indexed Lua tables are now correctly handled as "lists" when projected back to OCaml code. This makes it possible to render list items in order when calling
the template processor from Lua plugins. For example, this code works correctly now:

```lua
env = {}
env["vars"] = {"foo", "bar", "baz", "quux"}

tmpl = [[
Metasyntactic variables come in this order: {{join(", ", vars)}}
]]

result = String.render_template(tmpl, env)

Log.debug(result)
```

Now it will produce `Metasyntactic variables come in this order: foo, bar, baz, quux` as expected. 
In earlier versions it might have printed them out of order because Lua hashes aren't ordered internally
and their order is "in the eye of the beholder". Now soupault checks if every key of a hash
is an integer number to see if a table was _supposed_ to be a list.

Third, syntax errors in template strings cause a proper error message rather than a raw exception now.

## New features

### Custom options table

Some people asked if there's a way to share settings betweeen multiple plugins. Until this version, there was no way to do that.
You couldn't put custom options in built-in tables like `[settings]`, because they have a pre-determined list
of allowed options and invalid options cause errors.

I believe it's better than allowing invalid options to pass silently. There's nothing more frustrating than
making a typo in an option and wondering why it doesn't work. However, support for custom options is a valid request.

There's now a new `[custom_options]` table where you can out arbitrary options. It's exempt from the option validity checks,
and those options have no meaning to soupault itself. Plugins are free to interpret them however they want.

```toml
[custom_options]
  site_url = "https://example.com"
```

### Global config accessible to plugins

There was also no way for plugins to access the global config. The `config` variable in the plugin environment
only contains the _widget_ options. 

Now there's also a new `soupault_config` variable that contains the complete config—deserialized contents of `soupault.conf`.

Yes, exact words. Not "the effective config", but "deserialized contents of `soupault.conf`".
Default values are not automatically injected into the `soupault_config` yet.

Thus, you need to be careful with defaults for now. For example, if `site_dir` is not set explicitly in the config, then 
`soupault_config["settings"]["site_dir"]` will be `nil`, not `site/`, and you will need to substitute a default yourself.

I can see how this issue can become annoying, and I'm going to redesign this part in future releases.

### Time-aware dates

Older soupault versions allowed datetime formats with time variables like `%Y-%d-%m %H:%M`, but 
ignored the time part in practice. 

This issue is fixed now, and soupault will parse times and use them for sorting.

### New plugin functions

#### Datetime functions

Working with dates is awkward. It's also necessary.

One of the goals of soupault is to allow creating workflows resistant to software rot.
That's why I make statically linked executables and include things that seem out of scope,
like a "logicful" template processor. [Jingoo](https://github.com/tategakibunko/jingoo) isn't there
because it makes anything new possible that you cannot do with external scripts.
It's there so that you don't _have to_ use an external script just to add some template rendering
to your workflow.

Same with dates. Originally I used a Python script for generating the Atom feed for this blog.
Then I set out to write an Atom feed generator in Lua. The Atom standard demands that dates
use the ISO format (`2020-09-20T00:00:00+00:00`), while most people (myself included) don't write dates in that format.
Some date parsing and formatting functionality was necessary.

So, now there are some [date parsing and formatting functions](/reference-manual/#Date).

The Atom feed is now generated by a [Lua plugin](https://github.com/dmbaturin/soupault.neocities.org/blob/master/plugins/atom.lua) now.
It's still somewhat experimental and it may be too early to steal it for your own site, but the concept is working.

#### Unsafe JSON parsing

The original `JSON.from_string()` stops plugin execution if it encounters invalid JSON.
This isn't always desirable. There's now `JSON.unsafe_from_string()` that returns `nil` if it cannot parse its input.

Note that `"null"` is a valid JSON string that also parses to `nil`, so getting a `nil` from it doesn't always
mean there was a parse error.

#### Table helpers

Iterating through tables is a real weak point of the 2.5 era Lua implemented by Lua-ML.
To make working with tables simpler, I've added a high level few helpers functions, including:

* `Table.iter(func, table`) 
* `Table.iter_values(func, table)`
* `Table.apply(func, table)`
* `Table.fold(func, table)
* `Table.fold_values(func, table)`

Read the [reference manual](/reference-manual/#Table) for details.

Note that `Table.iter` functions don't take the order of keys into account, so if you are using an integer-indexed
table as an "array" and want to traverse it in order, you still need a loop with a counter
(and hope that there are no gaps between keys—Lua has no safeguards against that!).

I may try to add "order-aware" iteration functions, but this needs a serious design discussion.

An example of using `Table.iter`:

```lua
my_table = {}
my_table["foo"] = 0
my_table["bar"] = "quux"
my_table["baz"] = {1,2}

function show_pair(k, v)
  Log.debug(format("Key: %s, value: %s", k, JSON.to_string(v)))
end

Table.iter(show_pair, my_table)
```

This is what it will show in the build log:

```
[DEBUG] Key: foo, value: 0
[DEBUG] Key: bar, value: quux
[DEBUG] Key: baz, value: [1,2]
```

#### Datetime functions

### Other improvements

Lua-ML supports the modulo operator now (`5 % 2 = 1`).
I have no idea why original authors left it out, but it also took me a year to discover that it's missing...
in any case, it's not an issue anymore.

The `escape` filter of the template processor now emits symbolic HTML entities like `&gt;` now,
rather than numeric character codes.

## Win32 support is no more

32-bit Windows is deprecated and all new Windows versions are 64-bit.
It's getting harder to maintain 32-bit build setups as well,
so starting from this release I'll only be making Win64 builds.

If you really want Win32 support back, let me know. If enough people want it,
I suppose I can come up with something, e.g. keep supporting a 32-bit
_cross-compilation_ setup.

## Thanks to...

[Masaki WATANABE](https://github.com/tategakibunko), for adding symbolic HTML entity support to Jingoo.

[Hugo Heuzard](https://github.com/hhugo) for cooperation to resurrect [odate](https://github.com/hhugo/odate).

[Hristos N. Triantafillou](hristos.lol/) for testing experimental features.