# rsync-server-docker

A small `rsyncd` + `sshd` server in Docker, based on Debian 13 (trixie-slim). You know, for moving files.

Both protocols run in the same container, so you can use either the native rsync daemon (port 873) or rsync over SSH (port 22).

Fork of [axiom-data-science/rsync-server](https://github.com/axiom-data-science/rsync-server), rebuilt regularly with current security patches.

- **Image:** `gerald77/rsync-server-docker`
- **Tags:** `latest`
- **Source:** https://github.com/geraldpublic/rsync-server
- **Ports:** `873` (rsyncd), `22` (sshd)
- **Default data path:** `/data`

## Quick start

```shell
docker run -d \
    --name rsync-server \
    -p 873:873 \
    -p 9000:22 \
    -e USERNAME=user \
    -e PASSWORD=someSecurePassword_NOT_THIS \
    -v /your/folder:/data \
    gerald77/rsync-server-docker:latest
```

**A password is required** via `PASSWORD` or `PASSWORD_FILE`, even if you only use key authentication. The container exits with an error otherwise.

## Environment variables

| Variable | Description |
| :-- | :-- |
| `USERNAME` | rsync user for the daemon. Default: `user` |
| `PASSWORD` | rsync password (also set as root password for SSH). **One of `PASSWORD` / `PASSWORD_FILE` is required.** |
| `PASSWORD_FILE` | Path to a file containing the password |
| `AUTHORIZED_KEYS` | Public SSH key(s) for the `root` user. Default: empty |
| `VOLUME` | Path served by the rsync daemon inside the container. Default: `/data` |
| `PUID` | User ID (or name) used for file transfers by the daemon. Default: `root` |
| `GUID` | Group ID (or name) used for file transfers by the daemon. Default: `root` |
| `ALLOW` | Space-separated list of allowed client networks. Default: `10.0.0.0/8 192.168.0.0/16 172.16.0.0/12 127.0.0.1/32` |
| `DENY` | Space-separated list of denied sources. Default: `*` |
| `RO` | Make the rsync volume read-only (`true`/`false`). Default: `false` |
| `CUSTOMCONFIG` | Extra line(s) for the `[volume]` section of `rsyncd.conf` (`\n\t` for a new line), e.g. `uid = root\n\tgid = root` |

## Using the rsync daemon (port 873)

The module is always called `volume`, no matter what `VOLUME` points to.

```shell
rsync -av /your/folder/ rsync://user@localhost:873/volume
```

List the module:

```shell
rsync rsync://user@localhost:873
```

## Using SSH (port 22)

Connect as `root` (not as `USERNAME`). Provide a key via bind mount or `AUTHORIZED_KEYS`; without one, SSH asks for `PASSWORD`.

```shell
docker run -d \
    --name rsync-server \
    -p 9000:22 \
    -e PASSWORD=someSecurePassword_NOT_THIS \
    -v /your/folder:/data \
    -v /my/authorized_keys:/root/.ssh/authorized_keys \
    gerald77/rsync-server-docker:latest
```

```shell
rsync -av -e "ssh -i /your/private.key -p 9000" /your/folder/ root@localhost:/data
```

With SSH you give the real destination path (e.g. `/data`), not the module name.

## More examples

Password from a file:

```shell
docker run -d -p 873:873 \
    -v /your/folder:/data \
    -v ./password-file:/etc/rsyncd/password:ro \
    -e USERNAME=admin \
    -e PASSWORD_FILE=/etc/rsyncd/password \
    gerald77/rsync-server-docker:latest
```

Custom volume path and allowed clients:

```shell
docker run -d -p 873:873 \
    -v /your/folder:/myvolume \
    -e PASSWORD=changeme \
    -e VOLUME=/myvolume \
    -e ALLOW=192.168.24.0/24 \
    gerald77/rsync-server-docker:latest
```

Read-only share:

```shell
docker run -d -p 873:873 -v /your/folder:/data -e PASSWORD=changeme -e RO=true \
    gerald77/rsync-server-docker:latest
```

## Docker Compose

```yaml
services:
  rsync:
    image: gerald77/rsync-server-docker:latest
    container_name: rsync-server
    restart: unless-stopped
    ports:
      - "873:873"
      - "9000:22"
    environment:
      USERNAME: user
      PASSWORD_FILE: /run/secrets/rsync_password
      ALLOW: "192.168.0.0/16"
    volumes:
      - /your/folder:/data
      - ./authorized_keys:/root/.ssh/authorized_keys
    secrets:
      - rsync_password

secrets:
  rsync_password:
    file: ./rsync_password.txt
```

## Notes

- Only expose ports 873/22 to networks you trust. SSH root login with password is enabled, so use a strong password or key-only access and restrict clients with `ALLOW`.
- `authorized_keys` must be owned by root; the container fixes ownership and permissions (`400`) on start.
- The image is rebuilt automatically on every push to the source repository.

## License

This image is a fork of [axiom-data-science/rsync-server](https://github.com/axiom-data-science/rsync-server), which does not publish an explicit license file. No additional license is granted by this fork beyond what the original authors permit; all rights to the original work remain with its authors. The image also contains third-party software (Debian, OpenSSH, rsync), each under its own license; see the license files inside the container (`/usr/share/doc/*/copyright`) and the [Debian licensing information](https://www.debian.org/legal/licenses/).

If you are the original author and would like something changed or removed, please open an issue at https://github.com/geraldpublic/rsync-server.

## Disclaimer

THIS IMAGE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR MAINTAINERS BE LIABLE FOR ANY CLAIM, DAMAGES, DATA LOSS OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE IMAGE OR ITS USE.

Use at your own risk. You are responsible for securing your deployment (strong passwords or keys, network restrictions, regular updates) and for backing up your data. rsync can overwrite and delete files; test with `--dry-run` first.

## Credits

Original project by Kyle Wilcox / Axiom Data Science: https://github.com/axiom-data-science/rsync-server
