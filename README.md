Ansible Role: rsyslog
=========

An Ansible Role that install and configures rsyslog on RedHat/CentOS, Debian/Ubuntu and Fedora Linux systems.

Requirements
------------

No requirements.

Role Variables
--------------

**Available variables are listed below, along with default values (see defaults/main.yml):**

    rsyslog_port: 514

Port number will be use to configure system sending and receiving logs

    rsyslog_enable_log_host: no

To accept logs from remote systems.

    #rsyslog_log_server: loghost.example.com

Specify remote host to send the logs to

    #rsyslog_retry_count: -1

Number of times to retry communication with log host server. Set as -1 to do infinite retries if host is down

    #rsyslog_max_disk_space: '1g'

Max disk space limit (use as much as possible)

    #rsyslog_save_on_shutdown: 'on'

Save messages to disk on shutdown (on/off)

    #rsyslog_queue_filename: 'fwdRule1'

Unique name prefix for spool files

**The variables listed below do not need to be changed for targeted systems (see vars/main.yml):**

    rsyslog_conf_path: /etc/rsyslog.conf

Configuration file path.

    rsyslogd_path: /etc/rsyslog.d

Configuration directory path.

    rsyslog_package:

List of packages to be installed to provide rsyslog functionality.

    rsyslog_file:

Name of the file (located at `files` directory) to be used as target configuration.

Dependencies
------------

No dependencies.

Example Playbook
----------------

    - hosts: servers
      vars:
        rsyslog_log_server: loghost.example.com
        rsyslog_retry_count: -1
        rsyslog_max_disk_space: '1g'
        rsyslog_save_on_shutdown: 'on'
        rsyslog_queue_filename: 'fwdRule1'
      roles:
         - { role: guidugli.rsyslog }

License
-------

MIT / BSD

Author Information
------------------

This role was created in 2020 by Carlos Guidugli.
