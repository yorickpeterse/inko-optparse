# inko-optparse

An opinionated and lightweight command-line argument and option parser for
[Inko](https://inko-lang.org), inspired by getopts, with full support for
Unicode options.

Both short and long options are supported, using the syntax `-o` and `--option`
for short and long options respectively. Long option pairs (`--option=value`)
are also supported. Short option pairs (`-o=v`) are parsed such that the value
includes the `=`, so `-o=v` is parsed the same way as `-o "=v"`. There are no
plans to support parsing these such that the value doesn't include the `=` (i.e.
parsing `-o=v` as `-o v`).

Short option names are restricted to single extended grapheme clusters
("characters"), meaning `-foo` isn't a valid option. Long option names must
contain at least two characters, so `--h` isn't a valid option.

inko-optparse supports parsing `--` as an argument. Upon encountering this
argument, all arguments that followed are kept as-is. This allows passing these
arguments to subcommands or other processes.

# Requirements

- Inko 0.13.1 or newer

# Installation

```bash
inko pkg add github.com/yorickpeterse/inko-optparse 0.2.0
inko pkg sync
```

# Usage

The type `optparse.Options` is used to define options, using the methods
`Options.flag` for flags (e.g. `-h`/`--help`), `Options.single` for options that
require a single value (e.g. `--config=foo.json`), and `Options.multiple` for
options that accept multiple values (e.g. `--include=foo --include=bar`).

Once your options are defined, you can parse a list of arguments (e.g. the
return value of `std.env.arguments`) into a `Matches` type using
`Options.parse`:

```inko
import optparse.Options
import std.env

class async Main {
  fn async main {
    let opts = Options.new

    opts.flag('h', 'help', 'Show this help message')

    let matches = opts.parse(env.arguments).unwrap
  }
}
```

Using `Matches.contains?` you can check if an option is specified, using either
the short or long name. `Matches.value` returns the first argument of an option
(provided it accepts arguments), while `Matches.values` returns all the
arguments specified. Any remaining arguments are stored in `Matches.remaining`.

By default, `Options` parses any options it encounters and will throw an error
for any invalid options. The option `Options.stop_at_first_non_option` can be
used to customize this behaviour: when set to `true`, the first value
encountered that isn't an option argument results in it and any arguments that
follow it being stored in `Matches.remaining`:

```inko
import optparse.Options
import std.env
import std.fmt.(fmt)
import std.stdio.STDOUT

class async Main {
  fn async main {
    let opts = Options.new

    opts.flag('h', 'help', 'Show this help message')
    opts.stop_at_first_non_option = true

    let matches = opts.parse(env.arguments).unwrap

    STDOUT.new.print(fmt(matches.remaining))
  }
}
```

When run with the command-line arguments `['foo', '--bar']`, instead of
producing an error (or panic in the case of this example, due to the `unwrap`),
the output is `['foo', '--bar']`. This is useful if you want to support
subcommands with their own options, using the syntax
`root-command --root-option [...] sub-command --sub-option [...]`.

Here's a more in-depth example of using inko-optparse, including displaying a
usage message and the available options:

```inko
import optparse.(Help, Options)
import std.env
import std.fmt.(fmt)
import std.stdio.(STDOUT, STDERR)
import std.sys

class async Main {
  fn async main {
    let opts = Options.new

    opts.flag('h', 'help', 'Show this help message')
    opts.single('c', 'config', 'PATH', 'Use a custom configuration file')
    opts.multiple('i', 'include', 'DIR', 'Add the directory')
    opts.multiple('I', 'ignore', '', 'Ignore something')
    opts.flag('', 'verbose', "Use verbose output,\nlorem ipsum")
    opts.flag('', 'example', '')
    opts.flag('x', '', 'Foo')

    let matches = match opts.parse(env.arguments) {
      case Ok(v) -> v
      case Error(e) -> {
        STDERR.new.print("test: {e}")
        sys.exit(1)
      }
    }

    if matches.contains?('help') {
      let help = Help
        .new('test')
        .description(
          'This is an example program to showcase the optparse library.'
        )
        .section('Examples')
        .line('test --help')
        .section('Options')
        .options(opts)
        .to_string

      STDOUT.new.write_string(help)
    } else {
      STDOUT.new.print(fmt(matches.remaining))
    }
  }
}
```

When running this program and providing the `--help` option, the output is:

```
Usage: test [OPTIONS]

This is an example program to showcase the optparse library.

Examples:

  test --help

Options:

  -h, --help           Show this help message
  -c, --config=PATH    Use a custom configuration file
  -i, --include=DIR    Add the directory
  -I, --ignore         Ignore something
      --verbose        Use verbose output,
                       lorem ipsum
      --example
  -x                   Foo
```

# License

All source code in this repository is licensed under the Mozilla Public License
version 2.0, unless stated otherwise. A copy of this license can be found in the
file "LICENSE".
