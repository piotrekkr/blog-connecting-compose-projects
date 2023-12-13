# Connecting Two Docker Compose Projects 

Let's assume I have two Docker Compose projects:

```yaml
# project-a/compose.yml
version: '3.7'

services:
  app:
    image: alpine
    command: ['tail', '-f', '/dev/null']
```

```yaml
# project-b/compose.yml
version: '3.7'

services:
  app:
    image: nginx

  db:
    image: mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
```

I want to connect from `app` service in `project-a` to `app` service 
in `project-b`. 

## Project Networking

Since those are separate Docker Compose configurations, each of 
them will have different network configurations.

```bash
$ docker compose --project-directory=project-a ps   
NAME                IMAGE               COMMAND               SERVICE             CREATED             STATUS              PORTS
project-a-app-1     alpine              "tail -f /dev/null"   app                 13 seconds ago      Up 3 seconds


$ docker compose --project-directory=project-b ps          
NAME                IMAGE               COMMAND                  SERVICE             CREATED             STATUS              PORTS
project-b-app-1     nginx               "/docker-entrypoint.…"   app                 18 minutes ago      Up 18 minutes       80/tcp
project-b-db-1      mysql               "docker-entrypoint.s…"   db                  18 minutes ago      Up 18 minutes       3306/tcp, 33060/tcp
```

Networking for service `app` in `project-a`:

```bash
$ docker inspect project-a-app-1 | jq '.[0].NetworkSettings.Networks'
{
  "project-a_default": {
    "IPAMConfig": null,
    "Links": null,
    "Aliases": [
      "project-a-app-1",
      "app",
      "e715c34bc4ad"
    ],
    "NetworkID": "6beb2220ce42d44b570c2bb95cd613994454eb075a17c82ae28608378d221306",
    "EndpointID": "2621f1306401f4752d23002a4e4713e84244bc08e5c421b4bde82e46610d0d17",
    "Gateway": "172.19.0.1",
    "IPAddress": "172.19.0.2",
    "IPPrefixLen": 16,
    "IPv6Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "MacAddress": "02:42:ac:13:00:02",
    "DriverOpts": null
  }
}
```

Networking for service `app` in `project-b`:

```bash
$ docker inspect project-b-app-1 | jq '.[0].NetworkSettings.Networks'
{
  "project-b_default": {
    "IPAMConfig": null,
    "Links": null,
    "Aliases": [
      "project-b-app-1",
      "app",
      "c1fc59d59fb0"
    ],
    "NetworkID": "be7b70bc2897ad53021025bad3b937dd4abc159ebdc4a048f3a766e99f80ab99",
    "EndpointID": "1540af6b002f0cddb5643b80a484c4332ca2c2ad0c1b2b362c9cd8c7730e754d",
    "Gateway": "172.18.0.1",
    "IPAddress": "172.18.0.2",
    "IPPrefixLen": 16,
    "IPv6Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "MacAddress": "02:42:ac:12:00:02",
    "DriverOpts": null
  }
}
```

When containers are not in the same network, they can't communicate and `ping` 
will fail.

```bash
$ docker compose --project-directory=project-a exec app ping -c 3 172.18.0.2
PING 172.18.0.2 (172.18.0.2): 56 data bytes

--- 172.18.0.2 ping statistics ---
3 packets transmitted, 0 packets received, 100% packet loss
```

## Connecting Projects

Both services need to be in the same network, and I can do this in two ways. 

### Quick and dirty way

By default, when a project is starting, Docker Compose will create a 
[default network for the whole project to use](https://docs.docker.com/compose/networking/). 
The name of this default network will be generated based on the project name 
(if not specified `compose.yml` parent directory name is used) and 
`_default` suffix. In the case of `project-b`, it is `project-b_default`. 
Each service that did not specify any network will be attached to the default 
network. I can use this knowledge to quickly connect `project-a` `app` 
service to default network in `project-b`.

In `project-a/compose.yml` I will specify an external network with name 
matching `project-b` default network.

```yaml
# project-a/compose.yml
version: '3.7'

services:
  app:
    image: alpine
    command: ['tail', '-f', '/dev/null']
    networks:
      - project-b-network 

networks:
  project-b-network:
    external: true
    name: project-b_default # name matching project-b default network
```

To apply changes I will run `docker compose up` in `project-a`:

```bash
$ docker compose --project-directory=project-a up -d                     
[+] Running 1/1
 ✔ Container project-a-app-1  Started
```

Now `app` service is connected to the `project-b` default network.

```bash
$ docker inspect project-a-app-1 | jq '.[0].NetworkSettings.Networks'
{
  "project-b_default": {
    "IPAMConfig": null,
    "Links": null,
    "Aliases": [
      "project-a-app-1",
      "app",
      "92e7b9d1d09a"
    ],
    "NetworkID": "be7b70bc2897ad53021025bad3b937dd4abc159ebdc4a048f3a766e99f80ab99",
    "EndpointID": "539530074b89f1e2aa6b7ad724e0ca11ee6c9b6b3d3c1038b3a8fc243292eb58",
    "Gateway": "172.18.0.1",
    "IPAddress": "172.18.0.4",
    "IPPrefixLen": 16,
    "IPv6Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "MacAddress": "02:42:ac:12:00:04",
    "DriverOpts": null
  }
}
```

The `ping` command should work now.

```bash
$ docker compose --project-directory=project-a exec app ping -c 3 172.18.0.2
PING 172.18.0.2 (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.090 ms
64 bytes from 172.18.0.2: seq=1 ttl=64 time=0.081 ms
64 bytes from 172.18.0.2: seq=2 ttl=64 time=0.078 ms

--- 172.18.0.2 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.078/0.081/0.090 ms
```

Container IPs often change, so using them is not a good option. Fortunately, 
with docker instead of using IPs, I can use network aliases.

```bash
$ docker compose --project-directory=project-a exec app ping -c 3 project-b-app-1
PING project-b-app-1 (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.085 ms
64 bytes from 172.18.0.2: seq=1 ttl=64 time=0.078 ms
64 bytes from 172.18.0.2: seq=2 ttl=64 time=0.080 ms

--- project-b-app-1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.078/0.081/0.085 ms
```

#### Why is it not a good solution?

For a few reasons:

1. I need to remember to start `project-b` first, otherwise `project-a` 
   will not start
2. Project names can change, and network names will change too
3. `project-a` can also connect to other services in `project-b` default 
   network (like `db` service) which may not be desired

## Connecting using an external docker network

Most of the problems with the previous approach can be fixed by creating an 
external docker network and using it for services that need to interconnect.

```bash
$ docker network create project-a-b-network
395af51e4e71cec84f5e7c81e76d7063135048c0ae255ca8eb656853b94de335
```

I will use a new external network with services that need to interconnect.

New configuration for `project-a`:

```yaml
version: '3.7'

services:
  app:
    image: alpine
    command: ['tail', '-f', '/dev/null']
    networks:
      - project-a-b-network

networks:
  project-a-b-network:
    external: true
```

New configuration for `project-b`:

```yaml
version: '3.7'

services:
  app:
    image: nginx
    networks:
      default:
      project-a-b-network:
  db:
    image: mysql
    environment:
      MYSQL_ROOT_PASSWORD: root

networks:
  project-a-b-network:
    external: true
```

I need to apply changes:

```bash
$ docker compose --project-directory=project-a up -d                        
[+] Running 1/1
 ✔ Container project-a-app-1  Started


$ docker compose --project-directory=project-b up -d                 
[+] Running 2/2
 ✔ Container project-b-app-1  Started                                                                                                                                                                                                       0.9s 
 ✔ Container project-b-db-1   Running
```

Now, `app` service in `project-a` will be using the new external network.

```bash
$ docker inspect project-a-app-1 | jq '.[0].NetworkSettings.Networks'
{
  "project-a-b-network": {
    "IPAMConfig": null,
    "Links": null,
    "Aliases": [
      "project-a-app-1",
      "app",
      "106cd9833d6d"
    ],
    "NetworkID": "395af51e4e71cec84f5e7c81e76d7063135048c0ae255ca8eb656853b94de335",
    "EndpointID": "cbead073238355c02dc0953cf8d648f0517bbf4d03639ac3a2a7efdbd6f29a76",
    "Gateway": "172.20.0.1",
    "IPAddress": "172.20.0.2",
    "IPPrefixLen": 16,
    "IPv6Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "MacAddress": "02:42:ac:14:00:02",
    "DriverOpts": null
  }
}
```

`app` service in `project-b` is also using the new network in addition to the 
default one. It is using both because it needs to also communicate with the `db`
service.

```bash
$ docker inspect project-b-app-1 | jq '.[0].NetworkSettings.Networks'
{
  "project-a-b-network": {
    "IPAMConfig": null,
    "Links": null,
    "Aliases": [
      "project-b-app-1",
      "app",
      "eb88f1cf5a82"
    ],
    "NetworkID": "395af51e4e71cec84f5e7c81e76d7063135048c0ae255ca8eb656853b94de335",
    "EndpointID": "beafe3a2833fd69b86682a8c642b56bf202b3582f63f1fb9c015dd851c13089e",
    "Gateway": "172.20.0.1",
    "IPAddress": "172.20.0.3",
    "IPPrefixLen": 16,
    "IPv6Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "MacAddress": "02:42:ac:14:00:03",
    "DriverOpts": null
  },
  "project-b_default": {
    "IPAMConfig": null,
    "Links": null,
    "Aliases": [
      "project-b-app-1",
      "app",
      "eb88f1cf5a82"
    ],
    "NetworkID": "be7b70bc2897ad53021025bad3b937dd4abc159ebdc4a048f3a766e99f80ab99",
    "EndpointID": "34285accb2ab9633c57de26436499ac9c6ebd34e9cb29d0da42922fb930c48bd",
    "Gateway": "172.18.0.1",
    "IPAddress": "172.18.0.2",
    "IPPrefixLen": 16,
    "IPv6Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "MacAddress": "02:42:ac:12:00:02",
    "DriverOpts": null
  }
}
```

Ping is working fine from `app` service in `project-a`.

```bash
$ docker compose --project-directory=project-a exec app ping -c 3 project-b-app-1
PING project-b-app-1 (172.20.0.3): 56 data bytes
64 bytes from 172.20.0.3: seq=0 ttl=64 time=0.071 ms
64 bytes from 172.20.0.3: seq=1 ttl=64 time=0.082 ms
64 bytes from 172.20.0.3: seq=2 ttl=64 time=0.081 ms

--- project-b-app-1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.071/0.078/0.082 m
```

`db` service in `project-b` cannot be accessed from `project-a`.

```bash
$ docker compose --project-directory=project-a exec app ping -c 3 172.18.0.3     
PING 172.18.0.3 (172.18.0.3): 56 data bytes

--- 172.18.0.3 ping statistics ---
3 packets transmitted, 0 packets received, 100% packet loss
```

It is inaccessible because it is not in the same network.

```bash
$ docker inspect project-b-db-1 | jq '.[0].NetworkSettings.Networks'         
{
  "project-b_default": {
    "IPAMConfig": null,
    "Links": null,
    "Aliases": [
      "project-b-db-1",
      "db",
      "0b7ebade4bfb"
    ],
    "NetworkID": "be7b70bc2897ad53021025bad3b937dd4abc159ebdc4a048f3a766e99f80ab99",
    "EndpointID": "2369f221e48c9a55cf6cb5598aaacb3667f09a7bfadc3a19cbe802df43693f75",
    "Gateway": "172.18.0.1",
    "IPAddress": "172.18.0.3",
    "IPPrefixLen": 16,
    "IPv6Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "MacAddress": "02:42:ac:12:00:03",
    "DriverOpts": null
  }
}
```

