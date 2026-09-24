# Configure the .env file

For more detailed information, please visit https://remna.st/docs/install/remnawave-panel


Change dir
```bash
cd /opt/remnawave
```

Get .env file
```bash
curl -o .env https://raw.githubusercontent.com/remnawave/backend/refs/heads/main/.env.sample
```

`JWT_AUTH_SECRET` and `JWT_API_TOKENS_SECRET` are used for authentication or other purposes.

Generate secret key by running the following command:

**Generating secure keys**
```bash
sed -i "s/^JWT_AUTH_SECRET=.*/JWT_AUTH_SECRET=$(openssl rand -hex 64)/" .env && sed -i "s/^JWT_API_TOKENS_SECRET=.*/JWT_API_TOKENS_SECRET=$(openssl rand -hex 64)/" .env
```


**Generating passwords**

```bash
sed -i "s/^METRICS_PASS=.*/METRICS_PASS=$(openssl rand -hex 64)/" .env && sed -i "s/^WEBHOOK_SECRET_HEADER=.*/WEBHOOK_SECRET_HEADER=$(openssl rand -hex 64)/" .env
```

Also, it is better to change the default Postgres password.

**Changing Postgres password**

```bash
pw=$(openssl rand -hex 24) && sed -i "s/^POSTGRES_PASSWORD=.*/POSTGRES_PASSWORD=$pw/" .env && sed -i "s|^\(DATABASE_URL=\"postgresql://postgres:\)[^\@]*\(@.*\)|\1$pw\2|" .env
```

# Links
- [Docker Compose example](https://raw.githubusercontent.com/remnawave/backend/refs/heads/main/docker-compose-prod.yml)
