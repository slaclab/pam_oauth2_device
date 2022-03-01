# PAM module for OAuth 2.0 Device flow

This is a PAM module that lets you log in via SSH to servers using OpenID Connect credentials, instead of SSH Keys or a username and password combination.

It uses the OAuth2 Device Flow, which means that during the login process, you will click a link and log in to your OpenID Connect Provider, which will then authenticate you for the SSH session. 

This module will then check if you're in the right group(s) or have a specified username, and allow or deny access.

A demo video is avaliable here: https://drive.google.com/file/d/1WzDRL0RFDXfvUgabbXNzBppV-DKXyUN1/view?usp=sharing

This code was originally developed by Mazarykova Univerzita and has been refactored by UKRI-STFC.

## Build

The upstream build uses basic `make` and we have stuck with this for compatibility reasons.
The two basic targets are `make` and `make test`; the latter will build (some of) the tests and run them.
Note that at present some of the integration tests are failing.


### Build on Scientific Linux or CentOS7

```
yum install epel-release
yum install openldap-devel
yum install libcurl-devel
yum install pam-devel
yum install libldb-devel
yum install http://ftp.scientificlinux.org/linux/scientific/7x/external_products/softwarecollections/yum-conf-softwarecollections-2.0-1.el7.noarch.rpm
yum install devtoolset-8 # this is needed as we need a more up to date g++ version than is supplied by default in SL repos.
scl enable devtoolset-8 bash
git clone https://github.com/stfc/pam_oauth2_device.git
cd pam_oauth2_device/
make
cp pam_oauth2_device.so /lib64/security/pam_oauth2_device.so
cp config_template.json config.json
```

### Build on Debian 10/Ubuntu Focal (20.04)


## Installation

To install the module, copy `pam_oauth2_device.so` into the PAM modules directory (usually with permissions 0755).

On Debian-based systems, this would be `/lib/x86_64-linux-gnu/security` whereas CentOS and related flavours would use `/usr/lib64/security`.  If in doubt, check `dpkg --L libpam-modules` or `rpm -ql pam` respectively.


## User names

Usernames are mentioned several times in this document and could probably get a bit confusing.  This section attempts to give a short explanation.

For every user there are *three* usernames, which can be distinct.

The **local user name** is the name the user uses in the ssh login, as in `ssh fred@example.com` where the local user name is `fred`.  This is the name that is passed into the PAM module for authentication.

The **remote user name** is the corresponding name for the user as held by the IAM system.  Once the user has successfully authenticated to IAM, IAM publishes a "userinfo" structure with the user's name and email address and other attributes that IAM can assert.  Within this structure, the PAM module can pick an attribute to use as the remote user name (using the `username_attribute` option).

The **account name** is the name of the local Unix account that the user is mapped into once they have authenticated.  By default, it is the same as the local user name.

### Comprehensive Example

As above, user Fred Bloggs logs in with `ssh fred@example.com`.  The host at `example.com` asks Fred to authenticate.  Once successfully, it calls out to IAM to obtain the userinfo structure.  From this it picks the attribute specified with `username_attribute` in the configuration, `preferred_username`, say.  Let's say the value of `preferred_username` of Fred's userinfo structure is `bloggs`.  Additionally, the userinfo structure contains the list of groups "users", "iris" and "cloud".

Throughout the rest of this section, it is assumed that Fred has authenticated successfully to IAM.

If the **cloud** section is configured and `access` is true, a local file configured as the `metadata_file` is read.  This file should contain the structure

```
{"project_id": "fleeps"}
```

The module adds the string `fleeps` to the endpoint (with a slash) and calls the server (with *no* client authentication) to see what is at the endpoint.  It expects a JSON structure as response, structured like

```
{"groups": ["wop", "fap", "foo", "users"]}
```

If one of these groups matches Fred's groups as returned by the userinfo structure (it does here, "users"), then Fred is considered authorised.  An additional check is made whether the local username plus suffix equals the remote username.  If this check and the cloud group membership both pass, then Fred is considered authorised (the username check would fail in this example, because no suffix can make 'fred' equal to 'bloggs'.)


If the **group** section is configured and `access` is set to true, a check will be made whether the configured value for `service_name` is one of Fred's groups.  Note the service name is single valued.  Additionally, as for the cloud section, the local username plus suffix must equal the remote username.

If Fred's remote username were `fred_fleeps` then it *would* match the local username (`fred`) if the suffix were configured as `_fleeps`.

If Fred is not authorised through  the cloud or group sections, either because the check failed or they were not enabled, then a configured usermap is consulted.  This is written straight into the configuration file (it should probably be in its own file at some point), so would be suitable only for a smallish number of users.

This usermap is in the **users* section which expects a JSON object mapping the *remote* username to an array of permissible local usernames. Thus, the same user could have multiple local logins using this method.  If the local username is found here, Fred is considered authorised.  No suffix is used in this section.

If Fred is not authorised through any of these methods, the module falls back to an LDAP lookup (if the **ldap** section is configured).  The LDAP query takes a configured filter and substitutes the *remote* username for a `%s` part of the filter, and queries a configured attribute.

If the filter is `(&(objectClass=user)(cn=%s))` then `bloggs` is substituted in our example, and a target attribute (configured with `attr`) is queried from the LDAP server.  For example, if `attr` has the value `uid` then the equivalent of

```
ldapsearch -x -H ldap://host -b base '(&(objectClass=user)(cn=bloggs))' uid
```

is run and the result is compared with the local username, `fred`.  If these match (no suffix is used), then Fred is again considered authorised.


## SSH Configuration

You MUST edit the configuration before this module will work!

Make sure the module works correctly before changing your SSH config or you may be locked out!  See Testing below.

Edit `/etc/pam.d/sshd` and comment out the other `auth` sections (unless you need MFA or something else).

```
auth required pam_oauth2_device.so /etc/pam_oauth2_device/config.json
```

Edit `/etc/ssh/sshd_config` and make sure that the following configuration options are set

```
ChallengeResponseAuthentication yes
UsePAM yes
```

```
systemctl restart sshd
```

## Configuration config.json

The configuration should be located at `/etc/pam_oauth2_device/config.json`.  The file in the distro `config_template.json` should get you started.

As the name suggests, the file is in JSON, so it is recommended to check it with a JSON validator like `jq` after editing it (the PAM module will refuse to load an invalid JSON file, but you will not see this error till runtime.)

The file is divided into the following sections (each encoded as a JSON object).  A table below summarises all the options.

**oauth** - required section for your OAuth2 client configuration. You will be able to get most configuration attributes from your IAM administrator. The ```local_username_suffix``` option is used within the cloud and group configuration sections. If added, it appends this suffix to all username checks. e.g. if the suffix is set to "_irisiam" then it is expected that usernames on the system will follow the format: "<iam-username>_irisiam".  This feature makes it possible to have account names on the system all ending with the same suffix, thus avoiding clashes with any other system account.  The `username_attribute` describes which attribute from IAM's Userinfo structure is used as IAM's view of the username.

**tls** describes the TLS (transport layer security, previously known as SSL) parameters as they apply to both HTTPS and LDAPS connections.

**ldap** - optional section which activates the LDAP logic (see below for description).

**qr** - allowed correction levels are

  * -1 - no QR printed
  * 0 - low
  * 1 - medium
  * 2 - high

If no **qr** information is provided, one isn't printed.

**group** - if enabled, on login the users IAM groups will be checked against the group specified. If they are in this group, they will be allowed in with their IAM username (plus a suffix if set above). e.g. if the group is set to "my-service", any IAM user in the group "my-service" will be allowed access. If a user is ONLY in a subgroup, e.g. "my-service/special", they will NOT be allowed access.

**cloud** - this is designed for VMs in the STFC cloud. For this to work, the module will need to be installed on an OpenStack VM, and be enabled. The **group** section should be disabled. All OpenStack VMs will be part of a project on the OpenStack service. If the user logging in is in an IRIS IAM group which the VM project is a part of, then they will be allowed into the shared account specified.
e.g. The user is trying to access the shared "centos" account, which has been specified using the "username" attribute. The VM is in the IRIS AAI project in OpenStack, which maps to the IAM group "iris-iam-admins". If the user trying to gain access is in the "iris-iam-admins" IAM group, then they will be allowed access to the "centos" account.

The "endpoint" attribute should be set to the location of the irisiam-mapper.py CGI script.

**users** - user mapping. From claim configured in *username_attribute* to the local account name. e.g. you could map the IAM username "willfurnell" or "will.furnell@stfc.ac.uk" to the local account "root". This must be done on a per-user basis, so isn't suitable for large numbers of users.

**client_debug** - boolean value; if set to true, additional debugging information is printed to stdout by the module.  Useful for debugging (a future release will change this to a loglevel rather than a boolean, as internally the module distinguishes between debug, info, warn, error)

## Testing the module works

You are advised to do this before making changes to your main SSH config.  There are two tests to do which are recommended to do in the order described here.

### Preparing for the tests

It is recommended that you create a hardlink to your `sshd` called (for example) `pamsshd`, e.g. `/usr/local/sbin/pamsshd`.  This means you can have a PAM configuration for `pamsshd` which is different from the normal `sshd`.

In this case you can copy `/etc/pam.d/sshd` to `/etc/pam.d/pamsshd` and edit the latter, leaving the former to log you back into the system if something goes wrong.  Also copy `/etc/ssh/sshd_config` to `/etc/ssh/pamsshd_config` so you can edit the configuration for `pamsshd` separately.

Note that testing *requires* that you install the module in the system location and you have the configuration set up in `/etc/pam_oauth2_device/config.json` and `/etc/pam.d/` and `/etc/ssh/`.

### Test 1: pam tester

Follow instructions above, and additionally install `pamtester`.

Run
```
pamtester -v pamtester localusername authenticate
```
and follow the onscreen prompts.  Here, `localusername` refers to your local user name so replace it with whatever your name is.

You can check `/var/log/secure` or `/var/log/auth.log` to find what's wrong if there are errors authenticating.

### Test 2: sshd

While pamtester tests the authenticate section, you should try a proper ssh login from another host.  If you created `pamsshd` as above (and copied configuration as described above), start it with

```
/usr/local/sbin/pamsshd -f /etc/ssh/pamsshd_config -p 2222 -d
```

This should start `sshd` with the name `pamsshd` listening on port 2222.  Now try to log in from another host (bearing in mind the port should be open for incoming tcp).  On the other host, run `ssh -p 2222 localusername@myhost` where `localusername` is the local user name and `myhost` is the host running `pamsshd`.

Again check the logs as in the previous tests.


## Overview of Configuration Options

Note that values can be listed as *required* in a section but the whole section can be omitted.  For example, if LDAP is not used, it is possible to omit the entire LDAP section.  If it is *not* omitted, the value of `host` MUST be present.  The value can be an empty string, in which case the LDAP section is again bypassed.

| Section | Attribute | Req'd? | Value | Default |
| --- | --- | --- | --- | --- |
| `oauth` | (section) | Yes | Configuration for OIDC client | - |
| `oauth` | `client` | Yes | Object containing `id` and `secret` | None |
| `oauth` | `scope` | Yes | OIDC scope(s) | None |
| `oauth` | `device_endpoint` | Yes | OIDC device endpoint | None |
| `oauth` | `token_endpoint` | Yes | OIDC token endpoint | None |
| `oauth` | `userinfo_endpoint` | Yes | OIDC userinfo endpoint | None |
| `oauth` | `username_attribute` | Yes | Attribute from userinfo to use as remote username | None |
| `oauth` | `local_username_suffix` | Yes | Suffix (see username section) | empty string |
| --- | --- | --- | --- | --- |
| `tls` | (section) | No | TLS parameters | - |
| `tls` | `ca_path` | Yes | Repository with hashed names of trusted CA certs | `/etc/grid-security/certificates` |
| --- | --- | --- | --- | --- |
| `ldap` | (section) | No | LDAP parameters | - |
| `ldap` | `host` | Yes | LDAP host URL (ldap or ldaps schema) | None |
| `ldap` | `basedn` | Yes | LDAP base | None |
| `ldap` | `user` | No | LDAP client username | empty string |
| `ldap` | `passwd` | No | LDAP client password | empty string |
| `ldap` | `scope` | No | LDAP search scope | subtree |
| `ldap` | `filter` | Yes | LDAP search filter with %s for remote username | None |
| `ldap` | `attr` | Yes | Attribute expected to contain the local username | None |
| `ldap` | `preauth` | No | preauth bypass check | disabled |
| --- | --- | --- | --- | --- |
| `cloud` | (section) | No | Cloud section | - |
| `cloud` | `access` | Yes | Check enabled (true/false)? | None |
| `cloud` | `endpoint` | Yes | Endpoint to query project group | None |
| `cloud` | `username` | Yes | Currently not used, but required... | None |
| `cloud` | `metadata_file` | No | Location of metadata file | See below |
| --- | --- | --- | --- | --- |
| `group` | (section) | No | Group membership check | - |
| `group` | `access` | Yes | Check enabled (true/false)? | None |
| `group` | `service_name` | Yes | Name of group | None |
| --- | --- | --- | --- | --- |
| `users` | (section) | No | Usermap section | - |
| `users` | username | Yes | array of local usernames | None |
| --- | --- | --- | --- | --- |
| `qr` | (section) | No | QR code error correction | Disable QR |
| `qr` | `error_correction_level` | Error correction (0 to 2) | Disable QR |
| --- | --- | --- | --- | --- |
| `client_debug` | (entry) | No | true/false for debug | debug off |
| --- | --- | --- | --- | --- |

### Deprecated?

- Future releases should change the `client_debug` to loglevel.
  - Additionally, adding `debug` to the PAM config should enable debug, like for a normal PAM module
- The metadata file called `project_id` currently has a backwards compatible default of `/mnt/context/openstack/latest/meta_data.json`

