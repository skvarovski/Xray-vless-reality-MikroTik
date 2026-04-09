# Xray-vless-reality-MikroTik

## Обновление бинарников

### Источники
- **Xray-core**: https://github.com/XTLS/Xray-core/releases
- **tun2socks**: https://github.com/xjasonlyu/tun2socks/releases

### Скачивание и перепаковка zip → 7z

Релизы идут в `.zip`, в репо хранятся как `.7z`. Из каждого zip извлечь только бинарник, запаковать в `.7z` с тем же именем файла внутри (`start.sh` ожидает `xray` и `tun2socks`).

#### Xray-core: маппинг файлов

| GitHub zip | Репо .7z | Архитектура |
|---|---|---|
| `Xray-linux-64.zip` | `Containers/xray-core/Xray-linux-64.7z` | amd64 |
| `Xray-linux-arm64-v8a.zip` | `Containers/xray-core/Xray-linux-arm64-v8a.7z` | arm64 |
| `Xray-linux-arm32-v7a.zip` | `Containers/xray-core/Xray-linux-arm32-v7a.7z` | armv7 |

#### tun2socks: маппинг файлов

| GitHub zip | Репо .7z | Архитектура |
|---|---|---|
| `tun2socks-linux-amd64.zip` | `Containers/tun2socks/tun2socks-linux-amd64.7z` | amd64 |
| `tun2socks-linux-arm64.zip` | `Containers/tun2socks/tun2socks-linux-arm64.7z` | arm64 |
| `tun2socks-linux-armv7.zip` | `Containers/tun2socks/tun2socks-linux-armv7.7z` | armv7 |

#### Команды перепаковки (пример для Xray-core arm64)
```bash
unzip Xray-linux-arm64-v8a.zip xray
7z a -mx=9 Xray-linux-arm64-v8a.7z xray
cp Xray-linux-arm64-v8a.7z Containers/xray-core/
```

## Сборка и экспорт контейнера для RouterOS (arm64)

### Требования
- Docker с поддержкой buildx
- QEMU для кросс-платформенной сборки (если хост не arm64)

### 1. Включение QEMU (если хост amd64)
```bash
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```

### 2. Сборка образа
```bash
cd Containers
docker buildx build -f Dockerfile_arm64 --no-cache --progress=plain \
  --platform linux/arm64/v8 --provenance=false \
  --output=type=docker,compression=gzip \
  --tag xray-mikrotik-arm64:latest .
```

### 3. Экспорт для RouterOS (flatten в один слой)
RouterOS не поддерживает многослойные образы с zstd/gzip-сжатием слоёв. Необходимо сплющить образ в один слой через `docker export/import`:

```bash
CID=$(docker create xray-mikrotik-arm64:latest)
docker export $CID | docker import \
  --change 'ENTRYPOINT ["sh","-c"]' \
  --change 'CMD ["/bin/bash /opt/start.sh && /sbin/init"]' \
  - xray-mikrotik-arm64:latest
docker rm $CID
docker save xray-mikrotik-arm64:latest > xray-mikrotik-arm64.tar
```

Итоговый файл `xray-mikrotik-arm64.tar` (~22 MB) загружается в RouterOS.
