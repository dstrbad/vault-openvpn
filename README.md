![license](https://badges.fyi/github/license/Luzifer/vault-openvpn)
[![Go Report Card](https://goreportcard.com/badge/github.com/Luzifer/vault-openvpn)](https://goreportcard.com/report/github.com/Luzifer/vault-openvpn)

# Luzifer / vault-openvpn

`vault-openvpn` is a small wrapper utility to manage [OpenVPN](https://openvpn.net/) configuration combined with a [Vault PKI](https://www.vaultproject.io/docs/secrets/pki/index.html). It enables administrators with Vault access to create client / server configurations with only one command. No more hazzle to manage that easyrsa PKI, just some few commands to set up a Vault PKI and you're done.

## Setting up Vault for this

The Vault setup follows the Quick Start from the Vault documentation and is personalized for me so you need to adapt it to your domain if you want to follow it:

- Mount PKI for my domain  
`vault mount -path=luzifer_io pki`
- Enable long TTLs  
`vault mount-tune -max-lease-ttl=87600h luzifer_io`
- Generate a root certificate  
`vault write pki/root/generate/internal common_name=luzifer.io ttl=87600h`
- Set CA / CRL URLs  
`vault write luzifer_io/config/urls issuing_certificates=${VAULT_ADDR}/v1/luzifer_io/ca crl_distribution_points=${VAULT_ADDR}/v1/luzifer_io/crl`
- Set a rule for OpenVPN certificates  
`vault write luzifer_io/roles/openvpn allowed_domains="openvpn.luzifer.io" allow_subdomains="true" max_ttl="8760h" ttl="8760h" allow_ip_sans=false allow_localhost=false`

That's all you need to do to set up a whole PKI for your OpenVPN.

## Configuration of the tool

You can pass all configurations through commandline-parameters. To see the available options and their defaults use the `vault-openvpn --help` flag.

Additionally most of the parameters are also supported to be set using a configuration file to be stored in `~/.config/vault-openvpn.yaml`. To use that file you need to specify the arguments to the flags together with the flag name:

```yaml
---
log-level: debug
template-path: /path/to/templates
```

The flags not supported to be set through that file are `vault-addr`, `vault-token` and `version`. First two for security reasons, last because it does not make sense.

## Issuing configurations

You need to create a folder containing two files: `client.conf` and `server.conf`. Those two are templates to use for generating the configuration file used by `vault-openvpn`. Inside those files paste this block which will get replaced by the certificates:

```
<ca>
{{ .CertAuthority }}
</ca>

<cert>
{{ .Certificate }}
</cert>

<key>
{{ .PrivateKey }}
</key>
```

The configurations generated by this tool will not need multiple files but include the certificates inside the configuration. This makes it far more easy to pass them to your users. No unzip, no questions where to put the files, mostly the OpenVPN clients will know how to handle something called `my-vpn.conf`.

After you've set up your folder (you also could use one of the example configurations in the [`example` folder](https://github.com/Luzifer/vault-openvpn/tree/master/example) of this repository) you can issue your servers configuration:

```bash
# vault-openvpn --auto-revoke --pki-mountpoint luzifer_io server edda.openvpn.luzifer.io
server 10.231.0.0 255.255.255.0
route 10.231.0.0 255.255.255.0

[...]
```

And also you can generate client configurations:

```bash
# vault-openvpn --auto-revoke --pki-mountpoint luzifer_io client workwork01.openvpn.luzifer.io
remote myserver.com 1194 udp

[...]
```

In case someone needs to get removed from your OpenVPN there is also a revoke:

```bash
# vault-openvpn --auto-revoke --pki-mountpoint luzifer_io revoke baduser.openvpn.luzifer.io
[...]
2016/07/25 15:06:58 Found certificate 33:e1:0c:85:36:a5:c2:6b:05:85:f5:aa:9f:3b:f3:3a:a2:e0:ae:b0 with CN baduser.openvpn.luzifer.io
2016/07/25 15:06:58 Revoked certificate 33:e1:0c:85:36:a5:c2:6b:05:85:f5:aa:9f:3b:f3:3a:a2:e0:ae:b0
[...]
```

To have revokes being executed by OpenVPN you need to periodically update the CRL file OpenVPN reads. For my solution see the `living-example` in the `example` folder.
