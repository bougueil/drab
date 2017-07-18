[![drab logo](https://raw.githubusercontent.com/grych/drab-poc/master/web/static/assets/images/drab-banner.png)](https://tg.pl/drab)

[![hex.pm version](https://img.shields.io/hexpm/v/drab.svg)](https://hex.pm/packages/drab)
[![hex.pm downloads](https://img.shields.io/hexpm/dt/drab.svg?style=flat)](https://hex.pm/packages/drab)
[![API Docs](https://img.shields.io/badge/api-docs-yellow.svg?style=flat)](https://hexdocs.pm/drab/)

### See [Demo Page](https://tg.pl/drab) for live demo and description.

Drab - Access the browser User Interface from the Server Side. No Javascript programming needed anymore!

Drab extends Phoenix Framework to "remote control" UI on the browser, live. The idea is to move all User Interface
logic to the server-side, to eliminate Javascript and Ajax calls.

## Teaser

* Client side:

```html
<div class="progress">
  <div class="progress-bar <%= @progress_bar_class %>" role="progressbar" @style.width=<%= "#{@bar_width}%" %>>
    <%= "#{@bar_width}%" %>
  </div>
</div>
<button class="btn btn-primary" drab-click="perform_long_process">
  <%= @long_process_button_text %>
</button>
```

* Server side:

```elixir
def perform_long_process(socket, _sender) do
  poke socket, progress_bar_class: "progress-bar-danger", long_process_button_text: "Processing..."

  steps = :rand.uniform(100)
  for i <- 1..steps do
    Process.sleep(:rand.uniform(500)) #simulate real work
    poke socket, bar_width: Float.round(i * 100 / steps, 2)
  end

  poke socket, progress_bar_class: "progress-bar-success", long_process_button_text: "Click me to restart"
end
```

### Prerequisites

  1. Erlang ~> 19

  2. Elixir ~> 1.4

  3. Phoenix ~> 1.2

## Installation

  So far the process of the installation is rather manually, in the future will be automatized.

  1. Add `drab` to your list of dependencies in `mix.exs` in your Phoenix application and install it:

```elixir
def deps do
  [{:drab, "~> 0.5"}]
end
```

```bash
$ mix deps.get
$ mix compile
```

  2. Initialize Drab client library by adding to the layout page (`web/templates/layout/app.html.eex`)

```html
<%= Drab.Client.js(@conn) %>
```
    
  just after the following line:

```html
<script src="<%= static_path(@conn, "/js/app.js") %>"></script>
```
    
  3. Initialize Drab sockets by adding the following to `web/channels/user_socket.ex`:

```elixir
use Drab.Socket
```

  4. Add Drab template engine to `config.exs`:

```elixir
config :phoenix, :template_engines,
  drab: Drab.Live.Engine
```

  5. Add `:drab` to applications started by default in `mix.exs`:

```elixir
def application do
  [mod: {MyApp, []},
   applications: [:phoenix, :phoenix_pubsub, :phoenix_html, :cowboy, :logger, :gettext, :drab]]
end
```

  6. To enable live reload on Drab pages, add `.drab` extension to live reload patterns in `dev.exs`:

```elixir
config :my_app, MyApp.Endpoint,
  live_reload: [
    patterns: [
      ~r{priv/static/.*(js|css|png|jpeg|jpg|gif|svg)$},
      ~r{priv/gettext/.*(po)$},
      ~r{web/views/.*(ex)$},
      ~r{web/templates/.*(eex|drab)$}
    ]
  ]
```

#### If you want to use Drab.Query (jQuery based module):

  1. Add `jquery` and `boostrap` to `package.json`:

```json
"dependencies": {
  "jquery": ">= 3.1.1",
  "bootstrap": "~3.3.7"
}
```

  2. Add jQuery as a global at the end of `brunch-config.js`:

```javascript
npm: {globals: {
  $: 'jquery',
  jQuery: 'jquery',
  bootstrap: 'bootstrap'
}}
```

  3. And install it:

```bash
$ npm install && node_modules/brunch/bin/brunch build 
```


Congratullations! You have installed Drab and you can proceed with your own commanders.

## Usage

All the Drab functions (callbacks, event handlers) are placed in the module called `Commander`. 
Think about it as a controller for the live pages. Commanders should be placed in `web/commanders` directory.

To enable Drab on the specific pages, you need to add the directive `use Drab.Controller` to your controller. 

Remember the difference: `controller` renders the page, while `commander` works on the live page.

  1. Generate the page Commander. Commander name should correspond to controller, so PageController should have 
  PageCommander:

```bash
$ mix drab.gen.commander Page
* creating web/commanders/page_commander.ex

Add the following line to your Example.PageController:
    use Drab.Controller 
```

  2. As described in the previous task, add `Drab.Controller` to your page Controller (eg. `web/controllers/page_controller.ex` in the default app):

```elixir
defmodule MyApp.PageController do
  use Example.Web, :controller
  use Drab.Controller 

  def index(conn, _params) do
    render conn, "index.html", welcome_text: "Welcome to Phoenix!"
  end
end    
```

  Also add `@welcome_text` assing to `render/3` in index action, to be used in a future.

  3. Rename the template from `web/templates/page/index.html.eex` to `index.html.drab`

  4. Edit the template `web/templates/page/index.html.drab` and change the fixed welcome text to the assign:

```html
<div class="jumbotron">
  <h2><%= @welcome_text %></h2>
```

  3. Edit the commander file `web/commanders/page_commander.ex` and add some real action - the `onload` callback, which fires when the browser connects to the Drab server:

```elixir
defmodule DrabExample.PageCommander do
  use Drab.Commander

  onload :page_loaded

  # Drab Callbacks
  def page_loaded(socket) do
    poke socket, welcome_text: "This page has been drabbed"
    set_prop socket, "div.jumbotron p.lead", 
      innerHTML: "Please visit <a href='https://tg.pl/drab'>Drab</a> page for more"
  end
end
```

The `poke/2` function updates the assign. The `set_prop/3` updates any property of the DOM object. All is done live, without reloading the page.

  4. Run `iex -S mix phoenix.server`. Go to `http://localhost:4000` to see the changed web page. Now you may play with this page live, directly from the `iex`! Observe the instruction given when your browser connects to the page:

```elixir
[debug] 
    Started Drab for same_path:/, handling events in DrabExample.PageCommander
    You may debug Drab functions in IEx by copy/paste the following:
import Drab.{Core, Element, Live}
socket = Drab.get_socket(pid("0.653.0"))

    Examples:
socket |> exec_js("alert('hello from IEx!')")
socket |> poke(count: 42)

```

As instructed, copy and paste those to lines, and check out yourself how could you remote control the displayed page:

```elixir
iex> poke socket, welcome_text: "WOW, this is nice"
%Phoenix.Socket{...}

iex> query socket, "div.jumbotron h2", :innerText
{:ok,
 %{"[drab-id='425d4f73-9c14-4189-992b-41539377c9eb']" => %{"innerText" => "WOW, this is nice"}}}

iex> set_style socket, "div.jumbotron", backgroundColor: "red"
{:ok, 1}
```


### The example above is available [here](https://github.com/grych/drab-example)

## What now?

Visit [Demo Page](https://tg.pl/drab) for a live demo and more description.

Visit [Documentation](https://hexdocs.pm/drab) page.

## Getting help

There is a Drab's thread on [elixirforum.com](https://elixirforum.com/t/drab-phoenix-library-for-server-side-dom-access/3277), please address questions there.

## Tests and Sandbox

Since 0.3.2, Drab is equipped with its own Phoenix Server for automatic integration tests and for sandboxing and play
with it.

### Sandbox

* clone Drab from github:

```shell
git clone git@github.com:grych/drab.git
cd drab
```

* get deps and node modules:

```shell
mix deps.get
npm install && node_modules/brunch/bin/brunch build
```

* start Phoenix with Drab:

```shell
iex -S mix phoenix.server
```

* open the browser and navigate to http://localhost:4000

* follow the instructions in IEx to play with Drab functions:

```elixir
import Drab.{Core, Live, Element, Query, Waiter}
socket = Drab.get_socket(pid("0.784.0"))

iex> query socket, "h3", :innerText
{:ok,
 %{"#header" => %{"innerText" => "Drab Tests"},
   "#page_loaded_indicator" => %{"innerText" => "Page Loaded"}}}

iex> set_prop socket, "h3", innerText: "Updated from IEx"
{:ok, 2}

iex> exec_js socket, "alert('You do like alerts!')"
{:ok, nil}
```

### Tests

Most of the Drab tests are integration (end-to-end) tests, thus they require automated browser. Drab uses [chromedriver](https://sites.google.com/a/chromium.org/chromedriver/), which must be running while you run tests.

* clone Drab from github:

```shell
git clone git@github.com:grych/drab.git
cd drab
```

* get deps and node modules:

```shell
mix deps.get
npm install && node_modules/brunch/bin/brunch build
```

* run `chromedriver` 

* run tests:

```shell
% mix test                 
Compiling 23 files (.ex)
........................................

Finished in 120.9 seconds
123 tests, 0 failures

Randomized with seed 934572
```

## Contact

(c)2016-2017 Tomek "Grych" Gryszkiewicz, 
<grych@tg.pl>


Illustrations by https://www.vecteezy.com


