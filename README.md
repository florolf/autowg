# autowg

autowg is an opinionated tool for managing a zero-effort WireGuard VPN for a fleet of devices that only assumes the existence of a per-device TLS client certificate signed by some central CA.

autowg runs on the VPN concentrator and devices register with the VPN through an HTTPS endpoint, authenticated and identified by their TLS client certificate. autowg assings an IPv6 address to the device and adds a corresponding peer to the WireGuard interface.

No persistent state is kept on either end - a device is free to rejoin the VPN with a different WireGuard pubkey at any point in time, so it can just generate a fresh key on every reboot, for example. Similarly, autowg will clean out all of the peers that use a IP address prefix that it manages from the WireGuard interface upon startup. Devices are expected to re-register as needed (for example by checking if the WireGuard "last handshake" timer has expired).

An autowg VPN is expected to use IPv6 ULA-based addressing exclusively. This avoids collisions and provides a sufficiently large address space to use predictable IP address assignment algorithms that use the CN of the device certificate. Two assignment mechanisms are available:
 - `direct-bcd`, which encodes a numeric identifier from the CN directly into the IP address
 - `metans`, which uses the metans.net [hash-based AAAA record scheme](https://metans.net/hash.html)

autowg only manages peers on an existing WireGuard interface which have an IP address which matches the configured autowg address pool. This allows common configuration mechanisms like wg-quick to be used to set up the VPN and possibly statically configure peers for human users or other services.

## Usage

```
$ python server.py --help
usage: server.py [-h] --pool POOL --route ROUTE --endpoint ENDPOINT [--http-host HTTP_HOST] [--http-port HTTP_PORT] [--http-prefix HTTP_PREFIX] [--cn-pattern CN_PATTERN] [--converter {direct-bcd,metans}]
                 [--metans-template METANS_TEMPLATE]
                 interface

Wireguard autoconfig

positional arguments:
  interface             Wireguard interface to manage

options:
  -h, --help            show this help message and exit
  --pool POOL           IPv6 prefix to assign to clients (default: None)
  --route ROUTE         IPv6 route to push to clients (default: None)
  --endpoint ENDPOINT   Wireguard endpoint (host/ip, port ist inferred) (default: None)
  --http-host HTTP_HOST
                        HTTP host to listen on (default: )
  --http-port HTTP_PORT
                        HTTP port to listen on (default: 3000)
  --http-prefix HTTP_PREFIX
                        HTTP URL prefix (default: /)
  --cn-pattern CN_PATTERN
                        Regex to extract the peer name from the CN (default: ([0-9a-zA-Z]+))
  --converter {direct-bcd,metans}
                        Name-to-IP converter (default: direct-bcd)
  --metans-template METANS_TEMPLATE
                        Prefix template for metans name generation (default: None)
```

See below for a complete example on how to set up and use autowg.

## HTTP API

A client generates a WireGuard public key and POSTs it to `/v1/register`. If this request is successful, it will receive a set of configuration parameters in response:

```
endpoint=vpn.example.com:53092
pubkey=aqIUKESXUZSIXgqF9v7kSaEtwtu95jkVlFWrW0y7rUw=
route=fde3:25fb:7f6c::/48
ip=fde3:25fb:7f6c:1::1
keepalive=25
```

`endpoint` is the WireGuard endpoint. The port is taken from the configuration of the WireGuard interface itself. `pubkey` is the public key of the VPN endpoint, `route` corresponds to the `--route` argument and is the `allowed-ips` setting the client is supposed to apply on its end. `ip` is the address assigned to the client and `keepalive` is the keepalive interval the client is supposed to configure.

The client may optionally include a pre-shared key (PSK) by appending it on a second line of the POST body. If a PSK is provided, the server applies it to the WireGuard peer. This makes the VPN connection inherit the post-quantum security of the HTTPS/TLS handshake if it used an appropriate key-exchange algorithm.

See `autowg-client` for an example implementation of a client that is meant to be run in regular intervals, such as using a cron job.

Furthermore, a monitoring endpoint is available at `/v1/peers.json` when the environment variable `HTTP_AUTH` is set. The client is expected to provide an `Authorization: Basic xxx` header where `xxx` has to match the value of `HTTP_AUTH` exactly. A response will look like this:

```
{
    "23": {
        "created": 1735689600,
        "ip": "fde3:25fb:7f6c:1::1",
        "last_handshake": 1735776000,
        "pubkey": "Onuo9R6qqH7/r5bqItrl1pz09OKhyoAO24swZhxtYVk=",
        "rx_bytes": 1234567,
        "tx_bytes": 654321,
        "using_psk": true
    }
}
```

Where `23` is the name of the peer (as per the CN), `created` is the timestamp when it was (re-)registered, and the remaining values are the corresponding peer parameters similar to what `wg show` will print. `using_psk` indicates whether a pre-shared key is currently active for this peer.

## Example setup

First, randomly generate an IPv6 ULA. This example will use `fde3:25fb:7f6c::/48`. The prefix `fde3:25fb:7f6c:0::/64` will be used for statically allocated addresses such as the concentrator itself and a human operator. The prefix `fde3:25fb:7f6c:1::/64` will be managed by autowg.

Then, set up wg-quick on the concentrator:

```
$ cat /etc/wireguard/devices.conf
[Interface]
ListenPort = 51820
PrivateKey = TmV2ZXJHb25uYUdpdmVZb3VVcE5ldmVyR29ubmFMZXQ=
Address = fde3:25fb:7f6c::1/48

# VPN access for J. Random User
[Peer]
PublicKey = t6auphefW0us5R0RCd0U+o3E5E3Euj78pWdM4/chplA=
Address = fde3:25fb:7f6c::2/128
$ wg-quick up devices
```

Configure your HTTP server as a reverse proxy and enable TLS client authentication, for example using Caddy:

```
...
vpn.example.com {
	tls {
		client_auth {
			mode verify_if_given
			trusted_ca_cert_file /opt/vpn/device-ca.pem
		}
	}

	reverse_proxy /vpn/* localhost:3000 {
		header_up X-Client-Subject "{http.request.tls.client.subject}"
	}
}
...
```

Finally, start the autowg server:

```
$ python server.py \
    --http-host 127.0.0.1 --http-prefix /vpn/ \
    --route fde3:25fb:7f6c::/48 \
    --pool  fde3:25fb:7f6c:1::/64 \
    --endpoint vpn.example.com \
    devices
```

## Address assignment

In the example above, the default `direct-bcd` converter is used for address assignment. It is expected that the CN of the certificate is a plain integer in the range 0-9999 (inclusive) and this integer will be used as the peer name and BCD-encoded into the IP address, so a CN of `1234` will result in an IP address of `fde3:25fb:7f6c:1::1234`.

Sometimes, this is not flexible enough. First, the CN might be more complex than that. In such a case, the `--cn-pattern` argument can be used to supply a Python regular expression that extracts the peer name from the CN. For example, if your CNs look like `smart-toilet-123`, you can use `--cn-pattern="smart-toilet-(\d+)"`.

Encoding identifiers directly into the IP address is straightforward, but is still not flexible enough for more complex naming schemes. Furthermore, working with raw IPv6 addresses is tedious. Both problems can be solved using the `metans` converter. See https://metans.net/hash.html for a more detailed description of this mechanism.

When using the `metans` converter, the CN portion extracted by `--cn-pattern` (which defaults to the entire CN) is expanded using the format string supplied by `--metans-template` (which defaults to `%s`, i.e. a direct copy). Supplying multiple labels separated by `.` is allowed. The resulting label sequence is hashed and transformed into an IP address using the same scheme that metans.net uses and returned to the client, for example:

```
$ python server.py \
    --http-host 127.0.0.1 --http-prefix /vpn/ \
    --route fde3:25fb:7f6c::/48 \
    --pool  fde3:25fb:7f6c:1::/64 \
    --endpoint vpn.example.com \
    --converter metans \
    --cn-pattern "smart-toilet-(\d+)" --metans-template "st%s.0"
    devices
```

Will result in the client with the CN `smart-toilet-1234` receiving the IP address `fde3:25fb:7f6c:1:cb0b:5960:3f8c:99ad`.

Now suppose the zone `smartflush.example` contains this `DNAME` record:

```
devices 600 IN DNAME 0.fde325fb7f6c0001.64.hash.metans.net.
```

Then `st1234.devices.smartflush.example` or equivalently `st1234.0.fde325fb7f6c0001.64.hash.metans.net` has a matching `AAAA` record.
