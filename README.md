# baby's first matrix server

this guide assumes you're using debian. because I am.
it also uses docker so you'll need that. https://docs.docker.com/engine/install/debian/
if you don't, I'm sure you can make it happen. godspeed.

Note: never copy the `db/` directory between different `server_name`s. each tuwunel setup gets its own `server_name` which is set once and can never be changed. the `db/` is `server_name` specific and is heavily recommended to be backed up periodically as it stores your entire server's history.

### why are we using the things we're using?
tuwunel is an open source alternative to element's synapse server.
I like open source.

smtp is fully optional if you don't mind doing manual password recovery for your users. I added it because I don't want to do that.
matrixRTC is fully optional if you don't mind your ElementX users not being able to audio/video call.
caddy is fully optional if you don't mind your connection being HTTP

### can I also use nginx?
yes.

### will you add MAS to support a registration process via ElementX?
no.
MAS offers essentially no meaningful benefits as of time of writing and simply adds another required domain, another service clogging your cpu, another db to backup. all so that your users can use ElementX to register.
the issues can be entirely circumvented by using app.element.io or the desktop client. or by you manually creating users using the admin command
`!admin users create-user <USERNAME>`
this command may change so doublecheck it with `!admin users --help`. at the time of writing, it is the working command


## getting started

1. Hostnames
while you're of course free to simply use localhost for the entire setup, I would highly recommend creating a custom hostname for multiple reasons since it's self documenting. regardless of an `ss -tlnp` or config files, no guesswork is required.

something like: `echo "127.0.0.2 matrix-host" | sudo tee -a /etc/hosts`

2. DNS setups
you will need two `A` entries. one for tuwunel and one for the MatrixRTC. meaning if you don't care about MatrixRTC you can skip the second entry as you may have guessed.

something like

```
TYPE: A
NAME: matrix
TTL: 600
IP: <YOURSERVER>
```
and
```
TYPE: A
NAME: rtc.matrix
TTL: 600
IP: <YOURSERVER>
```

create these records wherever your domain's nameservers actually point (check with `nslookup -type=NS yourdomain`); a panel you don't delegate to does nothing.

3. Firewall setup
we actually need quite a few firewall rules so strap in:

`ssh ingress TCP destination port: 22 ACCEPT`

`HTTP (cert + redirect) ingress TCP destination port: 80 ACCEPT`

`HTTPS ingress TCP destination port: 443 ACCEPT`

`DNS replies ingress UDP source port: 53 ACCEPT`

`NTP replies ingress UDP source port: 123 ACCEPT`

potentially optional. not optional if IPv6 is at play. omit at your own risk:

`ICMP    ingress  protocol ICMP    ACCEPT    # ping, path-MTU discovery (IPv4)`

`ICMPv6  ingress  protocol ICMPv6  ACCEPT    # REQUIRED if the host has any IPv6`

this sets us up for matrix/tuwunel + caddy TLS

if you want MatrixRTC support you will need to add the following:

`ingress TCP destination port: 7881 ACCEPT`

`ingress UDP destination port: 50100-50200 ACCEPT`

50100-50200 is a fairly arbitrary value. you might need more than 100 ports, or way fewer, depending on the size of your community.

make sure you have no SMTP egress deny rules that would affect us if you plan to use SMTP. outbound SMTP 587/465 need to be accepted.

4. Config files

in total we want to setup 4 config files. of those 4, 3 are necessary and 1 is optional but helpful

### tuwunel.toml
create a root directory as well as a database directory for your tuwunel setup. this is our matrix dir.

`mkdir -p ~/tuwunel/db` (you can name it whatever you want of course)

use your preferred editor

`nano ~/tuwunel/tuwunel.toml`

`vim ~/tuwunel/tuwunel.toml`

```toml
[global]
server_name = "YOUR.MATRIX.DOMAIN" # this cannot be changed later
database_path = "/var/lib/tuwunel"
address = "0.0.0.0"
port = 8008
allow_registration = true # set this to false if you want to only be able to create users manually
registration_token = "YOURSTRONGSECRET HERE" # create some randomised registration token like with openssl rand -hex 32
allow_federation = false # whether or not you want your server to be connected to the matrix federation
new_user_displayname_suffix = "" # tuwunel adds hearts as a username suffix. cute but we don't need it. tuwunel may remove this in the future so check if it's still applicable. tuwunel *should* ignore unknown config items but future versions may not. if your server isn't reachable, refer to the testing section of this readme

[global.smtp]
connection_uri = "smtps://<your>%40<email>.<domain>:SMTP-<submission-token>@smtp.<yourmailprovider>.com:<port>"
sender = "YOUR@EMAIL.DOMAIN"

[global.well_known]
client = "https://YOUR.MATRIX.DOMAIN"
server = "YOUR.MATRIX.DOMAIN:443"
livekit_url = "https://YOUR.RTC.DOMAIN"
```

make sure to replace the fillers with the domains we set up in step 2.

### Caddy
first we need to install caddy

`sudo apt update && sudo apt install -y caddy`

then we want to create our caddyfile in root:

```bash
echo 'your.matrix.domain {
    reverse_proxy matrix-host:8008
}
your.rtc.domain {
    @jwt path /get_token* /sfu/get* /healthz* /sfu_webhook*
      handle @jwt {
          reverse_proxy matrix-host:8081
      }
      handle {
          reverse_proxy matrix-host:7880
      }
}' | sudo tee /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

remember that we're referencing the `matrix-host` we set up in step 1.

### lk-jwt

this one is optional but handy. it prevents us from having to write them into the `docker run` command any time we need to remove the container during testing.

```bash
# ~/rtc/lk-jwt.env
LIVEKIT_JWT_BIND=:8081
LIVEKIT_URL=wss://your.rtc.domain
LIVEKIT_KEY=matrixrtc
LIVEKIT_SECRET=REPLACE ME WITH SOMETHING LIKE openssl rand -hex 32 OUTPUT
LIVEKIT_FULL_ACCESS_HOMESERVERS=your.matrix.domain
```

### Livekit

`mkdir -p ~/rtc`

```bash
#~/rtc/livekit.yaml
port: 7880
bind_addresses:
  - "0.0.0.0"
rtc:
  tcp_port: 7881
  port_range_start: 50100
  port_range_end: 50200
  use_external_ip: false
  node_ip: "YOUR SERVER PUBLIC IP"
  enable_loopback_candidate: false
room:
  auto_create: false
keys:
  matrixrtc: LONG_RANDOM_SECRET openssl rand -hex 32 OUTPUT
webhook:
  api_key: matrixrtc
  urls:
    - https://your.rtc.domain/sfu_webhook
```

5. first run and test!
assuming all goes well, we're actually almost done. all we're now doing is starting the three docker containers (tuwunel, livekit and lk-jwt) as well as Caddy for TLS.

```bash
docker run -d --name tuwunel \
  -p matrix-host:8008:8008 \
  -v ~/tuwunel/db:/var/lib/tuwunel \
  -v ~/tuwunel/tuwunel.toml:/etc/tuwunel.toml:ro \
  -e TUWUNEL_CONFIG=/etc/tuwunel.toml \
  --restart unless-stopped \
  jevolk/tuwunel
```

`sudo systemctl reload caddy`

```bash
docker run -d --name livekit --network host --restart unless-stopped \
  -v ~/rtc/livekit.yaml:/etc/livekit.yaml:ro \
  livekit/livekit-server:latest --config /etc/livekit.yaml
```

```bash
docker run -d --name lk-jwt --restart unless-stopped \
  -p matrix-host:8081:8081 \
  --env-file ~/rtc/lk-jwt.env \
  ghcr.io/element-hq/lk-jwt-service:latest
```

the choice is yours how we proceed, either by looking into logs immediately, or trying to see if the server is reachable by your preferred matrix client.
for this guide we use element so we navigate to `https://app.element.io` and attempt to register using the matrix domain instead of the `matrix.org` default homeserver by element.
you will be asked for a registration token according to our `tuwunel.toml` so refer back to this config file and look for the token you set in `registration_token`.
don't share this token, especially not on unencrypted channels. it would completely void your security layers since this token will allow anyone to register until you change it in the config file and restart the tuwunel docker container.

if this errors in the application we have multiple options.
first we check the docker logs `docker logs -f tuwunel`
the last line should be `Listening on ["tcp:0.0.0.0:8008"]`

if the output is correct but you still can't connect, it's possible we're running into an issue with the resolver so we try curling:
`curl https://your.matrix.domain/_matrix/client/versions` (do this on your machine, not the server)

if it's not reachable, check your device's internet settings and add the DNS resolvers `1.1.1.1` and `8.8.8.8`. if you're connected via wifi, restart your wifi and run the curl again.

important notes:
- I specifically cited `https://app.element.io` because ElementX will not allow you to register a user due to a missing MAS setup.
don't @ me if you tried with ElementX because you didn't read carefully.
- lk-jwt caches your homeserver's address from its boot time. If you ever change well_known in tuwunel, run `docker restart lk-jwt` or calls will fail against the old configuration.
- any change in the tuwunel.toml also needs a `docker restart tuwunel`

if your curl looks good, you can now login via ElementX. you should do this because we will now be testing the MatrixRTC as this is only needed for ElementX.
navigate to your `Admin Room` within matrix and create a new test user with `!admin users create-user test` and login via `https://app.element.io`
go back to your server console and run `docker logs -f lk-jwt` as we need to see where and how it fails, in case it fails.
with the logs running, attempt to call your test user from your ElementX device.

the above configs are tested and in working condition so it *should* just work. if it doesn't, read the error log carefully. it is likely a mistake in a config file, your firewall settings or the DNS entries.

I can't write an answer for every error you may run into here but a helpful curl would be `curl -i https://your.rtc.domain/healthz` with the expected output of something like this:
```
HTTP/2 200 
alt-svc: h3=":443"; ma=2592000
date: Fri, 16 Oct 2026 09:46:35 GMT
via: 1.1 Caddy
content-length: 0
```
basically we want to see a HTTP 200, obviously.

testing TL;DR:

- `docker logs -f CONTAINER` to get the logs of any specific container that may be acting up
- `journalctl -u caddy` to get the caddy logs in case caddy is acting up 

I'm not adding the config files to the repo because I believe in your ability to copy paste.
good luck!
