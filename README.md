# Vault-Agent

Tested on: Debian 10, Debian 11

This sets up Hashicorp's vault in agent mode on a machine. For now, the agent is configured to use auto-auth with
kerberos and this role then supports automatic provisioning of TLS certificates.

## Configuration
|Var|Default value|Description|
|---|-------------|-----------|
|vault_server|`https://vault.example.org`|URL of the vault server to connect to|
|vault_have_certs|`False`|Whether client certificates are required to connect to vault, see below|
|vault_cert_dirs|`[]`|List of directories where certificates will be put in. This makes sure that they exist and that vault has access|
|vault_certs|`[]`|List of certificates to request, see below|
|vault_autoauth_krb_realm|`EXAMPLE.ORG`|Kerberos realm to use for autoauth|
|vault_bug_3827_workaround|`[]`|List of CA names to perform the CRL bug workaround for|
|vault_bug_3827_connect_addr|`https://{{ ansible_fqdn }}:8443`|URL to use when connecting to vault for performing the CRL workaround|

## Features

### Client certificate support
When `vault_have_certs`, client certificates are expected to be provided in `/etc/vault-agent.d/client.pem`,
`/etc/vault-agent.d/client.key` as well as the matching CA in `/etc/vault-agent.d/ca.pem`.

### Certificate requests
`vault_certs` contains a list of certificates to request as per the following example:
```
  - name: prom-write
    type: client
    pem: /etc/grafana-agent/client.pem
    key: /etc/grafana-agent/client.key
    ca: /etc/grafana-agent/client-ca.pem
    restart: systemctl restart grafana-agent
    cn: "{{ ansible_fqdn }}"
```
Available types are `client`, `server` and `dual` if the CA was set up with the matching terraform
role - its just the name of the PKI secret's backend role. Apart from `pem`, `key` and `ca`, `pem_with_key` and
`crl` exist - everything in PEM format for now.
`restart` is executed whenever the crl or the certificate changes - the command is executed as root, so you can restart
services here.

### Vault Bug 3827 workaround
Vault does currently not generate a new CRL if a new CRL wouldn't contain new information - so if no certificates are
revoked, your CRL will eventually "expire". Tools that use openssl (e.g. HAProxy) will then simply mark all requests as
invalid - so this workaround will rotate all listed CA's CRLs around midnight. You'll need to make sure that the node
has a policy assigned that allows it to do `pki/<name>/crl/rotate`.
Rolled into this workaround is also a script to call `pki/<name>/tidy` - so make sure you include that in your permissions aswell.

## Dependencies
Due to the nature of the agent, this role requires all filesystems you want to put certificates on to support
extended ACLs!
Additionally, it'll use sudo to elevate privileges for the restart command.
