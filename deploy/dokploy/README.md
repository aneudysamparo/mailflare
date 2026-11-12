# Mailflare en Dokploy (fork de Postor)

Servicio aparte. **Nunca se mezcla con el código de Postor** (Mailflare es AGPL-3.0: este fork es público para cumplir la licencia).

- Repo: `aneudysamparo/mailflare`, rama `postor`. `main` queda igual que el original para traer actualizaciones.
- Correo entrante: Cloudflare Email Routing → Worker relay → `/api/inbound`. No hace falta el puerto 25.
- Datos: SQLite y archivos en el volumen `mailflare-data` (`/data`).

## 1. Crear el servicio en Dokploy

1. **Create Service → Compose** (tipo *Docker Compose*).
2. Provider: GitHub → `aneudysamparo/mailflare`, branch `postor`.
3. Compose Path: `./deploy/dokploy/docker-compose.yml`.
4. **Environment:** copia `deploy/dokploy/env.example` y llena los valores. Genera el secreto con:
   ```bash
   openssl rand -hex 32
   ```
5. **Domains:** agrega el dominio (p. ej. `correo.postor.do`), servicio `mailflare`, puerto `3000` y HTTPS con Let's Encrypt. Activa WebSockets si Dokploy lo ofrece (los usa `/api/realtime`).
6. **Deploy.** El primer build tarda varios minutos (Next.js).
7. Abre `https://correo.postor.do/setup` y crea la cuenta de administrador **enseguida** (la primera persona que entre a `/setup` queda como admin).

Si el `.env` no se carga, pasa las mismas variables en la sección *Environment* del servicio. Dokploy las inyecta igual.

## 2. Correo entrante (Cloudflare)

El dominio tiene que estar en Cloudflare.

```bash
cd deploy/cloudflare-email-relay
npm install
npx wrangler secret put MAILFLARE_URL            # https://correo.postor.do
npx wrangler secret put INBOUND_WEBHOOK_SECRET   # el mismo valor del paso 1.4
npm run deploy
```

En Cloudflare → Email Routing del dominio, envía el *catch-all* (o las direcciones que quieras) al Worker `mailflare-email-relay`.

## 3. Correo saliente

Elige uno:
- **Cloudflare Email Sending:** `CF_ACCOUNT_ID` + `CF_TOKEN` (permisos en el README principal).
- **Resend** (3,000 correos gratis al mes): `RESEND_API_KEY` o configúralo en la página del dominio.
- **SMTP** cualquiera: `SMTP_URL=smtps://usuario:clave@host:465`.

En la app, la página del dominio muestra los registros SPF, DKIM y DMARC que faltan.

## 4. Operación

- **Respaldo:** diario a las 02:00 UTC dentro del volumen. Respalda también el volumen completo desde Dokploy (*Volume Backups*).
- **Actualizar desde el original:**
  ```bash
  git fetch upstream && git switch main && git merge --ff-only upstream/main && git push
  git switch postor && git merge main && git push
  ```
  Luego, *Redeploy* en Dokploy. Las migraciones corren solas al arrancar.
- **Logs:** pestaña *Logs* del servicio en Dokploy.
