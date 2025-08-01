# Server Installation & Configuration

The proxy server software mita needs to run on Linux. We provide both debian and RPM installers for installing mita on Debian / Ubuntu and Fedora / CentOS / Red Hat Enterprise Linux series distributions.

## Use installation script

Follow the installation script and complete the server installation and configuration.

```sh
curl -fSsLO https://raw.githubusercontent.com/enfein/mieru/refs/heads/main/tools/setup.py
chmod +x setup.py
sudo python3 setup.py
```

Or you can manually install and configure proxy server using the steps below.

## Download mita installation package

```sh
# Debian / Ubuntu - X86_64
curl -LSO https://github.com/enfein/mieru/releases/download/v3.17.1/mita_3.17.1_amd64.deb

# Debian / Ubuntu - ARM 64
curl -LSO https://github.com/enfein/mieru/releases/download/v3.17.1/mita_3.17.1_arm64.deb

# RedHat / CentOS / Rocky Linux - X86_64
curl -LSO https://github.com/enfein/mieru/releases/download/v3.17.1/mita-3.17.1-1.x86_64.rpm

# RedHat / CentOS / Rocky Linux - ARM 64
curl -LSO https://github.com/enfein/mieru/releases/download/v3.17.1/mita-3.17.1-1.aarch64.rpm
```

## Install mita package

```sh
# Debian / Ubuntu - X86_64
sudo dpkg -i mita_3.17.1_amd64.deb

# Debian / Ubuntu - ARM 64
sudo dpkg -i mita_3.17.1_arm64.deb

# RedHat / CentOS / Rocky Linux - X86_64
sudo rpm -Uvh --force mita-3.17.1-1.x86_64.rpm

# RedHat / CentOS / Rocky Linux - ARM 64
sudo rpm -Uvh --force mita-3.17.1-1.aarch64.rpm
```

Those instructions can also be used to upgrade the version of mita software package.

## Grant permissions, logout and login again to make the change effective

```sh
sudo usermod -a -G mita $USER

# logout
exit
```

## Reconnect the server via SSH, check mita daemon status

```sh
systemctl status mita
```

If the output contains `active (running)`, it means that the mita daemon is already running. Normally, mita will start automatically after the server is booted.

## Check mita working status

```sh
mita status
```

If the installation is just completed, the output will be `mita server status is "IDLE"`, indicating that mita has not yet listening to requests from the mieru client.

## Modify proxy server settings

The mieru proxy supports two different transport protocols, TCP and UDP. To understand the differences between the protocols, please read [mieru proxy protocols](./protocol.md).

Users should call

```sh
mita apply config <FILE>
```

to modify the proxy server settings. `<FILE>` is a JSON formatted configuration file. This configuration file does not need to specify the full proxy server settings. When you run command `mita apply config <FILE>`, the contents of the file will be merged into any existing proxy server settings.

Below is an example of the server configuration file.

```js
{
    "portBindings": [
        {
            "portRange": "2012-2022",
            "protocol": "TCP"
        },
        {
            "port": 2027,
            "protocol": "TCP"
        }
    ],
    "users": [
        {
            "name": "ducaiguozei",
            "password": "xijinping"
        },
        {
            "name": "meiyougongchandang",
            "password": "caiyouxinzhongguo"
        }
    ],
    "loggingLevel": "INFO",
    "mtu": 1400
}
```

1. The `portBindings` -> `port` property is the TCP or UDP port number that mita listens on, specify a value from 1025 to 65535. If you want to listen to a range of consecutive port numbers, you can also use the `portRange` property instead. **Please make sure that the firewall allows communication using these ports.**
2. The `portBindings` -> `protocol` property can be set to `TCP` or `UDP`.
3. Fill in the `users` -> `name` property with the user name.
4. Fill in the `users` -> `password` property with the user's password.
5. [Optional] The `mtu` property is the maximum transport layer payload size when using the UDP proxy protocol. The default value is 1400. The minimum value is 1280.

In addition to this, mita can listen to several different ports. We recommend using multiple ports in both server and client configurations.

You can also create multiple users if you want to share the proxy for others to use.

Assuming that on the server, the configuration file name is `server_config.json`, call the command `mita apply config server_config.json` to write the configuration after the file is modified.

If there is an error in the configuration, mita will print the problem that occurred. Follow the prompts to modify the configuration file and re-run the `mita apply config <FILE>` command to write the configuration.

After that, invoke command

```sh
mita describe config
```

to check the current proxy settings.

## Start proxy service

Use command

```sh
mita start
```

to start proxy service. At this point, mita will start listening to the port number specified in the settings.

Then, use command

```sh
mita status
```

to check working status. If `mita server status is "RUNNING"` is returned here, it means that the proxy service is running and can process client requests.

If you want to stop the proxy service, use command

```sh
mita stop
```

Note that each time you change the settings with `mita apply config <FILE>`, you need to restart the service with `mita stop` and `mita start` for the new settings to take effect. An exception is, if you only change `users` or `loggingLevel` settings, you may run `mita reload` to load the new settings, which will not disturb active connections between server and client.

After starting the proxy service, proceed to [Client Installation & Configuration](./client-install.md).

## Advanced Settings

### BBR Congestion Control Algorithm

[BBR](https://en.wikipedia.org/wiki/TCP_congestion_control#TCP_BBR) is a congestion control algorithm that does not rely on packet loss. Under poor network conditions, network transmission using BBR is faster than traditional algorithms.

mieru's UDP transmission protocol already uses the BBR algorithm.

Run the following script to use BBR algorithm on TCP transmission protocol on Linux.

```sh
curl -fSsLO https://raw.githubusercontent.com/enfein/mieru/refs/heads/main/tools/enable_tcp_bbr.py
chmod +x enable_tcp_bbr.py
sudo python3 enable_tcp_bbr.py
```

That script can be used on both server side and client side.

### Configuring Outbound Proxy

The outbound proxy feature allows mieru to work with other proxy tools to form a proxy chain. An example of the network topology of a proxy chain is shown in the diagram below:

```
mieru client -> GFW -> mita server -> cloudflare proxy client -> cloudflare CDN -> target website
```

Through proxy chain, the target website sees the IP address of the cloudflare CDN, not the address of the mita server.

Below is an example to configure a proxy chain.

```js
{
    "egress": {
        "proxies": [
            {
                "name": "cloudflare",
                "protocol": "SOCKS5_PROXY_PROTOCOL",
                "host": "127.0.0.1",
                "port": 4000,
                "socks5Authentication": {
                    "user": "shilishanlu",
                    "password": "buhuanjian"
                }
            }
        ],
        "rules": [
            {
                "ipRanges": ["8.8.4.4/32", "8.8.8.8/32"],
                "action": "REJECT"
            },
            {
                "domainNames": ["chatgpt.com", "grok.com"],
                "action": "PROXY",
                "proxyNames": ["cloudflare"]
            },
            {
                "ipRanges": ["*"],
                "domainNames": ["*"],
                "action": "DIRECT"
            }
        ]
    }
}
```

1. In the `egress` -> `proxies` property, list the information of outbound proxy servers. The current version only supports socks5 outbound, so the value of `protocol` must be set to `SOCKS5_PROXY_PROTOCOL`. If the outbound proxy server requires socks5 username and password authentication, please fill in the `socks5Authentication` property. Otherwise, please remove the `socks5Authentication` property.
2. In the `egress` -> `rules` property, list outbound rules. Outbound actions include `DIRECT`, `PROXY` and `REJECT`. `proxyNames` must be set if `PROXY` action is used. `proxyNames` needs to point to proxies that exist in `egress` -> `proxies` property.

If you want to turn off the outbound proxy feature, simply set the `egress` property to an empty value `{}`.

Note that proxy chain is different from nested proxy. An example of the network topology of a nested proxy is shown in the diagram below:

```
Tor browser -> mieru client -> GFW -> mita server -> Tor network -> target website
```

For information on how to configure nested proxy on a Tor browser, please refer to the [Security Guide](./security.md).

### DNS Policy in IPv4 / IPv6 Dual-Stack Network

When a proxy client requests a target website using a domain name instead of an IP address, the proxy server needs to initiate a DNS request. If the proxy server is in an IPv4 / IPv6 dual-stack network, you can adjust the DNS policy using the following configuration:

```js
{
    "dns": {
        "dualStack": "USE_FIRST_IP"
    }
}
```

The `dns` -> `dualStack` attribute supports the following values:

1. `USE_FIRST_IP`: Always use the first IP address returned by the DNS server. This is the default policy.
2. `PREFER_IPv4`: Prefer to use the first IPv4 address returned by the DNS server. If there is no IPv4 address, use the first IPv6 address.
3. `PREFER_IPv6`: Prefer to use the first IPv6 address returned by the DNS server. If there is no IPv6 address, use the first IPv4 address.
4. `ONLY_IPv4`: Force to use the first IPv4 address returned by the DNS server. If there is no IPv4 address, the connection fails.
5. `ONLY_IPv6`: Force to use the first IPv6 address returned by the DNS server. If there is no IPv6 address, the connection fails.

### Allow Users to Access Internal Network

By default, proxy server only allows users to send proxy requests to the Internet.

If you want to allow users to access private IP addresses (e.g., `192.168.1.100`) through the proxy server, set the `users` -> `allowPrivateIP` attribute.

If you want to allow users to access the server's local machine (e.g., `127.0.0.1`) through the proxy server, add the `users` -> `allowLoopbackIP` attribute.

```js
{
    "users": [
        {
            "name": "ducaiguozei",
            "password": "xijinping",
            "allowPrivateIP": true,
            "allowLoopbackIP": true
        },
        {
            "name": "meiyougongchandang",
            "password": "caiyouxinzhongguo"
        }
    ]
}
```

### Limiting User Traffic

We can use the `users` -> `quotas` property to limit the amount of traffic a user is allowed to use. For example, if you want user "ducaiguozei" to use no more than 1 GB of traffic within 1 day, and no more than 10 GB within 30 days, you can apply the following settings.

```js
{
    "users": [
        {
            "name": "ducaiguozei",
            "password": "xijinping",
            "quotas": [
                {
                    "days": 1,
                    "megabytes": 1024
                },
                {
                    "days": 30,
                    "megabytes": 10240
                }
            ]
        },
        {
            "name": "meiyougongchandang",
            "password": "caiyouxinzhongguo"
        }
    ]
}
```

## [Optional] Install NTP network time synchronization service

The client and proxy server software calculate the key based on the user name, password and system time. The server can decrypt and respond to the client's request only if the client and server have the same key. This requires that the system time of the client and the server must be in sync.

To ensure that the server system time is accurate, we recommend that users enable or install the NTP network time service.

In Linux, if system time synchronization is controlled by `systemd-timesyncd` service, you can modify configuration file `/etc/systemd/timesyncd.conf` to the following.

```
[Time]
NTP=time.google.com
```

Otherwise, install NTP with the following command.

```sh
# Debian / Ubuntu
sudo apt-get install ntp

# RedHat / CentOS / Rocky Linux
sudo dnf install ntp
```
