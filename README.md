# Пошаговая настройка — тестовый вариант (без SSL, локальная сеть)

Разобьём на **6 этапов**. Каждый этап — отдельный блок работы с проверкой результата.

---

## Этап 1 — Подготовка сервера

### 1.1 Проверяем Docker

```bash
docker --version
docker compose version
```

Если не установлен:
```bash
curl -fsSL https://get.docker.com | sh
```

### 1.2 Узнаём IP сервера в локальной сети

```bash
ip -4 addr show | grep inet
```

Запомни IP, например `192.168.1.100` — он будет везде вместо домена.

### 1.3 Создаём структуру папок

```bash
mkdir -p /opt/vpn-stack/{headscale/{config,data},headplane,rustdesk/data,tailscale/state}
cd /opt/vpn-stack
```

Проверяем:
```bash
find /opt/vpn-stack -type d
```

Ожидаемый вывод:
```
/opt/vpn-stack
/opt/vpn-stack/headscale
/opt/vpn-stack/headscale/config
/opt/vpn-stack/headscale/data
/opt/vpn-stack/headplane
/opt/vpn-stack/rustdesk
/opt/vpn-stack/rustdesk/data
/opt/vpn-stack/tailscale
/opt/vpn-stack/tailscale/state
```

✅ **Этап 1 готов** — переходи к Этапу 2.

---

## Этап 2 — Запускаем Headscale

### 2.1 Создаём конфиг Headscale

```bash
nano /opt/vpn-stack/headscale/config/config.yaml
```

Вставляем (**замени `192.168.1.100` на свой IP**):

```yaml
server_url: http://192.168.1.100:8080

listen_addr: 0.0.0.0:8080
metrics_listen_addr: 127.0.0.1:9090
grpc_listen_addr: 0.0.0.0:50443
grpc_allow_insecure: true

noise:
  private_key_path: /var/lib/headscale/noise_private.key

prefixes:
  v4: 100.64.0.0/10
  v6: fd7a:115c:a1e0::/48
  allocation: sequential

derp:
  server:
    enabled: false
  urls:
    - https://controlplane.tailscale.com/derpmap/default
  auto_update_enabled: true
  update_frequency: 24h

disable_check_updates: true
ephemeral_node_inactivity_timeout: 30m

database:
  type: sqlite
  sqlite:
    path: /var/lib/headscale/db.sqlite

log:
  level: info

policy:
  mode: file
  path: /etc/headscale/acl.yaml

dns:
  magic_dns: true
  base_domain: vpn.internal
  nameservers:
    global:
      - 1.1.1.1
      - 8.8.8.8
```

### 2.2 Создаём ACL политику

```bash
nano /opt/vpn-stack/headscale/config/acl.yaml
```

```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["*"],
      "dst": ["*:*"]
    }
  ]
}
```

### 2.3 Создаём docker-compose.yml — только Headscale

```bash
nano /opt/vpn-stack/docker-compose.yml
```

```yaml
services:

  headscale:
    image: headscale/headscale:latest
    restart: unless-stopped
    command: serve
    ports:
      - "8080:8080"
    volumes:
      - ./headscale/config:/etc/headscale
      - ./headscale/data:/var/lib/headscale
    networks:
      - internal

networks:
  internal:
    driver: bridge
```

### 2.4 Запускаем

```bash
cd /opt/vpn-stack
docker compose up -d headscale
```

### 2.5 Проверяем что работает

```bash
# Смотрим логи — ошибок не должно быть
docker compose logs headscale

# Проверяем HTTP ответ
curl http://192.168.1.100:8080/health
```

Ожидаемый ответ: `{"status":"pass"}`

✅ **Этап 2 готов** — переходи к Этапу 3.

---

## Этап 3 — Настраиваем Headscale, получаем ключи

### 3.1 Создаём пользователя (организацию)

```bash
docker compose exec headscale headscale users create myorg
```

Проверяем (запомни ID пользователя из вывода — понадобится на шаге 3.3):
```bash
docker compose exec headscale headscale users list
```

### 3.2 Создаём API ключ для Headplane

```bash
docker compose exec headscale headscale apikeys create
```

**Скопируй и сохрани вывод** — это API ключ. Показывается только один раз.

Создаём файл с переменными:
```bash
nano /opt/vpn-stack/.env
```

```env
SERVER_IP=192.168.1.100
HEADSCALE_API_KEY=вставь_ключ_сюда
TS_AUTHKEY=заполним_на_следующем_шаге
```

### 3.3 Создаём pre-auth ключ для Tailscale sidecar

> Используй числовой ID пользователя из вывода шага 3.1 (обычно `1`).

```bash
docker compose exec headscale headscale preauthkeys create --user 1 --reusable --expiration 999d
```

**Скопируй вывод** → вставь в `.env` как `TS_AUTHKEY`.

### 3.4 Проверяем ключи

```bash
docker compose exec headscale headscale apikeys list
docker compose exec headscale headscale preauthkeys list
```

✅ **Этап 3 готов** — переходи к Этапу 4.

---

## Этап 4 — Добавляем Headplane

> Новые версии Headplane не читают переменные окружения — требуется собственный конфиг-файл.

### 4.1 Создаём конфиг Headplane

```bash
nano /opt/vpn-stack/headplane/config.yaml
```

Вставляем (**замени значения на свои из `.env`**):

```yaml
headscale:
  url: http://192.168.1.100:8080
  api_key: "вставь_HEADSCALE_API_KEY"

server:
  host: 0.0.0.0
  port: 3000
  cookie_secure: false
  cookie_secret: "вставь_32_символа"

integration:
  docker:
    enabled: true
    container_name: vpn-stack-headscale-1
```

Для генерации `cookie_secret`:
```bash
openssl rand -hex 16
```

### 4.2 Обновляем docker-compose.yml

```bash
nano /opt/vpn-stack/docker-compose.yml
```

```yaml
services:

  headscale:
    image: headscale/headscale:latest
    restart: unless-stopped
    command: serve
    ports:
      - "8080:8080"
    volumes:
      - ./headscale/config:/etc/headscale
      - ./headscale/data:/var/lib/headscale
    networks:
      - internal

  headplane:
    image: ghcr.io/tale/headplane:latest
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - ./headplane/config.yaml:/etc/headplane/config.yaml:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      - headscale
    networks:
      - internal

networks:
  internal:
    driver: bridge
```

### 4.3 Запускаем

```bash
cd /opt/vpn-stack
docker compose up -d headplane
```

### 4.4 Проверяем

```bash
docker compose logs headplane --tail=20
```

Ожидаемый вывод в конце:
```
[server] INFO: Headplane started on 0.0.0.0:3000
```

Открываем в браузере: `http://192.168.1.100:3000/admin` — должна появиться страница Headplane.

✅ **Этап 4 готов** — переходи к Этапу 5.

---

## Этап 5 — Запускаем Tailscale sidecar

> Этот контейнер входит в VPN-сеть Headscale и «отдаёт» свой сетевой namespace контейнерам RustDesk.

### 5.1 Включаем TUN на хосте???????????????????????????????????????????????????????????????????????????????????????????????????????????????

```bash
modprobe tun
ls /dev/net/tun   # должен существовать
```

Чтобы модуль загружался после перезагрузки:
```bash
echo "tun" >> /etc/modules
```

### 5.2 Обновляем docker-compose.yml

```bash
nano /opt/vpn-stack/docker-compose.yml
```

```yaml
services:

  headscale:
    image: headscale/headscale:latest
    restart: unless-stopped
    command: serve
    ports:
      - "8080:8080"
    volumes:
      - ./headscale/config:/etc/headscale
      - ./headscale/data:/var/lib/headscale
    networks:
      - internal

  headplane:
    image: ghcr.io/tale/headplane:latest
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - ./headplane/config.yaml:/etc/headplane/config.yaml:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      - headscale
    networks:
      - internal

  tailscale:
    image: tailscale/tailscale:latest
    restart: unless-stopped
    hostname: rustdesk-node
    environment:
      TS_AUTHKEY: ${TS_AUTHKEY}
      TS_EXTRA_ARGS: "--login-server=http://192.168.1.100:8080"
      TS_STATE_DIR: /var/lib/tailscale
      TS_USERSPACE: "false"
      TS_ACCEPT_DNS: "false"
    volumes:
      - ./tailscale/state:/var/lib/tailscale
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
    networks:
      - internal

networks:
  internal:
    driver: bridge
```

### 5.3 Запускаем

```bash
cd /opt/vpn-stack
docker compose up -d tailscale
```

### 5.4 Проверяем регистрацию ноды

```bash
docker compose logs tailscale

# Нода должна появиться в headscale
docker compose exec headscale headscale nodes list
```

Ожидаемый вывод — строчка с `rustdesk-node` и VPN IP типа `100.64.0.1`.

**Запиши этот VPN IP** — он понадобится для настройки клиентов RustDesk.

✅ **Этап 5 готов** — переходи к Этапу 6.

---

## Этап 6 — Запускаем RustDesk

### 6.1 Финальный docker-compose.yml

```bash
nano /opt/vpn-stack/docker-compose.yml
```

```yaml
services:

  headscale:
    image: headscale/headscale:latest
    restart: unless-stopped
    command: serve
    ports:
      - "8080:8080"
    volumes:
      - ./headscale/config:/etc/headscale
      - ./headscale/data:/var/lib/headscale
    networks:
      - internal

  headplane:
    image: ghcr.io/tale/headplane:latest
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - ./headplane/config.yaml:/etc/headplane/config.yaml:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      - headscale
    networks:
      - internal

  tailscale:
    image: tailscale/tailscale:latest
    restart: unless-stopped
    hostname: rustdesk-node
    environment:
      TS_AUTHKEY: ${TS_AUTHKEY}
      TS_EXTRA_ARGS: "--login-server=http://192.168.1.100:8080"
      TS_STATE_DIR: /var/lib/tailscale
      TS_USERSPACE: "false"
      TS_ACCEPT_DNS: "false"
    volumes:
      - ./tailscale/state:/var/lib/tailscale
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
    networks:
      - internal

  rustdesk-hbbs:
    image: rustdesk/rustdesk-server:latest
    restart: unless-stopped
    command: hbbs
    network_mode: "service:tailscale"
    volumes:
      - ./rustdesk/data:/root
    depends_on:
      - tailscale

  rustdesk-hbbr:
    image: rustdesk/rustdesk-server:latest
    restart: unless-stopped
    command: hbbr
    network_mode: "service:tailscale"
    volumes:
      - ./rustdesk/data:/root
    depends_on:
      - tailscale

networks:
  internal:
    driver: bridge
```

### 6.2 Запускаем

```bash
cd /opt/vpn-stack
docker compose up -d rustdesk-hbbs rustdesk-hbbr
```

### 6.3 Проверяем

```bash
docker compose logs rustdesk-hbbs
docker compose logs rustdesk-hbbr

# Все контейнеры должны быть Up
docker compose ps
```

### 6.4 Получаем публичный ключ RustDesk

```bash
cat /opt/vpn-stack/rustdesk/data/*.pub
```

**Сохрани этот ключ** — вводится в настройках каждого клиента RustDesk.

---

## Итоговая проверка всего стека

```bash
docker compose ps
```
8:29 20.03.2026
Должны быть **Up**: `headscale`, `headplane`, `tailscale`, `rustdesk-hbbs`, `rustdesk-hbbr`

```bash
# Список всех нод в VPN
docker compose exec headscale headscale nodes list

# Headplane UI
# Открой в браузере: http://192.168.1.100:3000
```

---

## Что дальше — настройка клиентов

После того как все 6 этапов пройдены, напиши мне — разберём подключение клиентов (Tailscale + RustDesk) пошагово для Windows/Linux/macOS.



