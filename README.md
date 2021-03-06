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

## Hardware configuration

Production how to defines a hardware configuration of the node, which may be considered as an example instead of mandatory.

## Tor

we are not using `tor` at the moment

## Bitcoin core

We are not using options:
- dbcache=3700: because, this affects amount of RAM cache, so this value is expected to be fine-tuned on a node with fixed resources
- maxconnections=1337: because at the moment we are only use outbound connections, which are limited to 11. Affects RAM footprint as well.

## Elements

We don't use Elements Core at the moment

## Mempool configs

- `"MINED_BLOCKS_CACHE": 144` - we don't use such option, because there is no such option in https://github.com/mempool/mempool/blob/master/backend/src/config.ts, as it was removed in Oct 2020;
- `"SPAWN_CLUSTER_PROCS": 0` - as  this value is the default value;

## Nginx configs

At the moment, we are reusing the same nginx config, that had been provided by mempool's developers with addition of enabling per-network routing, dependently on enabled networks. The only difference is that we split this config into parts in order to use those parts for specifying in appropriate nixos options for nginx.

At the same time, production how to uses additional nginx features, like rate-limiting of the requests, which we are not using (at least yet)

# Deployment into DigitalOcean

There are 2 options of deployment to DigitalOcean:

1. through the NixOS being uploaded to DO as a 'custom' image;
2. taking over the another Linux distro with `nix-infect` ( https://github.com/elitak/nixos-infect );

Both options will achieve the result in 3 steps:

1. Initial OS takeover (to get clean NixOS running);
2. attaching additional volumes; 
3. deployment of the mempool instances on NixOS.


## Taking over Debian droplet with `nix-infect`

1. Goto 'Create' -> 'Droplet';

2. select 'CentOS 8' distribution. (CentOS 8 uses XFS as a rootfs, which is more preferred for NixOS than Ext4 due to `/nix/store` could have a big amount of inodes, which Ext4 handles not so well);

3. choose your plan. The more, the better, of course, but initially, I recommend to choose the smallest, as it will create the smallest storage volume for droplet. It will be possible to resize CPU/RAM/Storage later and, in fact, we will do that at some point for CPU and RAM;

4. 6 additional storage volumes will be needed for mempool setup, but droplet creation wizard does not allow to create more than 1 volume and/or select name for it. So, it will need to be done after droplet creation, but you need to confirm, that you are will create droplet in a datacenter, that support creating additional volumes.

choose DC;

5. In the "select additional options" form, select "user data" and paste this snippet in the appeared input:

```
#cloud-config
write_files:
- path: /etc/nixos/host.nix
  permissions: '0644'
  content: |
    {pkgs, ...}:
    {
      environment.systemPackages = with pkgs; [ vim git python3 ];
    }
runcmd:
  - curl https://raw.githubusercontent.com/elitak/nixos-infect/master/nixos-infect | PROVIDER=digitalocean NIXOS_IMPORT=./host.nix NIX_CHANNEL=nixos-21.05 bash 2>&1 | tee /tmp/infect.log
```

7. select ssh keys, that will be used to access as a root user;

8. select meaningful droplet name and tags;

9. hit `Create Droplet`.

The droplet will be created and then, DigitalOcean agent will start the process of taking over the OS. After that, droplet will be rebooted.

After reboot, droplet will have running clean NixOS with selected ssh-keys and vim and git installed. 

## Attaching additional volumes

We need to add 6 more volumes (format: volume-name (volume-size)) :
- bitcoind-mainnet (440 GiB);
- electrs-mainnet ( 90 GiB);
- bitcoind-testnet ( 40 GiB);
- electrs-testnet ( 10 GiB);
- bitcoind-signet ( 1 GiB);
- electrs-signet ( 500 MiB).

1. Go to "Volumes";
2. for each volume, declared above do: 
2.1. "Create volume";
2.2. Enter appropriate name and size;
2.3. choose the droplet to add to;
2.4. choose "Automatically Format & Mount" and "XFS" as a file system;
2.5. hit "Create Volume";
3. Login into droplet with `ssh -A root@<droplet_IP>` and confirm, that it is already running NixOS:

```
cat /etc/os-release | grep NAME | grep NixOS
```

if not, you can track the process of taking over of OS with:

```
tail -f /tmp/infect.log
```

droplet should reboot when taking over will be finished. Then, you will need to clean the ssh fingerprint for this host with `ssh-keygen -R <droplet_IP` and relogin again with `ssh -A root@<droplet_IP>`;

4. add appropriate mount points for created volumes with this command:

```
echo '--- a/hardware-configuration.nix  2021-09-21 00:32:37.078564115 +0000
+++ b/hardware-configuration.nix       2021-09-21 00:37:58.127504552 +0000
@@ -3,5 +3,11 @@
   imports = [ (modulesPath + "/profiles/qemu-guest.nix") ];
   boot.loader.grub.device = "/dev/vda";
   boot.initrd.kernelModules = [ "nvme" ];
-  fileSystems."/" = { device = "/dev/vda1"; fsType = "xfs"; };
+  fileSystems."/" = { device = "/dev/vda1"; fsType = "xfs"; options = [ "noatime" "discard"]; };
+  fileSystems."/mnt/bitcoind-mainnet" = { device = "/dev/disk/by-id/scsi-0DO_Volume_bitcoind-mainnet"; fsType = "xfs"; options = [ "noatime" "discard"]; };
+  fileSystems."/mnt/electrs-mainnet" = { device = "/dev/disk/by-id/scsi-0DO_Volume_electrs-mainnet"; fsType = "xfs"; options = [ "noatime" "discard"]; };
+  fileSystems."/mnt/bitcoind-testnet" = { device = "/dev/disk/by-id/scsi-0DO_Volume_bitcoind-testnet"; fsType = "xfs"; options = [ "noatime" "discard"]; };
+  fileSystems."/mnt/electrs-testnet" = { device = "/dev/disk/by-id/scsi-0DO_Volume_electrs-testnet"; fsType = "xfs"; options = [ "noatime" "discard"]; };
+  fileSystems."/mnt/bitcoind-signet" = { device = "/dev/disk/by-id/scsi-0DO_Volume_bitcoind-signet"; fsType = "xfs"; options = [ "noatime" "discard"]; };
+  fileSystems."/mnt/electrs-signet" = { device = "/dev/disk/by-id/scsi-0DO_Volume_electrs-signet"; fsType = "xfs"; options = [ "noatime" "discard"]; };
 }
 ' | patch -p1  -d /etc/nixos/
```

5. build and apply configuration with:

```
nixos-rebuild switch
```


Now you can confirm with `mount | grep --count /mnt`, that there are 6 mount points in the `/mnt`.

## Deployment of the mempool instances

Now it is time to replace config of the fresh NixOS with config from the current repo. Fot this:

1. Resize droplet's CPU and RAM to at least 4 vCPUs and 8 GiB of RAM to perform an initial sync. For this:
1.1. shutdown droplet;
1.2. goto Droplet -> Resize droplet;
1.3. choose approprite plan;
1.4. start the droplet

2. login to the Droplet with droplet IP from DO dashboard:

```
ssh -A root@<droplet_IP>
```

3. clone the repo with

```
git clone --recursive https://github.com/dambaev/nixos-test-repo.git
```

4. move the content of the repo into `/etc/nixos/`:

```
mv nixos-test-repo/* /etc/nixos/
mv nixos-test-repo/.git* /etc/nixos/
```
confirm, that `git` determines `/etc/nixos/` as a repo by doing:

```
cd /etc/nixos
git pull
```

it should not report any errors

5. fill the `/etc/nixos/local.hostname.nix` with hostname:

```
echo "\"$(hostname)\"" > /etc/nixos/local.hostname.nix
```

6. generate secrets for bitcoin rpc and dbs:

```
/etc/nixos/gen-psk.sh
```

7. rebuild and apply config:

```
nixos-rebuild switch
```

WARNING: currently, this step will replace os users and their public ssh keys

8. now wait for bitcoind and electrs instances to sync the data and you can resize the droplet back to your load
