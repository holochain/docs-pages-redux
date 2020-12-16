# More Info on Nix

We use the NixOS toolkit to build consistent development, testing, and deployment environments for Holochain Core and apps.

The Nix package manager runs on many operating systems. NixOS is also the OS we use in our automated testing and our HoloPorts.

The main components of the tooling for Holochain development are:

* The [Rust](https://rust-lang.org) programming language
* [Node.JS](https://nodejs.org) and [npm](https://npmjs.com)
* Cryptographic libraries
* Common automations and scripts

It is important that these remain consistent across compatible apps and the Holochain Core, so you can get your work done without fighting package and compiler issues.

The Holonix repository tracks standard, shared dependencies for all of the scenarios in which we use NixOS. Typically you won’t need to interact with Holonix directly; all you need to do is [install Nix](https://nixos.org/nix/download.html) and start Holonix using the install command

```bash
nix-shell https://github.com/holochain/holonix/archive/release-0.0.85.tar.gz
```

!!! note "What happened to holochain.love?"
    If you've been following Holochain development for a while, you might be wondering what happened to the quick install command that used the much shorter `https://holochain.love` to download the Holonix package.

    This still exists, but at present we're updating all of our dev tools (including the Holonix package at `holochain.love`) to use the [new version of Holochain](https://redux.developer.holochain.org/holochain-rsm-guidance.html). So we recommend using the GitHub release URL for the last version of Holonix to include the old Holochain-Redux., which is 0.0.85.

The main Nix tool used in Holochain development workflows is `nix-shell`, a managed Bash shell that overlays a new environment and set of tools on top of your existing environment.

Many popular package management tools only target a single OS. Nix package management supports most OSes.

The full suite of Nix tooling is broad and deep. There’s even a dedicated OS and functional programming language. Learn more with the [NixOS Wiki](https://nixos.wiki/wiki/Main_Page) or the [Pills](https://nixos.org/nixos/nix-pills/) Tutorial. The community IRC chat at `#nixos` on freenode is active and helpful.

## nix-shell

While working on Holochain, you will usually have an active `nix-shell` to run commands. This shell overlays Holochain-specific configuration on top of your existing shell---environment variables, binaries, and libraries---giving you a consistent development environment to build Holochain apps. All this setup will be cleaned up automatically when you close the shell.

If you want to re-enter the shell to do more work, or create multiple terminals to work in, you'll need to re-enter the `nix-shell`. The files are cached locally on your machine, so they will not be re-downloaded or rebuilt the next time you enter the shell.

## Three ways to install and enter the Holonix environment

Nix is configured by `default.nix` files. Running the command

```bash
nix-shell <path_or_url_to_nix_config_file>
```

will configure the environment and enter the newly created shell for you. On the initial run, and any time a component has been updated, it will take some time to download and build. It gets much faster on subsequent runs.

### Specific versions

If you want to install a specific version of Holochain or the developer tools, it's a bit more tricky because Holonix version numbers don't exactly correspond to the version numbers of the Holochain release they contain. You'll need to find the specific Holonix version number (the [Dev Pulse blog](https://blog.holochain.org/tag/dev-pulse/) is a good source for this) and enter this command:

```bash
nix-shell https://github.com/holochain/holonix/archive/release-<x.x.x>.tar.gz
```

### Per-project pinned releases

Every DNA project you create with `hc init` has its own `default.nix` file that targets the version of Holochain and the HDK that it was created with. To start `nix-shell` with that specific version, go into the project directory and type:

```bash
nix-shell
```