# Ansible Role: Apache 2.x

[![Build Status](https://travis-ci.org/geerlingguy/ansible-role-apache.svg?branch=master)](https://travis-ci.org/geerlingguy/ansible-role-apache)

An Ansible Role that installs Apache 2.x on RHEL/CentOS and Debian/Ubuntu.

## Requirements

None.

## Role Variables

Available variables are listed below, along with default values (see `vars/main.yml`):

    apache_enablerepo: ""

The repository to use when installing Apache (only used on RHEL/CentOS systems). If you'd like later versions of Apache than are available in the OS's core repositories, use a repository like EPEL (which can be installed with the `geerlingguy.repo-epel` role).

    apache_listen_port: 80

The port on which apache should be listening. Useful if you have another service (like a reverse proxy) listening on port 80.

    apache_vhosts:
      # Additional optional properties: 'serveradmin, extra_parameters'.
      - {servername: "local.dev", documentroot: "/var/www/html"}

Add a set of properties per virtualhost, including `servername` (required), `documentroot` (required), `serveradmin` (optional: the admin email address for this server), and `extra_parameters` (you can add whatever you'd like in here).

Note that this role doesn't configure SSL support out of the box; you would need to add in additional tasks to listen on port 443 and add your own VirtualHost directives for SSL. This may be improved in the future :)

## Dependencies

None.

## Example Playbook

    - hosts: webservers
      vars_files:
        - vars/main.yml
      roles:
        - { role: geerlingguy.apache }

*Inside `vars/main.yml`*:

    apache_listen_port: 8080
    apache_vhosts:
      - {servername: "example.com", documentroot: "/var/www/vhosts/example_com"}

On Debian/Ubuntu hosts, if you get the error `Unable to fetch some archives, maybe run apt-get update or try with --fix-missing?`, You should add a task to make sure your apt_cache is up to date, like:

    - name: Update apt cache if needed.
      apt: update_cache=yes cache_valid_time=3600

## License

MIT / BSD

## Author Information

This role was created in 2014 by [Jeff Geerling](http://jeffgeerling.com/), author of [Ansible for DevOps](http://ansiblefordevops.com/).