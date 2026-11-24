# Fail2ban for Nextcloud over Tailscale

This guide documents a general-purpose setup for using Fail2ban to detect repeated failed Nextcloud login attempts when Nextcloud is accessed through Tailscale. It covers logging, filtering, jail configuration, testing, and the important distinction between **detection** and **firewall enforcement**.

> **Privacy note:** Replace all IP addresses, hostnames, domains, usernames, and file paths with your own values. Do not commit private Tailscale addresses, public IP addresses, domain names, usernames, or real log excerpts to a public repository.

## Overview

Fail2ban watches application logs for repeated authentication failures and can apply a firewall action when a threshold is reached. Nextcloud can write authentication failures to a log file, and Fail2ban can parse those entries to identify the source IP address.

With Tailscale, Fail2ban can still detect and count failed logins from a device’s Tailscale IP. However, a UFW, iptables, or nftables ban may not reliably prevent access if the Tailscale traffic path bypasses the firewall rule that Fail2ban creates. For Tailscale-only access, Tailscale ACLs, device authorization, and MFA are usually the stronger enforcement controls.

## Prerequisites

- A working Nextcloud instance.
- Nextcloud accessible through a Tailscale device or tailnet.
- Fail2ban installed on the Nextcloud host.
- UFW installed and enabled on the Nextcloud host.
- Sudo access to the Nextcloud host.
- A separate test device with a Tailscale IP address.

Install Fail2ban if necessary:

```bash
sudo apt update
sudo apt install fail2ban
```

Confirm that UFW is active:

```bash
sudo ufw status verbose
```

## Nextcloud logging

Nextcloud must write failed login attempts to a file that Fail2ban can read.

Edit the Nextcloud configuration file:

```bash
sudo nano /var/www/nextcloud/config/config.php
```

The actual path may differ depending on the installation method. Inside the existing `$CONFIG` array, add or confirm these settings:

```php
'log_type' => 'file',
'logfile' => '/path/to/nextcloud-data/nextcloud.log',
'loglevel' => 2,
```

Example for a data directory stored on external storage:

```php
'log_type' => 'file',
'logfile' => '/mnt/external/nextcloud/data/nextcloud.log',
'loglevel' => 2,
```

- `log_type` must be `file` for file-based logging.
- `logfile` must point to a file Fail2ban can read.
- `loglevel = 2` corresponds to `WARN` and is generally sufficient for login-failure detection.
- Avoid leaving `loglevel = 0` permanently, because debug logging can create large log files and expose more operational detail.

Nextcloud’s logging documentation describes `loglevel = 2` as the default `WARN` level and recommends using debug logging only temporarily for troubleshooting. 

## Verify the log file

After a failed login attempt, check that Nextcloud created or updated the log file:

```bash
sudo tail -f /mnt/external/nextcloud/data/nextcloud.log
```

Attempt an incorrect password from another device, then look for an entry containing text similar to:

```text
Login failed
```

The entry should include a remote address or IP field. Do not paste real usernames, real IP addresses, or full log lines into a public document.

## Create the Fail2ban filter

Create a custom filter:

```bash
sudo nano /etc/fail2ban/filter.d/nextcloud.conf
```

Add a filter that matches the failed-login pattern produced by your Nextcloud version. The exact regular expression can vary between Nextcloud releases, so test it against a real log entry.

A simple starting point is:

```ini
[Definition]
failregex = ^.*Login failed.*Remote IP: <HOST>.*$
ignoreregex =
```

A more specific example may be needed if your Nextcloud log format differs:

```ini
[Definition]
failregex = ^.*"remoteAddr":"<HOST>".*Login failed.*$
ignoreregex =
```

Use the version that matches the actual log format. The important part is `<HOST>`, which tells Fail2ban where to extract the offending IP address.

Test the filter against the real log file:

```bash
sudo fail2ban-regex /mnt/external/nextcloud/data/nextcloud.log /etc/fail2ban/filter.d/nextcloud.conf
```

A successful result should report matched lines and identify the IP addresses extracted by `<HOST>`.

## Create the Fail2ban jail

Create a jail-specific configuration file:

```bash
sudo nano /etc/fail2ban/jail.d/nextcloud.local
```

Add:

```ini
[nextcloud]
enabled = true
port = http,https
filter = nextcloud
logpath = /mnt/external/nextcloud/data/nextcloud.log
maxretry = 5
findtime = 10m
bantime = 1h
backend = polling
```

Explanation:

| Setting | Purpose |
|---|---|
| `enabled` | Activates the jail. |
| `port` | Defines the ports associated with the ban action. |
| `filter` | Points to `/etc/fail2ban/filter.d/nextcloud.conf`. |
| `logpath` | Points to the Nextcloud log file. |
| `maxretry` | Number of failures before a ban. |
| `findtime` | Time window in which failures are counted. |
| `bantime` | How long a ban remains active. |
| `backend = polling` | Tells Fail2ban to monitor the log file by polling. |

With `maxretry = 5`, Fail2ban bans an IP after it accumulates five matching failures within `findtime`. Fail2ban reads logs containing password-failure reports and bans the corresponding IP addresses. 

## Restart and verify Fail2ban

Validate the configuration:

```bash
sudo fail2ban-client -t
```

Restart Fail2ban:

```bash
sudo systemctl restart fail2ban
```

Check that the jail is active:

```bash
sudo fail2ban-client status nextcloud
```

Expected output includes:

```text
Status for the jail: nextcloud
|- Filter
|  |- Currently failed: 0
|  |- Total failed: 0
|  `- File list: /mnt/external/nextcloud/data/nextcloud.log
`- Actions
   |- Currently banned: 0
   |- Total banned: 0
   `- Banned IP list:
```

## Test failed-login detection

From a separate Tailscale device, attempt to sign in to Nextcloud using a valid username but an incorrect password several times.

Then check Fail2ban:

```bash
sudo fail2ban-client status nextcloud
```

After enough failures, the Tailscale IP should appear in the banned list:

```bash
sudo fail2ban-client get nextcloud banned
```

This confirms that:

- Nextcloud is logging failed attempts.
- The Fail2ban filter matches the log format.
- Fail2ban extracts the Tailscale IP.
- The jail reaches the configured threshold.
- Fail2ban records the ban.

## Check UFW rules

List UFW rules:

```bash
sudo ufw status
```

Show detailed status and default policies:

```bash
sudo ufw status verbose
```

Show numbered rules:

```bash
sudo ufw status numbered
```

Search for a banned Tailscale IP:

```bash
sudo ufw status numbered | grep "100.x.y.z"
```

Replace `100.x.y.z` with the banned Tailscale IP.

If no UFW rule appears, Fail2ban may be using its default iptables or nftables action rather than UFW. Fail2ban’s default ban actions are commonly implemented through iptables or nftables, while UFW’s status output shows UFW-managed rules. 

## Check iptables or nftables

Check iptables:

```bash
sudo iptables -S | grep -i f2b
sudo iptables -L -n -v | grep -i f2b
```

Check nftables:

```bash
sudo nft list ruleset | grep -i -A10 -B5 f2b
```

If either command shows the banned IP, Fail2ban created the ban outside UFW.

## Configure Fail2ban to use UFW

To make future bans appear in UFW, edit:

```bash
sudo nano /etc/fail2ban/jail.local
```

Add or modify the `[DEFAULT]` section:

```ini
[DEFAULT]
banaction = ufw
```

Example:

```ini
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1
bantime = 1h
findtime = 10m
maxretry = 5
banaction = ufw
```

Restart Fail2ban:

```bash
sudo systemctl restart fail2ban
```

Fail2ban’s UFW action adds a reject rule for each banned address, and those rules can appear in UFW’s status output while UFW is active. 

## Remove a current ban

To remove a specific banned IP immediately:

```bash
sudo fail2ban-client set nextcloud unbanip 100.x.y.z
```

Replace `100.x.y.z` with the actual banned Tailscale IP.

Whitelisting an IP prevents future bans but does **not** clear an existing ban. To prevent future bans for a trusted device, add its IP to `ignoreip` in the `[DEFAULT]` section:

```ini
ignoreip = 127.0.0.1/8 ::1 100.x.y.z
```

Then restart Fail2ban:

```bash
sudo systemctl restart fail2ban
```

## Why access may continue after a ban

A Fail2ban ban does not necessarily terminate an already-established browser session or immediately log a user out of Nextcloud. It adds a firewall rule intended to prevent new connections from the banned source IP.

A UFW or iptables/nftables rule may not affect Tailscale traffic if:

- The traffic enters through a Tailscale-specific interface.
- The routing path bypasses the firewall chain used by the ban action.
- The jail only blocks `http` and `https` ports.
- The rule is created outside UFW.
- The connection was established before the ban was applied.

Fail2ban’s default jail action normally applies to the ports configured by the jail, rather than every port on the server. 

## Recommended security model

For a Nextcloud instance accessed only through Tailscale, use layered controls:

| Layer | Control | Purpose |
|---|---|---|
| Application | Nextcloud logging | Records failed login attempts. |
| Detection | Fail2ban | Counts repeated failures and identifies source IPs. |
| Firewall | UFW, iptables, or nftables | Can block selected traffic paths, depending on configuration. |
| Network | Tailscale ACLs or grants | Controls which tailnet devices may reach the server. |
| Identity | MFA and strong passwords | Reduces the impact of credential guessing. |
| Device management | Device approval or removal | Prevents unauthorized devices from accessing the tailnet. |

Tailscale uses encrypted WireGuard-based peer connections and provides policy controls for which source and destination pairs may communicate. 

## Useful commands

Check Fail2ban status:

```bash
sudo fail2ban-client status nextcloud
```

List banned IPs:

```bash
sudo fail2ban-client get nextcloud banned
```

Unban an IP:

```bash
sudo fail2ban-client set nextcloud unbanip 100.x.y.z
```

Test a filter against a log file:

```bash
sudo fail2ban-regex /path/to/nextcloud.log /etc/fail2ban/filter.d/nextcloud.conf
```

Check UFW:

```bash
sudo ufw status numbered
```

Check iptables:

```bash
sudo iptables -S | grep -i f2b
```

Check nftables:

```bash
sudo nft list ruleset | grep -i -A10 -B5 f2b
```

## Key takeaway

Fail2ban can successfully **detect and log** failed Nextcloud access attempts from a Tailscale IP. Whether it can **block** that IP depends on the firewall action and the traffic path. For Tailscale-only access, use Fail2ban for observability and defense-in-depth, and use Tailscale ACLs, device authorization, and MFA as the primary enforcement controls.

