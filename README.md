# rsnapshot

#### Table of Contents

1. [Overview](#overview)
2. [Module Description](#module-description)
3. [Differences between this and other modules](#differences-between-this-and-other-modules)
4. [Setup - The basics of getting started with rsnapshot](#setup)
    * [What rsnapshot affects](#what-rsnapshot-affects)
    * [Setup requirements](#setup-requirements)
    * [Beginning with rsnapshot](#beginning-with-rsnapshot)
5. [Usage - Configuration options and additional functionality](#usage)
    * [Configuring the Server](#configuring-the-server)
    * [Configuring the Client](#configuring-the-client)
6. [Reference](#reference)
    * [Public Classes](#public-classes)
    * [Defines](#public-defines)
    * [Private Classes](#private-classes)
    * [Private Defines](#private-defines)
7. [Resources](#resources)
    * [Define rsnapshot::backup](#[define-rsnapshotbackup)
    * [Class rsnapshot::client](#class-rsnapshotclient)
    * [Class rsnapshot::server](#class-rsnapshotserver)
    * [Define rsnapshot::server::config](#define-rsnapshotserverconfig)
8. [Development](#development)

## Overview

This module manages backups using rsnapshot.

## Module Description

> rsnapshot is a filesystem snapshot utility based on rsync. rsnapshot makes it
easy to make periodic snapshots of local machines, and remote machines over ssh.
The code makes extensive use of hard links whenever possible, to greatly reduce
the disk space required.

This module installs and configures the rsnapshot module, which is a backup tool
that utilizes rsync to create fast incremental backups as well as a hardlink
system which makes all incremental backups work as full ones.

This module makes it easy to manage a backup server based off of rsnapshot while
utilizing common Puppet patterns.

## Differences between this and other modules.

* **Client specific options instead of enforced globals.** Rather than rely on a
  single configuration file and monolithic backup runs this module uses stand
  alone configurations for each host. Besides being more resilient to errors,
  this enables unique client settings- for instance, using different retain
  settings for different hosts.

* **Backup Point resource type for true Puppet style backup control.** Rather
  than defining each backup point in the class file, the `backup` resource
  allows backups to be defined next to the profiles that need it.

* **Support for SSH without root access.** In most cases root login is not
  available over ssh for security reasons, so this module relies instead on
  having it's own unique user with restricted sudo access to give it the needed
  access to perform backups.

* **Support for automatic key sharing.** The client machine will automatically
  receive the ssh key from the server and user that it is backing up to.

* **Locked down ssh accounts.** All ssh accounts are locked down. SSH keys can
  only by used by the single backup host, without access to unneeded features
  like x-forwarding. Commands allowed by the ssh key are limited to specific
  wrapper scripts installed by this module.

* **Sender only rsync.** One of the biggest threats with rsync access is the
  potential to overwrite existing files on the system to gain unauthorized
  access. This module uses a wrapper script around rsync on the client side
  to make it a read only user.


## Setup

### What rsnapshot affects

* Installs rsync and rsnapshot on server machine.
* Installs rsync on client machine.
* Creates rsnapshot configuration files for each client on the server machine.
* Creates cron jobs for each client backup job.
* Installs wrapper scripts on the client machine to improve security.
* Creates directory for storing backups on the server.
* Creates an ssh key pair on the server if needed.
* Transfers SSH public key from server to client to enable ssh
  login.
* Creates backup user and group on client machine.
* Adds backup user to sudo.

### Setup Requirements

* PuppetDB needs to be installed for SSH keys to automatically transfer.
* Storeconfigs needs to be enabled for configurations defined on the client side
  to be installed on the backup server.
* Multiple puppet runs (client, then server, then client again) need to occur
  for all resources to be created on both servers.

### Beginning with rsnapshot

On the backup server (backups.example.com) include the `rsnapshot::server` class
and tell it where to store the backups.

```puppet
class { 'rsnapshot::server': }
```

On the machine you want to back up include the `rsnapshot::client` class and
tell it which server to back up to and what directories to back up.

```puppet
class { 'rsnapshot::client':
  server      => 'backups.example.com',
}
```
That's it! A secure backup user will be created on the client, with the
appropriate user, ssh key, and permissions, and that machine will get it's
configuration pushed to the backup server.

## Usage

### Configuring the Server

Settings in the server class are passed to all backup configurations that end up
on that server.

This class can be included without any parameters and the defaults should work.

```puppet
class { 'rsnapshot::server':
  config_path            => '/etc/rsnapshot',
  backup_path            => '/backups/rsnapshot',
  log_path               => '/var/log/rsnapshot',
  user                   => 'root',
  no_create_root         => 0,
  verbose                => 2,
  log_level              => 3,
  link_dest              => 1,
  sync_first             => 0,
  use_lazy_deletes       => 0,
  rsync_numtries         => 1,
  stop_on_stale_lockfile => 0,
  du_args                => '-csh'
}
```


### Configuring the Client

Settings in the client class are specific to that one client node. The
parameters in this class will get exported to a backup server and merged with
it's parameters to build the client specific configuration.

This class has 1 required parameter- the backup `server`, which should be an
fqdn, and an optional parameter for an array of `directories` to back up. Additional options, such as
retain rules or cronjob times, can be overridden as needed.

When the retain values are set to zero, no cron entry for that specific
period is created.

```puppet
class { 'rsnapshot::client':
  server             => 'backups.example.com',
  directories        => [
    '/etc',
    '/home',
    '/root'
  ],
  user                => 'backshots',
  remote_user         => 'root',
  backup_hourly_cron  => '*/2',
  backup_time_minute  => fqdn_rand(59, 'rsnapshot_minute'),
  backup_time_hour    => fqdn_rand(23, 'rsnapshot_hour'),
  backup_time_weekday => 6,
  backup_time_dom     => 15,
  cmd_preexec         => undef,
  cmd_postexec        => undef,
  cmd_client_rsync    => '/usr/bin/rsync',
  cmd_client_sudo     => '/usr/bin/sudo',
  retain_hourly       => 6,
  retain_daily        => 7,
  retain_weekly       => 4,
  retain_monthly      => 3,
  one_fs              => undef,
  rsync_short_args    => '-a',
  rsync_long_args     => '--delete --numeric-ids --relative --delete-excluded'
  ssh_args            => undef,
  use_sudo            => true,
  setup_sudo          => true,
  push_ssh_key        => true,
  wrapper_path        => '/opt/rsnapshot_wrappers/',
}
```

# Back Up Pre and Post Actions

When backing up clients hosting services like databases, you may want to run a
script to snapshot or quiesce the service.  You can do this by specifying pre
or post wrapper actions.  These will be run on the client immediately before or
after the rsync operation.

For example, to export the contents of the puppetdb database before running a
backup of your puppetmaster:

```puppet

class profiles::puppetmaster {
  rsnapshot::client {
    cmd_wrapper_preexec  => ['/usr/sbin/puppetdb export -o /root/puppetdb.export --port 8083'],
    cmd_wrapper_postexec => ['rm -f /root/puppetdb.export'],
  }
}
```

## Reference

### Public Classes

* [`rsnapshot::client`](#class-rsnapshotclient)
* [`rsnapshot::server`](#class-rsnapshotserver)

### Defines

* [`rsnapshot::backup`](#define-rsnapshotbackup)
* [`rsnapshot::server::config`](#define-rsnapshotserverconfig)

### Private Classes

* `rsnapshot::client::install`: Installs needed packages on client side.
* `rsnapshot::client::user`: Sets up client side user and permissions.
* `rsnapshot::client::wrappers`: Adds wrapper scripts to client machine.
* `rsnapshot::server::cron_script`: Adds a shell script wrapper to backup machines one at a time and associated CRON tasks.
* `rsnapshot::server::install`: Installs needed packages on server side.
* `rsnapshot::params` Contains default parameters used by this module.

### Private Defines

* `rsnapshot::server::backup_config`: Gets thrown and collected by the backup
   and config types.

## Development

Contributions are always welcome. Please read the [Contributing Guide](CONTRIBUTING.md)
to get started.
