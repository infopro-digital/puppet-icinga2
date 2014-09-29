puppet-icinga2
==========

Table of Contents
-----------------

1. [Overview - What is the Icinga 2 module?](#overview)
2. [Module Description - What does the module do?](#module-description)
3. [Setup - The basics of getting started with the Icinga 2 module](#setup)
4. [Usage - How to use the module for various tasks](#usage)
5. [Reference - The classes and defined types available in this module](#reference)
6. [Limitations - OS compatibility, etc.](#limitations)
7. [Development - Guide for contributing to the module](#development)
8. [Contributors - List of module contributors](#contributors)

[Overview](id:overview)
--------

This module installs and configures the [Icinga 2 monitoring system](https://www.icinga.org/icinga2/). It can also install and configure [NRPE](http://exchange.nagios.org/directory/Addons/Monitoring-Agents/NRPE--2D-Nagios-Remote-Plugin-Executor/details) on client systems that are being monitored by an Icinga 2 server.

[Module Description](id:module-description)
-------------------

Coming soon...

[Setup](id:setup)
-----

This module should be used with Puppet 3.6 or later. It may work with earlier versions of Puppet 3 but it has not been tested.

This module requires Facter 2.2 or later, specifically because it uses the `operatingsystemmajrelease` fact.

This module requires the [Puppet Labs stdlib module](https://github.com/puppetlabs/puppetlabs-stdlib).

For Ubuntu systems, this module requires the [Puppet Labs apt module](https://github.com/puppetlabs/puppetlabs-apt).

On EL-based systems (CentOS, Red Hat Enterprise Linux, Fedora, etc.), the [EPEL package repository](https://fedoraproject.org/wiki/EPEL) is required.

If you would like to use the `icinga2::object` defined types as [exported resources](https://docs.puppetlabs.com/guides/exported_resources.html), you'll need to have your Puppet master set up with PuppetDB. See the Puppet Labs documentation for more info: [Docs: PuppetDB](https://docs.puppetlabs.com/puppetdb/)

###Server requirements

Icinga 2 requires either a [MySQL](http://www.mysql.com/) or a [Postgres](http://www.postgresql.org/) database.

Currently, this module does not set up any databases. You'll have to create one before installing Icinga 2 via the module.

If you would like to set up your own database, either of the Puppet Labs [MySQL](https://github.com/puppetlabs/puppetlabs-mysql) or [Postgres](https://github.com/puppetlabs/puppetlabs-postgresql) modules can be used. 

The example below shows the [Puppet Labs Postgres module](https://github.com/puppetlabs/puppetlabs-postgresql) being used to install Postgres and create a database and database user for Icinga 2:

<pre>
  class { 'postgresql::server': }

  postgresql::server::db { 'icinga2_data':
    user     => 'icinga2',
    password => postgresql_password('icinga2', 'password'),
  }
</pre>

For production use, you'll probably want to get the database password via a [Hiera lookup](http://docs.puppetlabs.com/hiera/1/puppet.html) so the password isn't sitting in your site manifests in plain text.

[Usage](id:usage)
-----

###Server usage

To install Icinga 2, first set up a MySQL or Postgres database.

Once the database is set up, use the `icinga2::server` class with the database connection parameters to specify

<pre>
#Install Icinga 2:
class { 'icinga2::server': 
  server_db_type => 'pgsql',
  db_host => 'localhost'
  db_port => '5432'
  db_name => 'icinga2_data'
  db_user => 'icinga2'
  db_password => 'password',
}
</pre>

When the `server_db_type` parameter is set, the right IDO database connection packages are automatically installed and the schema is loaded.

**Note:** For production use, you'll probably want to get the database password via a [Hiera lookup](http://docs.puppetlabs.com/hiera/1/puppet.html) so the password isn't sitting in your site manifests in plain text:

<pre>
#Install Icinga 2:
class { 'icinga2::server':
  server_db_type => 'pgsql',
  db_host => 'localhost'
  db_port => '5432'
  db_name => 'icinga2_data'
  db_user => 'icinga2'
  db_password => hiera('icinga_db_password_key_here'),
}
</pre>

You'll also need to add an IDO connection object that has the same database settings and credentials as what you entered for your `icinga2::server` class.

You can do this by applying either the `icinga2::object::idomysqlconnection` or `icinga2::object::idopgsqlconnection` class to your Icinga 2 server, depending on which database you're using.

An example `icinga2::object::idopgsqlconnection` class is below:

<pre>
icinga2::object::idopgsqlconnection { 'postgres_connection':
   target_dir => '/etc/icinga2/features-enabled',
   target_file_name => 'ido-pgsql.conf',
   host             => '127.0.0.1',
   port             => 5432,
   user             => 'icinga2',
   password         => 'password',
   database         => 'icinga2_data',
   categories => ['DbCatConfig', 'DbCatState', 'DbCatAcknowledgement', 'DbCatComment', 'DbCatDowntime', 'DbCatEventHandler' ],
}
</pre>

In a future version, the module will automatically create the IDO connection objects.

**Note:** If you will be installing NRPE or the Nagios plugins packages with the `icinga2::nrpe` class on a node that also has the `icinga2::server` class applied, be sure to set the `$server_install_nagios_plugins` parameter in your call to `icinga2::server` to `false`:

<pre>
#Install Icinga 2:
class { 'icinga2::server':
  ...
  server_install_nagios_plugins => false,
  ...
 }
</pre>

This will stop the `icinga2::server` class from trying to install the plugins pacakges, since the `icinga2::nrpe` class will already be installing them and will prevent a resulting duplicate resource error.

If you would like to install packages to make a `mail` command binary available so that Icinga 2 can send out notifications, set the `install_mail_utils_package` parameter to **true**:

<pre>
  class { 'icinga2::server': 
    ...
    install_mail_utils_package => true,
    ...
  }
</pre>

###NRPE usage

To install NRPE and allow the local machine and Icinga 2 servers (or Icinga 1 or plain old Nagios servers) with various IP addresess to connect:

<pre>
class { 'icinga2::nrpe':
  nrpe_allowed_hosts => ['10.0.1.79', '10.0.1.80', '10.0.1.85', '127.0.0.1'],
}
</pre>

**Note:** If you would like to install NRPE on a node that also has the `icinga2::server` class applied, be sure to set the `$server_install_nagios_plugins` parameter in your call to `icinga2::server` to `false`:

<pre>
#Install Icinga 2:
class { 'icinga2::server': 
  server_db_type => 'pgsql',
  server_install_nagios_plugins => false,
 }
</pre>

This will stop the `icinga2::server` class from trying to install the plugins pacakges, since the `icinga2::nrpe` class will already be installing them and will prevent a resulting duplicate resource error.

###Object type usage

This module includes several defined types that can be used to automatically generate Icinga 2 format object definitions. They function in a similar way to [the built-in Nagios types that are included in Puppet](http://docs.puppetlabs.com/guides/exported_resources.html#exported-resources-with-nagios).

####Exported resources

Like the built-in Nagios types, the Icinga 2 objects in this module can be exported to PuppetDB as virtual resources and collected on your Icinga 2 server.

Nodes that are being monitored can have the `@@` virtual resources applied to them:

<pre>
@@icinga2::object::host { $::fqdn:
  display_name => $::fqdn,
  ipv4_address => $::ipaddress_eth0,
  groups => ['linux_servers', 'mysql_servers'],
  vars => {
    os              => 'linux',
    virtual_machine => 'true',
    distro          => $::operatingsystem,
  },
  target_dir => '/etc/icinga2/objects/hosts',
  target_file_name => "${fqdn}.conf"
}
</pre>

Then, on your Icinga 2 server, you can collect the exported virtual resources (notice the camel casing in the class name):

<pre>
#Collect all @@icinga2::object::host resources from PuppetDB that were exported by other machines:
Icinga2::Object::Host <<| |>> { }
</pre>

Unlike the built-in Nagios types, the file owner, group and mode of the automatically generated files can be controlled via the `target_file_owner`, `target_file_group` and `target_file_mode` parameters:

<pre>
@@icinga2::object::host { $::fqdn:
  display_name => $::fqdn,
  ipv4_address => $::ipaddress_eth0,
  groups => ['linux_servers', 'mysql_servers'],
  vars => {
    os              => 'linux',
    virtual_machine => 'true',
    distro          => $::operatingsystem,
  },
  target_dir        => '/etc/icinga2/objects/hosts',
  target_file_name  => "${fqdn}.conf"
  target_file_owner => 'root',
  target_file_group => 'root',
  target_file_mode  => '644'
}
</pre>

####`undef` and default object values

Most of the object parameters *in the Puppet module* are set to **undef**.

This means that they will not be added to the rendered object definition files.

**However**, this doesn't mean that the values are undefined in Icinga 2. Icinga 2 itself has built-in default values for many object parameters and falls back to them if one isn't present in an object definition. See the docs for individual object types in [Configuring Icinga 2](http://docs.icinga.org/icinga2/latest/doc/module/icinga2/toc#!/icinga2/latest/doc/module/icinga2/chapter/configuring-icinga2) for more info about which object parameters have what default values.

[Reference](id:reference)
---------

Classes:

Coming soon...

Defined types:

Coming soon...

[Limitations](id:limitations)
------------

Coming soon...

[Development](id:contributors)
------------

Coming soon...

[Contributors](id:contributors)
------------

Coming soon...