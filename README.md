# infra-local

💻 Repositório para ambiente de desenvolvimento local com serviços em Docker Compose (Keycloak, Postgres, MongoDB, Redis, SonarQube, MSSQL, MySQL e Nginx como reverse proxy HTTPS).

---

## O que este repo faz

Este projeto provê um ambiente de infra local para desenvolvimento e testes, com containers para:

- Keycloak (autenticação) + PostgreSQL para Keycloak
- SonarQube + PostgreSQL
- MongoDB
- Redis
- MySQL
- MSSQL (SQL Server)
- Um Nginx atuando como reverse-proxy com TLS usando certificados em `nginx/certs`

Os dados persistem localmente na pasta `data/` (contêineres mapeiam volumes para `./data/*`).

---

## Estrutura rápida

- `docker-compose.yml` — orquestração dos serviços.
- `nginx/nginx.conf` — configuração do Nginx (escuta 443, proxy para Keycloak).
- `nginx/certs/server.crt` e `nginx/certs/server.key` — certificados TLS usados pelo Nginx.
- `data/` — diretórios com dados persistidos dos bancos e serviços.

---

## Como rodar

1. Tenha Docker e Docker Compose instalados.
2. A partir da raiz do repo:

```bash
# Inicializar todos os serviços em background
docker compose up -d

# Ver logs (exemplo para nginx)
docker compose logs -f nginx

# Parar e remover
docker compose down
```

Observações:
- Os serviços expõem portas padrão para ambiente local (ex.: Keycloak 8080, SonarQube 9000, Postgres 5432, etc.). Veja `docker-compose.yml` para mapeamentos completos.
- Volumes montados em `data/` mantêm o estado entre reinicializações.

---

## Certificados TLS (onde e como foram gerados)

Os certificados que o Nginx usa estão em `nginx/certs/server.crt` e `nginx/certs/server.key`.

Recomendações para desenvolvimento local:

1) Comando único (gera chave RSA não criptografada + certificado auto-assinado):

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/certs/server.key -out nginx/certs/server.crt \
  -subj "/CN=192.168.100.120"
```

2) Com SAN (recomendado para navegadores e clientes modernos — inclui IP como subjectAltName):

```bash
# OpenSSL com -addext (OpenSSL >= 1.1.1/3.x)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/certs/server.key -out nginx/certs/server.crt \
  -subj "/CN=192.168.100.120" \
  -addext "subjectAltName = IP:192.168.100.120,IP:192.168.100.118,DNS:localhost"
```

Se a sua versão do OpenSSL não suportar `-addext`, crie um arquivo de configuração com `subjectAltName` e use `-config` para passar esse arquivo na criação do CSR / certificado.

Importante: para que esse fluxo funcione de forma confiável em um ambiente local, a máquina que hospeda o Nginx deve ter um IP fixo — remova o DHCP ou configure um endereço IP estático. Se o IP mudar com DHCP, você terá que regenerar os certificados (ou ajustar os SANs) sempre que o endereço mudar, o que causa falhas de validação TLS nos clientes.

Motivo: no meu computador principal eu estava ficando sem RAM ao rodar todos os serviços simultaneamente e ainda desenvolver o código. Por isso recomendo usar um IP estático na máquina que roda o Nginx (ou mover os containers para outra máquina/VM com mais memória) — assim evita refazer certificados sempre que o IP mudar e reduz a necessidade de parar/reativar serviços por falta de recursos.

3) Alternativa mais amigável para dev (mkcert):

```bash
# Instale mkcert (se ainda não tiver)
mkcert -install

# Gere o par key+crt confiável para a máquina
mkcert -key-file nginx/certs/server.key -cert-file nginx/certs/server.crt \
  192.168.100.120 192.168.100.118 localhost
```

4) Verificar o certificado atual:

```bash
openssl x509 -in nginx/certs/server.crt -noout -subject -issuer -dates -text
```