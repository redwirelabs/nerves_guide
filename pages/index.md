---
title: Home
layout: home
nav_order: 1
---

# The Redwire Labs Nerves Guide

This is a guide to developing commercial products with [Nerves](https://nerves-project.org/)
based on the lessons learned by the team at [Redwire Labs](https://www.redwirelabs.com/).
Product development can be a challenging journey, so we are sharing this
resource with the community so that you can avoid the pitfalls we have seen
and shortcut the path to success.

*Redwire Labs is a product development agency that specializes in commercial
IoT products built with Nerves and Elixir.*

## Why we use Nerves

In our experience, Nerves has been an excellent technology that expedites the
development of connected products, and is built on a platform that supports
robustness and reliability.

Nerves is the framework for running an [Elixir](https://elixir-lang.org/)
application on embedded Linux. It uses [Buildroot](http://buildroot.org/) to
create a custom Linux system that's tailored to your project and boots the
system into the application (as process 1), opposed to running a Linux distro,
which tends to be heavier weight and comes with extra cruft. Nerves also
provides facilities to update firmware in the field, and an A/B partition scheme
that allows for rolling back in the case of a failed firmware update.

Elixir is used as the main application programming language. From a firmware
development perspective, this allows developers to reason about the system at a
higher level, avoiding the need to worry about things like memory management and
segmentation faults. However, developers still have the low-level tools they
need to interface with hardware, like bitwise operations. From the perspective
of developing a connected product, Elixir is a language that can be used for
both the firmware and backend server, making it easier for developers to
understand each side of the system. Our full stack engineers are even able to
develop features in vertical slices through the firmware and backend codebases.

The [BEAM VM](https://www.erlang-solutions.com/blog/erlangs-virtual-machine-the-beam/)
runs the Elixir application, and is known for its use in developing low-latency,
distributed, and fault-tolerant systems. It also comes with [OTP](https://www.erlang.org/doc/),
which can be thought of as the standard library. OTP provides much more
functionality out of the box than most other languages, like [state machines](https://www.erlang.org/doc/man/gen_statem),
[directed graphs](https://www.erlang.org/doc/man/digraph), and [ETS tables](https://www.erlang.org/doc/man/ets)
(in-memory database).
