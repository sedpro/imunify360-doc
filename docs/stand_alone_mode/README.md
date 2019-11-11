# How to run Imunify360 in stand-alone mode on a Linux server

::: tip Note
Prior to version 4.5, Imunify360 supported cPanel, Plesk, and
DirectAdmin control panels only. Starting from v4.5, Imunify360 could
be run on any CloudLinux OS, CentoOS 6/7, RHEL 6/7, and Ubuntu
16.04/18.04 systems without requiring any of these web panels to be
installed. Imunify360 can be run with any web panel, as the UI is
implemented using the single-page application (SPA) approach.
:::

Below you can find the steps to install and run Imunify360, in
stand-alone mode (without panel), or within any hosting panel.


#### What Imunify360 requires to run properly

There are some basic steps to run Imunify360
 as a stand-alone application. They are:

1. Define a way to serve web-based UI
2. Configure a user authentication process
3. Implement integration scripts that provide Imunify360 with
   a necessary information suchs as list of users, domains hosted on
   the server, panel administrators, general panel info
4. Install prerequisites such as ModSecurity, remoteip web server modules
5. Configure ModSecurity integration
6. Configure malware scanner integration


#### How to configure the Imunify360 UI

Create the file `/etc/sysconfig/imunify360/integration.conf` with a
`UI_PATH` option defining the path that will serve web-based UI. For
example:

``` ini
[PATHS]
UI_PATH = /var/www/vhosts/imunify360/imunify360.hosting.example.com/html/im360
```

Imunify360 will automatically copy UI files there during installation/upgrade.

::: tip Note
Ensure that the domain you are going to use for the Imunify360
web-based UI refers to this path, and that there are no other scripts
or files under `UI_PATH`, as they might be overridden by the Imunify360
installation.
:::


#### How to configure authentication for Imunify360 (optional)

Imunify360 can use PAM to authenticate users.
Once the UI is opened, the user sees a sign-in form. The credentials are checked via PAM.

You can specify which PAM service Imunify360 should use with the `SERVICE_NAME` option:

``` ini
[PAM]
SERVICE_NAME = system-auth
```


If it is not specified, the “`system-auth`” service is used.

By default, “`root`” is considered to be the only “`admin`” user.  To
add more administrators, list them in the
`/etc/sysconfig/imunify360/auth.admin` file or specify the `ADMINS`
option in `/etc/sysconfig/imunify360/integration.conf`:

``` ini
[INTEGRATION_SCRIPTS]
ADMINS = /path/to/get-admins-script.sh

```
It should point to an executable file that generates a json file similar to:

<!-- identical to CloudLinux "admins" script
 https://docs.cloudlinux.com/control_panel_integration/#admins
-->

``` json
{
  "data": [
    {
      "name": "admin1",
      "unix_user": "admin",
      "locale_code": "EN_us",
      "email": "admin1@domain.zone",
      "is_main": true
    },
	{
      "name": "admin2",
      "unix_user": "admin",
      "locale_code": "Ru_ru",
      "email": "admin2@domain.zone",
      "is_main": false
    },
  ],
  "metadata": {
    "result": "ok"
  }
}
```


#### How to provide Imunify360 with an actual list of users (optional)

By default, Imunify360 will use Linux system users, limited
by `UID_MIN` and `UID_MAX` from `/etc/login.defs`

If you want to see a specific list of users (note, that all of them
must be real linux users accessible via PAM), you can specify the
`USERS` option in `/etc/sysconfig/imunify360/integration.conf`:

<!--
XXX USERS name differs from USER_LIST_SCRIPT for AV
https://docs.imunifyav.com/stand_alone_mode/#how-to-provide-imunifyav-with-an-actual-list-of-users-optional
-->

``` ini
[INTEGRATION_SCRIPTS]
USERS = /path/to/get-users-script.sh
```


It should point to an executable file that generates a json file similar to (domains are optional):

<!--
XXX data format is incompatible with CL "users" script 
https://docs.cloudlinux.com/control_panel_integration/#users
-->

``` json
{"version": 1, "users": [{"name": "user1", "domains": ["example.com"]}, {"name": "user2"}]}
```

::: tip Note
Any type of executable file is acceptable. For example,
you can use a Python or PHP script.
:::


#### How to provide information about domains

Specify `DOMAINS` option in `/etc/sysconfig/imunify360/integration.conf`:

``` ini
[INTEGRATION_SCRIPTS]
DOMAINS = /path/to/get-domains-script.sh
```

It should point to an executable file that generates a json file similar to:

<!-- it extends Cloudlinux OS `domains` script output
https://docs.cloudlinux.com/control_panel_integration/#domains
-->

``` json
{
  "data": {
    "example.com": {
      "document_root": "/home/username/public_html/",
      "is_main": true,
      "owner": "username",
      "web_server_config_path": "/path/to/example.com/specific/config/to/include",
    },
    "subdomain.example.com": {
      "document_root": "/home/username/public_html/subdomain/",
      "is_main": false,
      "owner": "username",
      "web_server_config_path": "/path/to/subdomain.example.com/specific/config/to/include",
    }
  },
  "metadata": {
    "result": "ok"
  }
}
```

`web_server_config_path` should point to a path that is added as `IncludeOptional` in this domain's virtual host e.g., `/path/to/example.com/specific/config/to/include` path should be added for the `example.com` domain.

#### How to provide information about the control panel

Specify `PANEL_INFO` option in `/etc/sysconfig/imunify360/integration.conf`:

``` ini
[INTEGRATION_SCRIPTS]
PANEL_INFO = /path/to/get-panel-info-script.sh
```

It should point to an executable file that generates a json file similar to:

<!-- identical to Cloudlinux OS `panel_info` output
https://docs.cloudlinux.com/control_panel_integration/#the-list-of-the-integration-scripts
-->

``` json
{
	"data": {
		"name": "SomeCoolWebPanel",
		"version": "1.0.1",
		"user_login_url": "https://{domain}:1111/"
	},
	"metadata": {
		"result": "ok"
	}
}
```

#### How to install Imunify360

##### Prerequisites

-   [ModSecurity 2.9.x](https://modsecurity.org/)
-   [Apache Module mod_remoteip](http://httpd.apache.org/docs/2.4/mod/mod_remoteip.html)

Now everything is ready to install Imunify360.

The installation instructions are the same as for
cPanel/Plesk/DirectAdmin version, and can be found in the technical
documentation:
<https://docs.imunify360.com/installation/> 


#### How to configure ModSecurity integration

Configure [ModSecurity configuration
directives](https://github.com/SpiderLabs/ModSecurity/wiki/Reference-Manual-%28v2.x%29#Configuration_Directives)
(so that it can block):

``` apacheconf
SecAuditEngine = "RelevantOnly"
SecConnEngine = "Off"
SecRuleEngine = "On"
```

Update your your web server config: add `IncludeOptional` that points to
`/etc/sysconfig/imunify360/generic/modsec.conf` file. Create the file
if it doesn't exist, otherwise the web server might not start until
imunify360 is installed (which creates the file).

Set in `integration.conf`:

- `[WEB_SERVER].SERVER_TYPE`​ -- `apache`/`litespeed`
- `[WEB_SERVER].GRACEFUL_RESTART_SCRIPT​` -- a script that restarts the
   web server to be called after any changes in web-server config or modsec rules. 
- `[WEB_SERVER].MODSEC_AUDIT_LOG`​ -- path to ModSecurity audit log file
- `[WEB_SERVER].MODSEC_AUDIT_LOGDIR​` -- path to ModSecurity audit log dir 

Example:

``` ini
[WEB_SERVER]
SERVER_TYPE = apache
GRACEFUL_RESTART_SCRIPT = /path/to/a/script/that/restarts/web-server/properly
MODSEC_AUDIT_LOG = /var/log/httpd/modsec_audit.log
MODSEC_AUDIT_LOGDIR = /var/log/modsec_audit
```


##### How to configure ModSecurity domain level integration

To enable domain-specific ModSecurity configuration, specify
`MODSEC_DOMAIN_CONFIG_SCRIPT` in `integration.conf`:

``` ini
[WEB_SERVER]
MODSEC_DOMAIN_CONFIG_SCRIPT=/path/to/inject/domain/specific/config/script.sh
```

It should point to an executable file that accepts as an input a list
of domain-specific web server settings and injects them into the
server config. The standard input (stdin) is given in the
[jsonlines](http://jsonlines.org/) format similar to:

``` json
{"user": "username", "domain": "example.com", "config": "modsec config text"}
{"user": "another", "domain": "another.example.com", "config": "..."}
```

Each line contains config for a single domain e.g., it may contain
rule tags excluded for the domain.

The script should also restart the web server to apply the
configuration. This should be done so that the script could implement
the check that webserver comes up after config change, and reset
configuration if it doesn't.

If configuration change failed, script should return `1`, and in the
standard error stream (stderr) it should return the reason for
failure. On success, script should return `0`.

In a single run of the script we might update single domain/user, as
well as multiple users (all users) on the system.


#### How to configure malware scanner integration

To scan files uploaded via FTP, configure
[PureFTPd](https://www.pureftpd.org/project/pure-ftpd/). Write in `pure-ftp.conf`:

    CallUploadScript             yes

To scan files for changes (to detect malware) using inotify, configure
which directories to watch and which to ignore in the
`integration.conf` file:

- configure `[MALWARE].BASEDIR` -- a root directory to watch (recursively)
- configure `[MALWARE].PATTERN_TO_WATCH` -- only directories that match this
  ([Python](https://docs.python.org/3/howto/regex.html#regex-howto))
  regex in the `BASEDIR` are actually going to be watched


Examples for already supported panels:

- cPanel

``` ini
BASEDIR = /home
PATTERN_TO_WATCH = ^/home/.+?/(public_html|public_ftp|private_html)(/.*)?$
```
- DirectAdmin

``` ini
BASEDIR = /home
PATTERN_TO_WATCH = ^/home/(.+?|.+?/domains/.+?)/(public_html|public_ftp|private_html)(/.*)?$
```
-  Plesk

``` ini
BASEDIR = /var/www/vhosts/
PATTERN_TO_WATCH = ^/var/www/vhosts/.+?(?<!/.skel|/chroot|/default|/system)(/.+?(?<!/logs|/bin|/chroot|/other-from-VHOST_IGNORE)(/.*)?)?$
```


#### How  to open Imunify360 UI Once Imunify360 is installed

The web-based UI is available via the domain configured in `UI_PATH`.

For example, if
`/var/www/vhosts/imunify360/imunify360.hosting.example.com/html/im360`
is the document root folder for `imunify360.hosting.example.com`
domain, then you could open ImunifyAV with the following URL:

* `https://imunify360.hosting.example.com/` (when you have TLS
certificate configured for the domain) or
* `http://imunify360.hosting.example.com/` 

#### Extended `integration.conf` example for reference

`integration.conf` - a config in which all integration points should
be defined. It contains 2 kinds of fields:
* A simple variable, for example, `[WEB_SERVER].SERVER_TYPE`
* A path to a script that will return the data for Imunify360, for example,
  `[INTEGRATION_SCRIPTS].DOMAINS`.

``` ini
[INTEGRATION_SCRIPTS]
# same as in CloudLinux
PANEL_INFO = /opt/cpvendor/bin/panel_info
USERS = /opt/cpvendor/bin/users
DOMAINS = /opt/cpvendor/bin/vendor_integration_script domains
ADMINS = /opt/cpvendor/bin/vendor_integration_script admins

[PATHS]
UI_PATH = /var/www/vhosts/im360/im360.example-hosting.com/html

[WEB_SERVER]
SERVER_TYPE = apache
GRACEFUL_RESTART_SCRIPT =
/path/to/a/script/that/restarts/web-server/properly
MODSEC_AUDIT_LOG = /var/log/httpd/modsec_audit.log
MODSEC_AUDIT_LOGDIR = /var/log/modsec_audit

[MALWARE]
BASEDIR = /home
PATTERN_TO_WATCH = ^/home/.+?/(public_html|public_ftp|private_html)(/.*)?$
```

Unless otherwise stated, the expected output/error handling for the
integrations scripts follows
<https://docs.cloudlinux.com/control_panel_integration/#expected-structure-of-replies>
interface.


##### Further steps

Detailed instructions on how to use Imunify360 can be found in the
technical documentation: <https://docs.imunify360.com/>


If you have questions, or experience any issues regarding the
product, submit a ticket to our technical support team. For feature
requests, just drop us an email to <feedback@imunify360.com>.

