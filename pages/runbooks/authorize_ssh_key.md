---
title: Authorize SSH key at runtime
layout: page
parent: Runbooks
---

# Authorize SSH key at runtime

To add an authorized SSH key at runtime, connect to the device's console with
a serial cable or OTA platform. At the IEx prompt, replace your public key in
the following snippet and paste it into IEx. You should now be able to SSH into
the device with the new key.

```elixir
key = "<public key>"
Application.put_env(:nerves_ssh, :authorized_keys, [key], persistent: true)
Application.stop(:nerves_ssh); Application.ensure_all_started(:nerves_ssh)
```

The authorized keys will persist after a reboot.

## Background

When building the firmware bundle for a Nerves application (`mix firmware`),
the SSH public key of the build computer is baked into the authorized keys of
the firmware. This is defined in the Nerves application's `config/target.exs`.
This is convenient for development, allowing the computer the firmware was built
on to SSH into the device after it's flashed. If another computer tries to SSH
into the device, it will be blocked and a password prompt will appear. This
computer is locked out of the device.

```text
$ ssh 192.168.1.2
SSH server
Enter password for "nerves"
password: 
```

This is good as a general security practice, but inconvenient when another
computer is legitimately trying to connect to the device. This can be the case
if more than one computer is used for development, multiple developers share
hardware, or keys are baked into (or removed in) a CI pipeline.

Sharing SSH keys is bad practice for security, so it is better to add all of the
authorized keys for the devices that should have SSH access.

## Authorize SSH keys at compile time

The device's authorized keys are set in the project's [`config/target.exs`](https://github.com/nerves-project/nerves_bootstrap/blob/main/templates/new/config/target.exs).

```elixir
config :nerves_ssh, authorized_keys: [...]
```

The source code above this line in `target.exs` searches the development
computer for SSH keys. This can be removed and `authorized_keys` can be set from
another source, like a hard-coded list, a file of authorized keys, or using the
[GitHub SSH key API](https://docs.github.com/en/rest/users/keys?apiVersion=2022-11-28#list-public-ssh-keys-for-the-authenticated-user)
to get the keys of your developers.

> 🛈 Tip
> 
> The Redwire Labs [Nerves project generator](https://github.com/redwirelabs/redwire_tasks)
> creates a project that will pull users' SSH keys from GitHub at compile time
> if the environment variable `SSH_GITHUB_USERS` is set to a space-separated
> list of GitHub user names.
