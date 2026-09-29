# Immich

This container allows you to deploy [Immich](https://immich.app/), a self-hosted photo and video backup solution. It deploys a PostgreSQL database, a Valkey cache and a machine learning container as sidecars.

- Immich server: `nextcloud-aio-immich-server`
- Immich machine learning: `nextcloud-aio-immich-machine-learning`
- Valkey cache: `nextcloud-aio-immich-valkey`
- PostgreSQL database: `nextcloud-aio-immich-database`

PostgreSQL, Valkey and the machine learning service are not exposed and are only reachable through the Docker network.

### Why

Immich complements Nextcloud with fast mobile backup, a polished photo timeline, search and face recognition. Running it inside AIO lets AIO back up its data together with the rest of the instance.

### Requirements

- The [Caddy](https://github.com/nextcloud/all-in-one/tree/main/community-containers/caddy) community container is required to expose Immich over HTTPS at `photo.$NC_DOMAIN`. Immich does not support being served on a sub-path, so it needs its own subdomain.
- Set a DNS record for `photo.$NC_DOMAIN` pointing to your AIO server (a CNAME to `$NC_DOMAIN` or A/AAAA records). Keep it DNS only; do not enable a CDN or proxy in front of it.
- Set both DNS records before starting the containers so that Caddy can issue a certificate.

### Initial setup

1. Open `https://photo.$NC_DOMAIN` and create the first administrator account.
2. Open the admin settings (Administration > Settings) and set:
   - **Server > External Domain** to `https://photo.$NC_DOMAIN`.
   - **Machine Learning > URLs** to `http://nextcloud-aio-immich-machine-learning:3003`.

> [!Important]
> Immich v3 no longer supports an environment variable for the machine learning URL. Its default value is `http://immich-machine-learning:3003`, which does not resolve in AIO because AIO container names are prefixed with `nextcloud-aio-`. Until AIO supports network aliases, the URL must be adjusted once in the Immich admin settings as described above. Without this change, smart search and face detection jobs fail.

### Caddy

If the currently released Caddy container does not yet configure Immich automatically, add a custom config via the import mechanism described in the [Caddy readme](https://github.com/nextcloud/all-in-one/tree/main/community-containers/caddy#custom-configuration):

```
https://photo.your-nc-domain.com:443 {
    reverse_proxy nextcloud-aio-immich-server:2283

    tls {
        issuer acme {
            disable_http_challenge
        }
    }
}
```

### Volumes

| Volume | Mount | Backed up |
| --- | --- | --- |
| `nextcloud_aio_immich_data` | Immich library (`/data`), including uploaded photos and videos | yes |
| `nextcloud_aio_immich_database` | PostgreSQL data directory | yes |
| `nextcloud_aio_immich_model_cache` | Machine learning model cache (`/cache`) | no |

The model cache is intentionally not backed up because it only contains downloadable models. Valkey keeps no data of its own and matches the persistence semantics of the upstream Immich deployment (no volume).

### Direct access

The Immich server port `2283` is published on `APACHE_IP_BINDING`, so Immich is also reachable directly on the LAN via `http://<host-lan-ip>:2283`. Do not forward this port from the internet and restrict it to trusted networks with a firewall if needed.

### Backups

Both `nextcloud_aio_immich_data` and `nextcloud_aio_immich_database` are included in the AIO backup. Immich should be stopped by AIO during a backup, which keeps the database consistent.

> [!Note]
> Additionally enable Immich's own database dumps (Administration > Settings > Backup) if you want an application-level dump in addition to the AIO backup. They are written to the Immich library (`/data/backups`) and are therefore included in the AIO backup.

### Upgrade

The images are pinned to a fixed Immich release. To upgrade, update the `image_tag` of `nextcloud-aio-immich-server` and `nextcloud-aio-immich-machine-learning` to the same new release, keep `nextcloud-aio-immich-database` on the required PostgreSQL/VectorChord image, then update the containers from the AIO interface.

> [!Caution]
> Always read the [Immich release notes](https://github.com/immich-app/immich/releases) before upgrading. Database migrations are not reversible, so take an AIO backup first. The server and the machine learning container must always run the same version.

### Known limitations

- The machine learning URL has to be set once in the Immich admin settings (see above). AIO does not yet support additional Docker network aliases.
- Images are pinned by tag only. The AIO schema cannot express the `@sha256:` digests that upstream Immich uses for Valkey and PostgreSQL, so the database and cache images follow the versioned tags of the Immich release instead of the exact digests.
- Only CPU based machine learning is configured. GPU acceleration (NVIDIA, Intel/OpenVINO, AMD/ROCm) is not part of this container.

### Repository

- https://github.com/immich-app/immich
- https://immich.app/
- https://docs.immich.app/

### Maintainer

https://github.com/mdeix
