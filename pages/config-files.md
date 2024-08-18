# Config files

An Elixir project configuration resides in the `<project>/config` directory. When creating a new Nerves project, the official Nerves project generator creates configuration files for the host (development computer) and target (hardware). While this makes things simple for people learning Nerves, we prefer a more configurable, granular approach for commercial projects.

For professional firmware projects, there are two axes we take into consideration: environment, and target. Environment (dev, prod, test) deals with the context in which the firmware is running. For example, in dev we may want to see debug logs over the serial console, whereas in prod where we don't have access to the serial port, we only want to write errors to disk to reduce flash wear. Target, on the other hand, is the hardware platform that the firmware is running on. This could be a development kit, production PCB, or even the computer used to develop the code or run CI. Since different targets have different capabilities, we need to turn various functionality on/off, select different drivers, or inject mocks depending on the target the firmware is running on.

To do this, we add a directory for environment configuration files, and a directory for target configuration files. The top level config directory includes `config.exs` which applies to all environments and targets, `target.exs` which applies to all targets, and `runtime.exs` which is able to access environment variables at runtime instead of compile time. The file structure looks like this:

```
config
├── env
│   ├── dev.exs
│   ├── prod.exs
│   └── test.exs
├── target
│   ├── host.exs
│   └── imx8.exs
├── config.exs
├── runtime.exs
└── target.exs
```

## Mix Task

It can be a fair amount of work to set up these config files by hand, so the `mix red.nerves.new` task is available from the [Redwire Tasks](https://github.com/redwirelabs/redwire\_tasks) package.

