# Meeseeks

[![Build Status](https://travis-ci.org/mischov/meeseeks.svg?branch=master)](https://travis-ci.org/mischov/meeseeks)
[![Meeseeks version](https://img.shields.io/hexpm/v/meeseeks.svg)](https://hex.pm/packages/meeseeks)

Meeseeks is an Elixir library for parsing and extracting data from HTML and XML with CSS or XPath selectors.

```elixir
import Meeseeks.CSS

html = HTTPoison.get!("https://news.ycombinator.com/").body

for story <- Meeseeks.all(html, css("tr.athing")) do
  title = Meeseeks.one(story, css(".title a"))
  %{title: Meeseeks.text(title),
    url: Meeseeks.attr(title, "href")}
end
#=> [%{title: "...", url: "..."}, %{title: "...", url: "..."}, ...]
```
See [HexDocs](https://hexdocs.pm/meeseeks/Meeseeks.html) for additional documentation.

## Dependencies

Meeseeks depends on [html5ever](https://github.com/servo/html5ever) via [meeseeks_html5ever](https://github.com/mischov/meeseeks_html5ever).

Because html5ever is a Rust library, you will need to have the Rust compiler [installed](https://www.rust-lang.org/en-US/install.html).

This dependency is necessary because there are no HTML5 spec compliant parsers written in Elixir/Erlang.

## Installation

Ensure Rust is installed, then add Meeseeks to your `mix.exs`:

```elixir
defp deps do
  [
    {:meeseeks, "~> 0.9.0"}
  ]
end
```

Finally, run `mix deps.get`.

## Getting Started

### Parse

Start by parsing a source (HTML/XML string or [`Meeseeks.TupleTree`](https://hexdocs.pm/meeseeks/Meeseeks.TupleTree.html)) into a [`Meeseeks.Document`](https://hexdocs.pm/meeseeks/Meeseeks.Document.html) so that it can be queried.

`Meeseeks.parse/1` parses the source as HTML, but `Meeseeks.parse/2` accepts a second argument of either `:html` or `:xml` that specifies how the source is parsed.

```elixir
document = Meeseeks.parse("<div id=main><p>1</p><p>2</p><p>3</p></div>")
#=> #Meeseeks.Document<{...}>
```

The selection functions accept an unparsed source, parsing it as HTML, but parsing is expensive so parse ahead of time when running multiple selections on the same document.

### Select

Next, use one of Meeseeks's selection functions - `fetch_all`, `all`, `fetch_one`, or `one` - to search for nodes.

All these functions accept a queryable (a source, a document, or a [`Meeseeks.Result`](https://hexdocs.pm/meeseeks/Meeseeks.Result.html)), one or more [`Meeseeks.Selector`](https://hexdocs.pm/meeseeks/Meeseeks.Selector.html)s, and optionally an initial context.

`all` returns a (possibly empty) list of results representing every node matching one of the provided selectors, while `one` returns a result representing the first node to match a selector (depth-first) or nil if there is no match.

`fetch_all` and `fetch_one` work like `all` and `one` respectively, but wrap the result in `{:ok, ...}` if there is a match or return `{:error, %Meeseeks.Error{type: :select, reason: :no_match}}` if there is not.

To generate selectors, use the `css` macro provided by [`Meeseeks.CSS`](https://hexdocs.pm/meeseeks/Meeseeks.CSS.html) or the `xpath` macro provided by [`Meeseeks.XPath`](https://hexdocs.pm/meeseeks/Meeseeks.XPath.html).

```elixir
import Meeseeks.CSS
result = Meeseeks.one(document, css("#main p"))
#=> #Meeseeks.Result<{ <p>1</p> }>

import Meeseeks.XPath
result = Meeseeks.one(document, xpath("//*[@id='main']//p"))
#=> #Meeseeks.Result<{ <p>1</p> }>
```

### Extract

Retrieve information from the [`Meeseeks.Result`](https://hexdocs.pm/meeseeks/Meeseeks.Result.html) with an extraction function.

The extraction functions are `attr`, `attrs`, `data`, `dataset`, `html`, `own_text`, `tag`, `text`, `tree`.

```elixir
Meeseeks.tag(result)
#=> "p"
Meeseeks.text(result)
#=> "1"
Meeseeks.tree(result)
#=> {"p", [], ["1"]}
```

The extraction functions `html` and `tree` work on [`Meeseeks.Document`](https://hexdocs.pm/meeseeks/Meeseeks.Document.html)s in addition to [`Meeseeks.Result`](https://hexdocs.pm/meeseeks/Meeseeks.Result.html)s.

```elixir
Meeseeks.html(document)
#=> "<html><head></head><body><div id=\"main\"><p>1</p><p>2</p><p>3</p></div></body></html>"
```

## Custom Selectors

Meeseeks is designed to have extremely extensible selectors, and creating a custom selector is as easy as defining a struct that implements the [`Meeseeks.Selector`](https://hexdocs.pm/meeseeks/Meeseeks.Selector.html) behaviour.

```elixir
defmodule CommentContainsSelector do
  use Meeseeks.Selector

  alias Meeseeks.Document

  defstruct value: ""

  def match(selector, %Document.Comment{} = node, _document, _context) do
    String.contains?(node.content, selector.value)
  end

  def match(_selector, _node, _document, _context) do
    false
  end
end

selector = %CommentContainsSelector{value: "TODO"}

Meeseeks.one("<!-- TODO: Close vuln! -->", selector)
#=> #Meeseeks.Result<{ <!-- TODO: Close vuln! --> }>
```

To learn more, check the documentation for [`Meeseeks.Selector`](https://hexdocs.pm/meeseeks/Meeseeks.Selector.html) and [`Meeseeks.Selector.Combinator`](https://hexdocs.pm/meeseeks/Meeseeks.Selector.Combinator.html)

## Contribute

Contributions are very welcome, especially bug reports.

If submitting a bug report, please search open and closed issues first.

To make a pull request, fork the project, create a topic branch off of `master`, push your topic branch to your fork, and open a pull request.

If you're submitting a bug fix, please include a test or tests that would have caught the problem.

If you're submitting new features, please test and document as appropriate.

Before submitting a PR, please run `mix format`.

By submitting a patch, you agree to license your work under the license of this project.

### Running Tests

```
$ git clone https://github.com/mischov/meeseeks.git
$ cd meeseeks
$ mix deps.get
$ mix test
```

### Building Docs

```
$ MIX_ENV=docs mix docs
```

## License

Meeseeks is licensed under the [MIT License](LICENSE)
