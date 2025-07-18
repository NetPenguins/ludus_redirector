# netpenguins.ludus_redirector

An Ansible role for Ludus to deploy and configure an Apache-based redirector, supporting custom rewrite rules. This role is designed for use in Ludus ranges and can be easily integrated with other roles and custom configurations.

- [Usage](#usage)
- [Role Variables](#role-variables)
- [Network & Routing](#how-routing-and-network-restrictions-work)



## Features
- Installs and configures Apache2 with SSL support
- Supports custom rewrite rules (dynamic via Ansible variables)
- Deploys a custom `index.html` to the web root


## Installation in Ludus

#### Install via Ansible Galaxy:

```sh
ludus ansible role add netpenguins.ludus_redirector
```

#### Or clone directly:

```sh
git clone https://github.com/netpenguins/netpenguins.ludus_redirector.git /opt/netpenguins.ludus_redirector
ludus ansible role add /opt/netpenguins.ludus_redirector
```

### Customizing index.html or Apache2 Config

If you want to **customize the `index.html` page** or the **Apache2 configuration template** (`redirector.conf.j2`), you should use the [**git clone method**](#or-clone-directly) above. After cloning:

- Place your custom `index.html` in:
  - `netpenguins.ludus_redirector/files/index.html`
- Place your custom Apache2 config template in:
  - `netpenguins.ludus_redirector/templates/redirector.conf.j2`

Edit these files as needed, then run your playbook. Your custom files will be deployed to the target server.



## Usage

### Example Ludus Configuration

```yaml
# yaml-language-server: $schema=https://docs.ludus.cloud/schemas/range-config.json
ludus:
  - vm_name: REDIRECTOR
    hostname: redirector
    template: debian-12-x64-server-template
    vlan: 100
    ip_last_octet: 10
    ram_gb: 2
    cpus: 2
    linux: true
    dns_rewrites:
      - redir.ludus
      - '*.redir.ludus'
    roles:
      - netpenguins.ludus_redirector
    role_vars:
      redirector_host: 'c2.redir.ludus'
      redirector_target_host: "c2.ludus"  # Change to your C2
      kali_vlan_subnet: "10.{{ range_second_octet }}.200.0/24" # Change to the subnet of your Kali VLAN 
      rewrite_rules:
        - 'RewriteRule ^/l33t$ https://10.100.200.5/getsome [P,L]'
  - vm_name: KALI
    hostname: kali
    template: kali-x64-desktop-template
    dns_rewrites:
      - "c2.ludus"
      - "*.c2.ludus"
    vlan: 200
    ip_last_octet: 5
    ram_gb: 4
    cpus: 2
    linux: true
network:
  rules:
    - name: Allow Kali to Redirector
      vlan_src: 200
      vlan_dst: 100
      protocol: "all"
      ports: "all"
      action: ACCEPT
    - name: Allow Redirector to Kali 
      vlan_src: 100
      vlan_dst: 200
      protocol: "all"
      ports: "all"
      action: ACCEPT
    - name: Allow Target to Redirector HTTP
      vlan_src: 10
      vlan_dst: 100
      protocol: tcp
      ports: 80
      action: ACCEPT
    - name: Allow Target to Redirector HTTPS
      vlan_src: 10
      vlan_dst: 100
      protocol: tcp
      ports: 443
      action: ACCEPT
  inter_vlan_default: DROP # Drop all by default 
  external_default: ACCEPT  # Allow outbound internet if needed 
```

## Role Variables

Available variables (see `defaults/main.yml`):

| Variable                   | Description                                      | Default/Example                                 |
|----------------------------|--------------------------------------------------|-------------------------------------------------|
| `redirector_target_host`   | Target host for proxy/rewrite                    | `c2.ludus`                                      |
| `redirector_host`          | The ServerName for Apache (used in config)       | `c2.redir.ludus`                                |
| `ssl_certificate_file`     | Path to SSL certificate file                     | `/etc/ssl/certs/ssl-cert-snakeoil.pem`          |
| `ssl_certificate_key_file` | Path to SSL certificate key                      | `/etc/ssl/private/ssl-cert-snakeoil.key`        |
| `ssl_certificate_chain_file`| Path to SSL certificate chain file (optional)    | `/etc/ssl/certs/chain.pem` (uncomment to use)   |
| `rewrite_rules`            | List of Apache RewriteRule strings (optional)    | See example in `defaults/main.yml`              |


## How Routing and Network Restrictions Work

This role configures Apache2 to act as a redirector/proxy, using rewrite rules and network restrictions:

- **Rewrite Rules:**
  - You can define custom rewrite rules via the `rewrite_rules` variable. These are rendered into the Apache config and control which routes are proxied or redirected to your target host.
  - Example: Only requests to `/getsome` are proxied to your C2 server, while all other requests serve local content.
  - The `range_second_octet` variable allows you to dynamically set the second octet of your IP addresses, making your configuration portable across different ranges.

- **Subnet Restrictions:**
  - The `kali_vlan_subnet` variable is used to restrict which subnets can access the redirector. Only clients from the specified subnet will be allowed by Apache's `Require ip` directive.
  - Example: If `kali_vlan_subnet` is set to `10.{{ range_second_octet }}.200.0/24`, only hosts in that subnet can access the redirector.

- **Typical Flow:**
  - A user from an allowed subnet (e.g., Kali) accesses the redirector.
  - If the request matches a rewrite rule (e.g., `/getsome`), it is proxied to the backend target (e.g., your C2 server).
  - All other requests are served from the local web root (e.g., `index.html`).



## License

GPLv3



## Author

This role was created by NetPenguins for Ludus.
