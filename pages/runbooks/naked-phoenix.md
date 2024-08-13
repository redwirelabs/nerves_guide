---
description: >-
  Run a Phoenix web-based UI directly within a Nerves firmware project. No
  umbrella, no poncho.
---

# Naked Phoenix

The Nerves project will be the main project, and the basis for the firmware. Phoenix boilerplate will be pulled into the Nerves project. This guide uses Elixir 1.17.2-otp-27, Erlang 27.0.1, Phoenix 1.17.14, and Nerves 1.11.1. You may need to adapt the source code examples on other versions.

Start by running the Nerves project generator first.

```
mix nerves.new --target=bbb my_firmware
```

Create the Phoenix project out of tree from the firmware, with the same name as the firmare project. This project will be thrown away at the end. The `--module` param is intentionally omitted here, as using it doesn’t work as well.

```
mix phx.new --no-ecto --no-mailer my_firmware
```

Copy the Phoenix web files to the Nerves project. The context layer won’t be copied.

```
cp <phoenix>/lib/my_firmware_web.ex <nerves>/lib/
cp -r <phoenix>/lib/my_firmware_web <nerves>/lib/
cp -r <phoenix>/assets <nerves>/
cp -r <phoenix>/priv <nerves>/
```

Copy the Phoenix project dependencies into the Nerves project’s `mix.exs` file. The host dependencies will need the host target specified so that they only run on the development computer, not on the firmware.

{% code title="mix.exs" %}
```elixir
defp deps do
  [
    # Dependencies for all targets
    {:nerves, "~> 1.10", runtime: false},
    {:shoehorn, "~> 0.9.1"},
    {:ring_logger, "~> 0.11.0"},
    {:toolshed, "~> 0.4.0"},

    # Allow Nerves.Runtime on host to support development, testing and CI.
    # See config/host.exs for usage.
    {:nerves_runtime, "~> 0.13.0"},

    # Dependencies for all targets except :host
    {:nerves_pack, "~> 0.7.1", targets: @all_targets},

    # Dependencies for specific targets
    # NOTE: It's generally low risk and recommended to follow minor version
    # bumps to Nerves systems. Since these include Linux kernel and Erlang
    # version updates, please review their release notes in case
    # changes to your application are needed.
    {:nerves_system_bbb, "~> 2.19", runtime: false, targets: :bbb},

    # Phoenix dependencies
    {:phoenix, "~> 1.7.14"},
    {:phoenix_html, "~> 4.1"},
    {:phoenix_live_reload, "~> 1.2", only: :dev},
    {:phoenix_live_view, "~> 1.0.0-rc.1", override: true},
    {:floki, ">= 0.30.0", only: :test},
    {:phoenix_live_dashboard, "~> 0.8.3"},
    {:esbuild, "~> 0.8", runtime: Mix.env() == :dev, targets: :host},
    {:tailwind, "~> 0.2", runtime: Mix.env() == :dev, targets: :host},
    {:heroicons,
     github: "tailwindlabs/heroicons",
     tag: "v2.1.1",
     sparse: "optimized",
     app: false,
     compile: false,
     depth: 1},
    {:telemetry_metrics, "~> 1.0"},
    {:telemetry_poller, "~> 1.0"},
    {:gettext, "~> 0.20"},
    {:jason, "~> 1.2"},
    {:dns_cluster, "~> 0.1.1"},
    {:bandit, "~> 1.5"}
  ]
end
```
{% endcode %}

Copy the Phoenix aliases to the Nerves project.

{% code title="mix.exs" %}
```elixir
def project do
  [
    # ...
    aliases: aliases()
  ]
end

defp aliases do
  [
    setup: ["deps.get", "assets.setup", "assets.build"],
    "assets.setup": ["tailwind.install --if-missing", "esbuild.install --if-missing"],
    "assets.build": ["tailwind my_firmware", "esbuild my_firmware"],
    "assets.deploy": [
      "tailwind my_firmware --minify",
      "esbuild my_firmware --minify",
      "phx.digest"
    ]
  ]
```
{% endcode %}

In `lib/my_firmware/application.ex`, copy the child processes and `config_change` callback. `DNSCluster` can be removed from the list of children.

```elixir
defmodule MyFirmware.Application do
  @moduledoc false

  use Application

  @impl true
  def start(_type, _args) do
    children =
      [
        MyFirmwareWeb.Telemetry,
        {Phoenix.PubSub, name: MyFirmware.PubSub},
        MyFirmwareWeb.Endpoint
      ] ++ children(Nerves.Runtime.mix_target())

    opts = [strategy: :one_for_one, name: MyFirmware.Supervisor]
    Supervisor.start_link(children, opts)
  end

  defp children(:host) do
    []
  end

  defp children(_target) do
    []
  end

  @impl true
  def config_change(changed, _new, removed) do
    MyFirmwareWeb.Endpoint.config_change(changed, removed)
    :ok
  end
end

```

For Nerves projects we like to use an Elixir config that is split out both by target and environment. There is a little more ceremony to set it up, but it provides for much more fine-grained configuration once set up, and will make it easier to migrate over the Phoenix configuration.

Change the Nerves project `config/config.exs` to use the following structure.

```elixir
import Config

# Configuration files are applied in the following order:
#
# 1. config/config.exs
# 2. config/target.exs
# 3. config/target/<platform>.exs
# 4. config/env/<environment>.exs

Application.start(:nerves_bootstrap)

config :my_firmware,
  target: Mix.target(),
  env: Mix.env()

config :logger, backends: [RingLogger]

config :nerves, :firmware, rootfs_overlay: "rootfs_overlay"

config :nerves, source_date_epoch: "1723330274"

if Mix.target() != :host,
  do: import_config "target.exs"

import_config "target/#{Mix.target()}.exs"
import_config "env/#{Mix.env()}.exs"

```

Change the `config` directory to look like the following tree. Move the `host.exs` file to the `target` directory, and create the remaining files. Phoenix config will be brought over in the next step.

```
config
├─ env
│  ├─ dev.exs
│  ├─ prod.exs
│  └─ test.exs
├─ target
│  ├─ bbb.exs
│  └─ host.exs
├─ config.exs
├─ runtime.exs
└─ target.exs
```

Merge `config.exs`

```elixir
import Config

# Configuration files are applied in the following order:
#
# 1. config/config.exs
# 2. config/target.exs
# 3. config/target/<platform>.exs
# 4. config/env/<environment>.exs

Application.start(:nerves_bootstrap)

config :my_firmware,
  target: Mix.target(),
  env: Mix.env(),
  generators: [timestamp_type: :utc_datetime]

config :logger, backends: [RingLogger]

config :nerves, :firmware, rootfs_overlay: "rootfs_overlay"

config :nerves, source_date_epoch: "1723330274"

# Configures the endpoint
config :my_firmware, MyFirmwareWeb.Endpoint,
  url: [host: "localhost"],
  adapter: Bandit.PhoenixAdapter,
  render_errors: [
    formats: [html: MyFirmwareWeb.ErrorHTML, json: MyFirmwareWeb.ErrorJSON],
    layout: false
  ],
  pubsub_server: MyFirmware.PubSub,
  live_view: [signing_salt: "CCcgfn6u"]

# Configure esbuild (the version is required)
config :esbuild,
  version: "0.17.11",
  my_firmware: [
    args:
      ~w(js/app.js --bundle --target=es2017 --outdir=../priv/static/assets --external:/fonts/* --external:/images/*),
    cd: Path.expand("../assets", __DIR__),
    env: %{"NODE_PATH" => Path.expand("../deps", __DIR__)}
  ]

# Configure tailwind (the version is required)
config :tailwind,
  version: "3.4.3",
  my_firmware: [
    args: ~w(
      --config=tailwind.config.js
      --input=css/app.css
      --output=../priv/static/assets/app.css
    ),
    cd: Path.expand("../assets", __DIR__)
  ]

# Configures Elixir's Logger
config :logger, :console,
  format: "$time $metadata[$level] $message\n",
  metadata: [:request_id]

# Use Jason for JSON parsing in Phoenix
config :phoenix, :json_library, Jason

if Mix.target() != :host,
  do: import_config "target.exs"

import_config "target/#{Mix.target()}.exs"
import_config "env/#{Mix.env()}.exs"
```

In `target.exs`, copy over the `Endpoint` config and modify it to run on the hardware. Generate a `secret_base_key` that will be used for development. `server: true` is set here so that the Phoenix server will start as part of the Erlang release bundle. `code_reloader` is disabled on the hardware since the assets won’t change and it could be an attack vector.

```elixir
config :my_firmware, MyFirmwareWeb.Endpoint,
  server: true,
  url: [host: "nerves.local", port: 80, scheme: "http"],
  http: [ip: {0, 0, 0, 0}, port: 80],
  code_reloader: false,
  check_origin: false,
  debug_errors: true,
  secret_key_base: "Si+zfZ2mAG9PptCX0OzAwXh0RzQ757VLFywjD33p0jEQDZFLjCBu33nw8q+QijoY"
```

The web server can also be added to the advertised mDNS services in `target.exs` if it should be discoverable on the network.

```elixir
# Advertise the following services over mDNS.
services: [
  %{
    protocol: "http",
    transport: "tcp",
    port: 80
  }
]
```

Merge `dev.exs`

```elixir
import Config

# Enable dev routes for dashboard and mailbox
config :my_firmware, dev_routes: true

# Do not include metadata nor timestamps in development logs
config :logger, :console, format: "[$level] $message\n"

# Set a higher stacktrace during development. Avoid configuring such
# in production as building large stacktraces may be expensive.
config :phoenix, :stacktrace_depth, 20

# Initialize plugs at runtime for faster development compilation
config :phoenix, :plug_init_mode, :runtime

config :phoenix_live_view,
  # Include HEEx debug annotations as HTML comments in rendered markup
  debug_heex_annotations: true,
  # Enable helpful, but potentially expensive runtime checks
  enable_expensive_runtime_checks: true
```

Merge `prod.exs`

```elixir
# Note we also include the path to a cache manifest
# containing the digested version of static files. This
# manifest is generated by the `mix assets.deploy` task,
# which you should run after static files are built and
# before starting your production server.
config :my_firmware, MyFirmwareWeb.Endpoint,
  cache_static_manifest: "priv/static/cache_manifest.json"

# Do not print debug messages in production
config :logger, level: :info
```

Merge `test.exs`. `code_reloader` is disabled.

```elixir
import Config

# We don't run a server during test. If one is required,
# you can enable the server option below.
config :my_firmware, MyFirmwareWeb.Endpoint,
  http: [ip: {127, 0, 0, 1}, port: 4002],
  secret_key_base: "5JDEbgF+J3b4JxnV+Lnk2Awdk5PcjRcsKP234y3zaUB+T5lqI4NK6W/mKmuJePj/",
  code_reloader: false,
  server: false

# Print only warnings and errors during test
config :logger, level: :warning

# Initialize plugs at runtime for faster test compilation
config :phoenix, :plug_init_mode, :runtime

# Enable helpful, but potentially expensive runtime checks
config :phoenix_live_view,
  enable_expensive_runtime_checks: true
```

Merge `runtime.exs`. Most of the Phoenix file has been configured elsewhere at this point and can be condensed to the following code. This will still use `SECRET_KEY_BASE` in production, which means this environment variable will need to be flashed into the bootloader during the factory provisioning process. Factory provisioning is out of the scope of this guide.

```elixir
import Config

if config_env() == :prod && config_target() != :host do
  secret_key_base =
    System.get_env("SECRET_KEY_BASE") ||
      raise """
      environment variable SECRET_KEY_BASE is missing.
      You can generate one by calling: mix phx.gen.secret
      """

  config :my_firmware, MyFirmwareWeb.Endpoint, secret_key_base: secret_key_base
end
```

Build the assets and firmware, then burn an SD card.

```
MIX_TARGET=bbb mix deps.get
mix assets.build
MIX_TARGET=bbb mix firmware.burn
```

## Background

There have been two prevalent methods for getting a Phoenix web server onto a Nerves embedded device: umbrella projects, and poncho projects. This is often done to host a browser-based user interface, although it can also host an API for a mobile app to connect to the device.

Umbrella projects are an Elixir design pattern to configure and manage multiple applications in a project. These applications are siblings of each other, rather than dependencies. Therefore a top-level project to organize them makes more sense than including them as any particular application’s dependencies. In a Nerves umbrella architecture, the Nerves project generated by `mix nerves.new` needs to be the umbrella application and contain the Nerves configuration. This is because when Nerves bundles the firmware image, you want it to bundle up all the applications below it. The firmware and Phoenix applications would then be siblings under the umbrella.

Poncho projects are a way to maintain the separation between the firmware and web applications, but without the need for the top-level shell application used in an umbrella project. In the poncho architecture, the project generated with `mix nerves.new` contains the Nerves configuration as well as the firmware application. A Phoenix project is generated and added as a firmware application dependency. Firmware and UI can still be maintained as separate projects.

### Challenges

Something to be aware of when working with ponchos is that the Nerves project configuration is the one that is used when bundling the firmware. This means that your nice isolated UI project also has config files that must be duplicated in the Nerves config. A Phoenix configuration isn’t exactly trivial.

The thing that doesn't sit right with us about poncho projects is the reverse dependency issue: A dependency on the Phoenix project is declared in the Nerves project `mix.exs` file. However, the Phoenix application is the one making function calls into the Nerves firmware. Although this physically works due to the way the BEAM compiles files, we feel like this setup is misleading and harder to understand the relationship between components.

There is another elephant in the room as well: The claim for separating the UI from the firmware is for ease of development, without being tied to the hardware. But Nerves can run on the host development machine, firmware and all. We use [resolve](https://github.com/redwirelabs/resolve) to inject virtual components on the host that physically exist on the hardware, like sensors. The other claim is that the UI can be versioned separately from the firmware. Although this is a nice thought, in practice we have seen the UI and firmware developed in lockstep. Agile product development focuses on delivering complete features, and that means a feature being developed in the UI is also going to have its firmware portion completed before that feature is released. This also allows for QA testing the vertical feature slice, rather than having dormant code lurking in the codebase.

### A new approach

Our preferred approach is to run Phoenix directly inside of the Nerves project. This looks very similar to a web-based Phoenix project: there is `lib/project` for the business logic and `lib/project_web` for the UI (presentation layer). The subtle difference in a Nerves firmware project is that instead of `lib/project` being context modules that interact with a database, this is your firmware, gathering data from sensors and interacting with other devices.

The downside to this approach is that it requires merging the output of two project generators. The good news is that it only happens when starting a new project, and becomes more comfortable the more you get familiar with Nerves and Phoenix.
