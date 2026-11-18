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

### Requirements

- A Ludus range with a primary Active Directory domain controller.
- A Debian proxy with access to package repositories and the domain controller.
- Domain administrator credentials supplied through the Ludus configuration.
- Domain-joined Windows clients and the `ansible.windows` collection.

The role installs its Linux package dependencies. Keep credentials in protected
Ludus configuration or Ansible Vault; they are not part of this repository.

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
`http://proxy.ludus.domain:8080/proxy.pac` for the example domain. To verify
proxy authentication, run the following command from the domain user's session:

```powershell
curl.exe --silent --show-error --proxy-negotiate -U : -x http://proxy.ludus.domain:3128 --dump-header - https://example.com/ -o NUL
```

The response headers should show an initial `407 Proxy Authentication Required`
with `Proxy-Authenticate: Negotiate`, followed by a successful response without
a credential prompt. Substitute your proxy FQDN and port when using another
range.

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

### Service traffic without proxy authentication

For services such as Defender that cannot authenticate to a proxy, supply both
`spnego_unauthenticated_clients` (source IPs/CIDRs) and
`spnego_unauthenticated_domains` (Squid destination domains). Both lists default
empty. An exception requires a match in both lists; other destinations and
clients retain Kerberos authentication. A leading dot matches a domain and its
subdomains. Squid validates the configuration before installation.

Set these variables on the proxy host's `role_vars`, for example:

```yaml
spnego_unauthenticated_clients:
  - 192.0.2.25/32
spnego_unauthenticated_domains:
  - .updates.example.test
```

Replace the documentation address and domain with the required clients and
service destinations. Restrict exceptions to the necessary scope. Safe-port
checks still apply before the exception, and traffic outside both allowlists
continues to require authentication.

## License

GPLv3

## Author

This role was created by Zach Stein for [Ludus](https://ludus.cloud).
