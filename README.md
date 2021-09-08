# Brief

This repo contains nixos configuration for showcase of NixOS. NixOS is following the goal to perform whole OS config in one configuration file. In order to achieve this, NixOS relies on:
1. a "pure" language 'Nix';
2. a package manager 'nix', which goal is to make build process to be referential transparent (ie, to use only defined dependencies in order to produce the result);
3. modeling a whole system as a 'set' of options.


# Structure

NixOS allows us to import configuration files ("nix expressions"), so we can split configuration in modules and import them with `imports` option.

So there are those configuration files used:

`configuration.nix` - this is the 'main' configuration module in NixOS. All other modules are being imported from this file.
`auto-apply-config.nix` - contains routine for performing automatical fetching config from git repo, building and switching to this configuration.
`local_settings.nix` - this file included as an example of how to provide per-instance local settings, such that you will be able to use one repo for multiple NixOS instances and some of them may use specific settings (like static IP address/routes and etc).
`os-users.nix` - contains set of users, that are allowed to access NixOS instance by ssh-public-key.
`overlays/` - directory, which is supposed to contain number of git submodules with nix-expressions, which are supposed to be imported from `configuration.nix`.

# Submodules

`overlays/mempool-overlay` is a git submodule, which contains an extension to NixOS configuration options, which adds possibility to build and enable mempool-backend instances and mempool-frontend as well. 
`overlays/electrs-overlay` is a git submodule, which contains an extension to NixOS configuration options, which adds possibility to build and enable electrs instances

# Difference with mempool's production how-to

Mempool developers provide example of configs, that are being used for production instance of the mempool.space. Here goes a list of things, that are different from those production examples:

## electrs

the original Mempool's README defines a `Electrum Server (romanz/electrs)` as a dependency, but the production example is using `Blockstream/electrs`, which is the fork of the former. There are differences with arguments support between them. We are using `Electrum Server (romanz/electrs)` and there is a NixOS overlay for it: https://github.com/dambaev/electrs-overlay
