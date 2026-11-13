# SPNEGO Role for Ansible and [Ludus](https://ludus.cloud)

This role deploys a Squid forward proxy using Active Directory
Kerberos/SPNEGO authentication and configures domain-joined Windows clients to
discover it through a PAC file. For a complete example, see
`spnego-proxy-range-config.yml`.

## Installation in [Ludus](https://ludus.cloud)

```sh
git clone https://github.com/Synzack/ludus_spnego_proxy
ludus ansible role add -d ./ludus_spnego_proxy
ludus range config set -f spnego-proxy-range-config.yml
ludus range deploy
```

### Role Requirements

None

## Usage

Apply the role with `spnego_mode: proxy` to the Debian proxy and
`spnego_mode: client` to each domain-member Windows VM. The role derives the
Kerberos realm and proxy FQDN from the primary domain controller.

The example's network policy denies direct external traffic and permits only
the proxy VM to access external networks. Remove or adjust the `network` block
when adding the role to a range with different routing requirements.

### Verify

Log on to the workstation with the configured domain user. The Startup-folder
script sets the PAC URL to
`http://proxy.ludus.domain:8080/proxy.pac`. Run
`C:\Users\Public\Desktop\verify-spnego-proxy.cmd`; the request should first
receive `407 Proxy Authentication Required` with `Proxy-Authenticate:
Negotiate`, then succeed without prompting for credentials.

On the workstation, these commands verify the PAC and Kerberos service ticket:

```powershell
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings" /v AutoConfigURL
klist get HTTP/proxy.ludus.domain
```

On the proxy, inspect `journalctl -u squid`, `/var/log/squid/cache.log`, and
`/var/log/squid/access.log`. The access log should include the authenticated
domain principal. Kerberos failures usually indicate DNS resolution, clock
skew, a duplicate SPN, or a stale keytab.

### Existing Ranges

The role can also be applied to an existing single-domain Ludus range. The range
must contain a `primary-dc` entry, and clients must resolve the proxy FQDN
through the domain DNS service.

## License

GPLv3

## Author

This role was created by Zach Stein for [Ludus](https://ludus.cloud).
