# kafka-flink-platform

A production-ready stream processing platform built on top of Apache Kafka and Apache Flink, fully containerized with Docker Compose.

---

## Origin

This project is inspired by the official Confluent all-in-one stack:
[https://github.com/confluentinc/cp-all-in-one](https://github.com/confluentinc/cp-all-in-one)

The original stack ships with Confluent Control Center as the management UI, which requires an enterprise license. This project replaces it with [kafka-ui](https://github.com/provectuslabs/kafka-ui)  a free, open-source alternative that covers everything you actually need day to day.

On top of that, all internal communication between Kafka and its clients is secured with SSL, and the entire stack sits behind an Nginx reverse proxy that terminates TLS using your own certificates.

---

## What's Inside

| Service           | Image                                   | Purpose                                      |
| ----------------- | --------------------------------------- | -------------------------------------------- |
| Kafka Broker      | `confluentinc/cp-kafka`               | KRaft mode, no ZooKeeper                     |
| Schema Registry   | `confluentinc/cp-schema-registry`     | Avro / Protobuf / JSON Schema                |
| Kafka Connect     | `cnfldemos/cp-server-connect-datagen` | Connectors + Datagen source                  |
| REST Proxy        | `confluentinc/cp-kafka-rest`          | HTTP interface for Kafka                     |
| Flink JobManager  | `cnfldemos/flink-kafka`               | Job coordination + Web UI                    |
| Flink TaskManager | `cnfldemos/flink-kafka`               | Job execution                                |
| Flink SQL Client  | `cnfldemos/flink-sql-client-kafka`    | Interactive SQL shell                        |
| kafka-ui          | `provectuslabs/kafka-ui`              | Management UI (free)                         |
| Nginx             | `nginx:alpine`                        | TLS termination + reverse proxy + basic auth |

ksqlDB is included in the codebase but disabled by default. Uncomment the relevant blocks in `docker-compose.yml` to enable it.

---

## Security

**Internal:** Kafka and all its clients communicate over SSL using self-signed JKS certificates. The `generate-kafka-ssl` service handles certificate generation automatically on first run. Certificates are stored in `./certs/` and reused on subsequent starts.

**External:** Nginx sits in front of every service. It terminates TLS using your real certificates and enforces HTTP Basic Authentication on all subdomains. Nothing is exposed directly to the internet except Nginx on ports 80 and 443.

---

## Prerequisites

* Docker and Docker Compose
* A domain with DNS access
* A valid TLS certificate (`fullchain.pem` and `privkey.pem`) — Let's Encrypt works perfectly

---

## Directory Structure

```
.
├── docker-compose.yml
├── .env
├── cert/
│   ├── fullchain.pem
│   └── privkey.pem
├── certs/                  # auto-generated on first run
└── nginx/
    ├── nginx.conf
    └── auth/
        └── .htpasswd
```

---

## Setup

**1. Clone and configure**

```bash
git clone https://github.com/enavid/kafka-flink-platform.git
cd kafka-flink-platform
cp .env.example .env
```

Edit `.env` and set your passwords and cluster ID at minimum.

**2. Place your TLS certificates**

```bash
mkdir -p cert
cp /path/to/fullchain.pem ./cert/
cp /path/to/privkey.pem ./cert/
```

**3. Generate Basic Auth credentials**

```bash
mkdir -p ./nginx/auth
sudo apt install apache2-utils -y
htpasswd -c ./nginx/auth/.htpasswd your-username
```

You will be prompted to set a password. To add more users, run the same command without `-c`.

**4. Start the stack**

```bash
docker compose -f docker-compose up -d
```

Kafka SSL certificates are generated automatically on first start. Subsequent starts skip this step.

---

## DNS Configuration

Create DNS A records for each subdomain pointing to your server's public IP. This needs to be done in your DNS provider or CDN dashboard (Cloudflare, AWS Route 53, etc.).

| Subdomain                          | Routes to             |
| ---------------------------------- | --------------------- |
| `kafka-ui.yourdomain.com`        | kafka-ui:8080         |
| `flink.yourdomain.com`           | flink-jobmanager:9081 |
| `schema-registry.yourdomain.com` | schema-registry:8081  |
| `kafka-connect.yourdomain.com`   | kafka-connect:8083    |
| `rest-proxy.yourdomain.com`      | rest-proxy:8082       |

If you are using Cloudflare, set the proxy status to DNS only (gray cloud) for these records. Cloudflare's proxy does not play well with arbitrary TCP ports, and you want your own certificate to terminate TLS, not Cloudflare's.

---

## Adding a New Service Behind Nginx

Open `nginx/nginx.conf` and add two blocks:

```nginx
# 1. upstream
upstream my-service { server my-service:PORT; }

# 2. server block
server {
    listen      443 ssl;
    server_name my-service.yourdomain.com;
    location /  { proxy_pass http://my-service; }
}
```

Then add the corresponding DNS A record and restart Nginx:

```bash
docker compose -f docker-compose.yml restart nginx
```

---

## Disabling Basic Auth for a Specific Subdomain

If a service has its own authentication (like kafka-ui with LOGIN_FORM), you can exempt it from Basic Auth by adding `auth_basic off` to its server block:

```nginx
server {
    listen      443 ssl;
    server_name kafka-ui.yourdomain.com;
    auth_basic  off;
    location /  { proxy_pass http://kafka-ui; }
}
```

---

## Flink SQL Client

To open an interactive Flink SQL session:

```bash
docker exec -it flink-sql-client ./bin/sql-client.sh
```

---

## Enabling ksqlDB

ksqlDB is commented out in `docker-compose.yml`. To enable it, uncomment the `ksqldb-server` and `ksqldb-cli` service blocks and restart the stack.
