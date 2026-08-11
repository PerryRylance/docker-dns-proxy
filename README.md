# docker-dns-proxy

A shared Traefik reverse proxy for local development. It is the only container
that binds a host port (80). Every project container joins the external
`proxy` Docker network and picks up routing from labels instead of publishing
its own ports — so multiple projects can run at once without fighting over
ports, and each is reachable at `http://<name>.localhost`.

`*.localhost` hostnames resolve to `127.0.0.1` automatically in modern
browsers (RFC 6761), so no `/etc/hosts` or DNS server entries are needed.

## Usage

Start the proxy once (it stays up in the background):

```sh
docker compose up -d
```

This creates the external `proxy` network. Traefik's dashboard is at
http://traefik.localhost.

In a project's own `compose.yaml`, join the network and label the service
instead of publishing ports:

```yaml
services:
    web:
        # no `ports:` block
        networks:
            - proxy
        labels:
            - traefik.enable=true
            - traefik.docker.network=proxy
            - traefik.http.routers.myproject.rule=Host(`myproject.localhost`)
            - traefik.http.routers.myproject.entrypoints=web
            - traefik.http.services.myproject.loadbalancer.server.port=80

networks:
    proxy:
        external: true
```

If a project also needs a second entry point (e.g. a Vite dev server), give
it its own router/service pair with a distinct host, e.g.
`vite.myproject.localhost`, pointing `loadbalancer.server.port` at that
process's port.

## Non-HTTP services (databases, etc.)

Traefik only multiplexes multiple backends on one shared port for HTTP
(`Host()` header) or TLS (`HostSNI()`, requires the backend to speak TLS).
Plain TCP services like Postgres or Redis have neither, so they can't be
routed by hostname on a shared port — each one needs its own host port,
published directly on the service (not through Traefik).

Convention: don't hardcode a host port (that just moves the collision problem
to needing team-wide coordination of who uses which number). Instead publish
only the container-side port and let Docker assign a free one dynamically:

```yaml
services:
    pgsql:
        ports:
            - '5432'
```

Any number of projects can run at once this way with zero coordination and
zero collisions. Look up the assigned host port when you need to connect a
GUI client:

```sh
docker compose port pgsql 5432
```
