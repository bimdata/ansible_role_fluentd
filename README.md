Ansible role Fluentd
=========

This role installs and configures Fluentd.

Requirements
------------

* This role is only tested on Debian 10.x (Buster).

Role Variables
--------------

This role tries to keep the same default configuration as if you manually install
Fluentd.
You can find more information about each option in the [Fluentd documentation](
https://docs.fluentd.org/).

Variables used for the installation:

| Variables                    | Default value                                      | Description                                               |
|--------------------------|--------------------------------------------------------|-----------------------------------------------------------|
| fluentd_prerequisites        | ['apt-transport-https', 'curl', 'gnupg']           | List of package that need to be installed before Fluentd. |
| fluentd_major_version        | 5                                                  | Major version of Fluentd you want to install.             |
| fluentd_apt_key_url          | https://packages.treasuredata.com/GPG-KEY-td-agent | The APT key for the Fluentd package.                      |
| fluentd_apt_repos_url        | "https://packages.treasuredata.com/{{ fluentd_major_version }}/{{ ansible_distribution | lower }}/{{ ansible_distribution_release | lower }}" | The APT repository address needed to install Fluentd.     |
| fluentd_apt_repos_suites     | "{{ ansible_distribution_release \| lower }}"      | The APT repos suites.                                     |
| fluentd_apt_repos_components | ['contrib']                                        | The APT repos components.                                 |
| fluentd_pkg_name             | fluent-package                                     | The Fluentd APT package name.                             |
| fluentd_pkg_version:         | ""                                                 | Install a specific version of the package.                |
| fluentd_pkg_version_hold     | "{{ fluentd_pkg_version \| default(False) \| ternary(True, False) }}" | Lock package version to prevent accidental update. By default, `True` if `fluentd_pkg_version` is defined, `False` otherwise. |
| fluentd_svc_name             | fluentd                                            | The Fluentd service name to start/stop the daemon.        |

Variables used to remove legacy stuff (for example if install before the renaming td-agent -> fluentd)

| Variables                     | Default value                            | Description                                           |
|-------------------------------|------------------------------------------|-------------------------------------------------------|
| fluentd_apt_cleanup_legacy    | false                                    | Remove old keys and old APT sources if true.          |
| fluentd_apt_key_legacy_id     | BEE682289B2217F45AF4CC3F901F9177AB97ACBE | ID of the old GPG key to remove from keyring.         |
| fluentd_naming_cleanup_legacy | false                                    | Remove old service / conf / apt with td-agent name.   | 
| fluentd_pkg_name_legacy       | td-agent                                 | Pkg name that will be remove by legacy cleaner.       |
| fluentd_svc_name_legacy       | td-agent                                 | Service name that will be remove by legacy cleaner.   |
| fluentd_conf_directory_legacy | /etc/td-agent/                           | Conf directory that will be remove by legacy cleaner. |

Variables used for the general configuration:

| Variables                 | Default value           | Description                                               |
|---------------------------|-------------------------|-----------------------------------------------------------|
| fluentd_plugins           | []                      | List of plugins to install.                               |
| fluentd_gem_exec_path     | /usr/sbin/fluent-gem    | Gem binary path to install fluentd plugins.               |
| fluentd_system            | {}                      | fluentd system configuration.                             |
| fluentd_conf_directory    | "/etc/fluent"           | Fluentd configuration directory.                          |
| fluentd_conf_file_path    | "{{ fluentd_conf_directory }}/fluentd.conf" | Fluentd configuration file path.      |
| fluentd_include_directory | *Undefined*             | Can be use to include other configuration files. It will add `@include {{ fluentd_include_directory }}*.conf` in the conf. |

For `fluentd_system`, each key is used as a configuration option name and values as values.
You can see all the avaible options in the [Fluentd documentation](https://docs.fluentd.org/deployment/system-config)

`fluentd_plugins` is a list of the plugins that will be installed by `fluentd_gem_exec_path`.
You can specify the wanted version for each plugin:
```
fluentd_plugins:
  - name: fluent-plugin-prometheus
    version: 2.0.0
```

Variables for log processing:

| Variables               | Default value                                         | Description                                              |
|-------------------------|-------------------------------------------------------|----------------------------------------------------------|
| _fluentd_sources:       | "{{ lookup('template', './lookup/get_sources.j2') }}" | List of dictionaries defining all source configurations. |
| _fluentd_filters:       | "{{ lookup('template', './lookup/get_filters.j2') }}" | List of dictionaries defining all filter configurations. |
| _fluentd_labels:        | "{{ lookup('template', './lookup/get_labels.j2') }}"  | List of dictionaries defining all label configurations.  |
| _fluentd_matches:       | "{{ lookup('template', './lookup/get_matches.j2') }}" | List of dictionaries defining all match configurations.  |

**In most cases, you should not modify these variables.**
Templating is used to build these lists with other variables.
* `_fluentd_sources` will aggregate all the variables whose name matches this regex: `^fluentd_.+_source(s)?$`.
* `_fluentd_filters` will aggregate all the variables whose name matches this regex: `^fluentd_.+_filter(s)?$`.
* `_fluentd_labels` will aggregate all the variables whose name matches this regex: `^fluentd_.+_label(s)$`.
* `_fluentd_matches` will aggregate all the variables whose name matches this regex: `^fluentd_.+_match(es)?$`.

Each variables matching these regexes must be:
  - a dictionary defining one source/filter/label/match **or**
  - a list of dictionaries defining one or more sources/filters/labels/matches.

Each dictionary is used to define one `<source>`, `<filter>`, `<label>` or `<match>`
section in the fluend configuration file. Each configuration section is configured
with key/value couples, si the dictionary's keys are used as configuration keys and
values as values.

Some keys have special behaviours:
* `type` will be replace by `@type`.
* `id` will be replace by `@id`.
* `label` will be replace by `@label`.
* `log_level` will be replace by `@log_level`.
* `_section_args` is a special key used to define stuff like tags that will be match by the configuration.
* For label configuration (variables matching `^fluentd_.+_label(s)$`), the key `name` is used to defined
the name of the label, like this `<label @my_name>`.
* Dictionaries can't have duplicate keys, so each key that needed to be repeat can be preceded by a prefix
that matches this regex `^[0-9]+__`. The prefix'll be remove when the configuration is generated.

For example:
```
fluentd_default_forward_source:
  type: forward
  id: forward
  bind: 0.0.0.0
  port: 24224

fluentd_systemd_filters:
  - _section_args: "systemd.**"
    type: systemd_entry
    id: parse_systemd
    field_map: '{"_HOSTNAME": "host"}'
    field_map_strict: "false"
    fields_lowercase: "true"
    fields_strip_underscores: "true"
  - _section_args: "systemd.unknown"
    type: record_modifier
    id: parse_unknown
    record:
      comm: unknown

fluentd_archive_label:
  name: archive
  filter:
    _section_args: "**"
    type: record_modifier
    whitelist_keys: host, message
  0__match:
    _section_args: "syslog.*.4 syslog.*.10"
    type: file
    id: archives_syslog_auth
    path: /logs/archives/${host}/auth-%Y%m%d
  1__match:
    _section_args: "**"
    type: "null"
    id: archives_drop
```

will create:
```
<source>                                                                                                  
  @type forward                                                                                           
  @id forward                                                                                             
  bind 0.0.0.0                                                                                            
  port 24224                                                                                              
</source>

<filter systemd.**>                                                                                       
  @type systemd_entry                                                                                     
  @id parse_systemd                                                                                       
  field_map {"_HOSTNAME": "host"}                                                                         
  field_map_strict false                                                                                  
  fields_lowercase true                                                                                   
  fields_strip_underscores true                                                                           
</filter>                                                                                                 
<filter systemd.unknown>                                                                                  
  @type record_modifier                                                                                   
  @id parse_unknown                                                                                       
  <record>                                                                                                
    comm unknown                                                                                          
  </record>                                                                                               
</filter>

<label @archive>
  <filter **>
    @type record_modifier
    whitelist_keys host, message
  </filter>
  <match syslog.*.4 syslog.*.10>
    @type file
    @id archives_syslog_auth
    path /logs/archives/${host}/auth-%Y%m%d
  </match>
  <match **>
    @type null
    @id archives_drop
  </match>
</label>

```

It allows you to define variables in multiple group_vars and cumulate them for
hosts in multiples groups without the need to rewrite the complete list.


Dependencies
------------

None


License
-------

BSD

Author Information
------------------

[BIMData.io](https://bimdata.io/)
