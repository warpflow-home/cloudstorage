# cloudstorage

Docker Swarm on GlusterFS with:

- Cloudflare Tunnel
- MinIO for Nextcloud primary object storage
- Nextcloud + PostgreSQL + Redis
- Immich + PostgreSQL + Redis

Important:

- Nextcloud stores file payloads in MinIO via S3-compatible primary storage.
- Immich does not use MinIO as its main asset store here. Immich stores assets on GlusterFS-backed bind mounts.

## Prerequisites

- Docker Swarm is already initialized
- `/mnt/gluster/appdata` is mounted on every node that can run the stack
- Cloudflare Tunnel token is issued from a pre-created Named Tunnel
- DNS/public hostnames are prepared in Cloudflare

## Persistent directories

Create these directories on all relevant Swarm nodes:

```bash
sudo mkdir -p \
  /mnt/gluster/appdata/cloudstorage/minio \
  /mnt/gluster/appdata/cloudstorage/nextcloud_config \
  /mnt/gluster/appdata/cloudstorage/nextcloud_db \
  /mnt/gluster/appdata/cloudstorage/immich_app \
  /mnt/gluster/appdata/cloudstorage/immich_db
```

## Environment variables

Create `.env` in the project root.

```env
CLOUDFLARE_TUNNEL_TOKEN=your-tunnel-token

MINIO_USER=minioadmin
MINIO_PASSWORD=change-this-password

NC_DB_USER=nextcloud
NC_DB_PASSWORD=change-this-password
NC_ADMIN_USER=admin
NC_ADMIN_PASSWORD=change-this-password
NC_DOMAIN=cloud.example.com
NC_DOMAIN_BARE=example.com
NC_DOMAIN_ESCAPED=cloud\.example\.com

IMMICH_VERSION=release
IMMICH_DB_USER=immich
IMMICH_DB_PASSWORD=change-this-password

TZ=Asia/Tokyo
```

Notes:

- `NC_DOMAIN_ESCAPED` is used by Collabora.
- If you do not need Nextcloud Office, you can later remove `collabora` and `nextcloud-setup` from `compose.yml`.

## Cloudflare Tunnel routing

Create public hostnames in the Cloudflare Zero Trust dashboard for the Named Tunnel tied to `CLOUDFLARE_TUNNEL_TOKEN`.

Recommended routes:

- `cloud.example.com` -> `http://nextcloud:80`
- `photos.example.com` -> `http://immich-server:2283`
- `collabora.example.com` -> `http://collabora:9980` if you use Nextcloud Office

Because `cloudflared` joins the same overlay network as the app services, these internal service names are resolvable from the tunnel container.

## Deploy on Swarm

`docker stack deploy` does not reliably consume `.env` the same way `docker compose` does, so export variables into the shell first.

```bash
set -a
source .env
set +a

docker stack deploy -c compose.yml cloudstorage
```

Check rollout status:

```bash
docker stack services cloudstorage
docker stack ps cloudstorage
```

## First boot checks

Check service logs if needed:

```bash
docker service logs cloudstorage_minio --follow
docker service logs cloudstorage_nextcloud --follow
docker service logs cloudstorage_immich-server --follow
```

Verify:

- MinIO bucket bootstrap created `nextcloud` and `immich`
- Nextcloud opens on `https://cloud.example.com`
- Immich opens on `https://photos.example.com`
- Nextcloud finishes initial install and can upload files
- Immich can upload photos and generate thumbnails

## Nextcloud storage model

This stack configures Nextcloud to use MinIO as primary object storage through these S3 settings in `compose.yml`:

- `OBJECTSTORE_S3_BUCKET=nextcloud`
- `OBJECTSTORE_S3_HOST=minio`
- `OBJECTSTORE_S3_PORT=9000`
- `OBJECTSTORE_S3_USEPATH_STYLE=true`

Warning:

- Enable this only for a fresh Nextcloud install.
- Switching an existing Nextcloud instance to primary object storage later can make existing files inaccessible.

## Immich storage model

Immich uses:

- PostgreSQL for metadata
- Redis for queue/cache functions
- `/mnt/gluster/appdata/cloudstorage/immich_app` for uploaded assets

This is intentional. Immich should keep its asset storage on the mounted filesystem in this stack.

## Useful commands

Force service reschedule:

```bash
docker service update --force cloudstorage_nextcloud
docker service update --force cloudstorage_immich-server
```

Remove the stack:

```bash
docker stack rm cloudstorage
```
