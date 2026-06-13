# Prompts Ejercicio: Creando un Pipeline en GitHub Actions

> **Proyecto:** LTI — Sistema de Seguimiento de Talento  
> **Archivo de salida:** `.github/workflows/pipeline.yml`

---

## Prompt 1: Análisis del proyecto

Actúa como un DevOps Senior experto en GitHub Actions, AWS EC2, Node.js y CI/CD.

Analiza el repositorio (`README.md`, `backend/package.json`, `docker-compose.yml`) y determina:

* Cómo se ejecutan los tests del backend.
* Cómo se genera el build del backend.
* Qué versión de Node.js utiliza.
* Qué dependencias de infraestructura necesita (PostgreSQL, Prisma, etc.).
* Qué secrets harán falta para el despliegue en EC2.

Propón una estrategia de pipeline CI/CD siguiendo buenas prácticas de GitHub Actions. No generes código.

---

## Prompt 2: Validación local de tests

Actúa como un DevOps Senior experto en Node.js y CI/CD.

Antes de crear el workflow de GitHub Actions, ejecuta y valida los tests del backend en local, replicando los pasos que luego correrán en CI:

* Directorio de trabajo: `./backend`.
* Pasos: `npm ci`, `npx prisma generate`, `npm test`.
* Usa la misma versión de Node.js que se usará en el pipeline (20 LTS).
* Servicio PostgreSQL solo si los tests lo requieren; los tests actuales mockean Prisma, por lo que no debería ser necesario.
* Si algún test falla, corrígelo antes de continuar con el siguiente prompt.
* Resume el resultado: tests pasados/fallidos, tiempo de ejecución y si el entorno local está listo para automatizar en GitHub Actions.

No generes el workflow aún.

---

## Prompt 3: Tests de backend

Actúa como un DevOps Senior experto en GitHub Actions.

Crea el archivo `.github/workflows/pipeline.yml` con un job `test` para el backend en `./backend`:

* **Trigger:** `pull_request` a `main` con tipos `opened`, `synchronize` y `reopened` (cada push a una rama con PR abierta).
* Runner: `ubuntu-latest`.
* Pasos: checkout, setup Node.js (versión del proyecto), caché npm, `npm ci`, `npx prisma generate`, `npm test`.
* Servicio PostgreSQL en el job si los tests lo requieren; `DATABASE_URL` desde secrets, nunca en claro.
* Falle el pipeline si algún test falla.

Devuélveme únicamente el YAML. Sin build ni deploy aún.

> **Nota:** Una vez creado el workflow, la prueba se hará en GitHub Actions: commitea y pushea `.github/workflows/pipeline.yml`, abre un PR hacia `main` y verifica en la pestaña *Checks* que el job `test` pasa antes de continuar con el build.

---

## Prompt 4: Generación del build del backend

Partiendo del workflow anterior, añade un job `build` con `needs: test`:

* Generar el build con `npm run build` en `./backend` (tras `npm ci` y `npx prisma generate`).
* Subir como artifact `dist/`, `package.json`, `package-lock.json` y `prisma/` con `actions/upload-artifact@v4`.
* Usar caché de dependencias y variables globales (`NODE_VERSION`, `WORKING_DIR`) para evitar repetición.

Devuélveme el YAML actualizado con jobs `test` y `build`. Sin deploy aún.

> **Nota:** El job `build` no se puede probar como pipeline en local (solo opcionalmente `npm run build` en `./backend`). Para relanzar el workflow, se debe hacer commit en local y push de los cambios a la rama del PR ya abierto; el evento `synchronize` reejecutará automáticamente los jobs `test` y `build` sin necesidad de crear un PR nuevo. Verifica en *Checks* que ambos pasan antes de continuar con el deploy.

---

## Prompt 5: Despliegue del backend en EC2

Actúa como un DevOps Senior experto en AWS.

Añade un job `deploy` con `needs: build` al workflow:

* Descargar el artifact del job `build`.
* Conectar por SSH usando secrets: `EC2_HOST`, `EC2_USER`, `EC2_SSH_KEY`, `EC2_PATH`.
* Copiar el artifact a EC2 (rsync/scp) en `EC2_PATH`.
* En el servidor: `npm ci --omit=dev`, `npx prisma generate`, reiniciar con PM2 (`pm2 reload` o `pm2 start dist/index.js --name lti-backend`).
* El `.env` de producción ya debe existir en EC2; no subirlo desde el repo.
* El despliegue solo se ejecuta si tests y build han pasado.

Devuélveme el YAML completo con los tres jobs encadenados: test → build → deploy.

---

## Prompt 6: Revisión final del pipeline

Actúa como un arquitecto DevOps experto.

Revisa el workflow completo en `.github/workflows/pipeline.yml` y valida:

* Trigger correcto: push a rama con PR abierta (`pull_request` + `synchronize`).
* Cadena de jobs: test → build → deploy con `needs`.
* Seguridad y uso correcto de GitHub Secrets.
* Artifact entre build y deploy (sin rebuild innecesario en EC2).
* Posibles mejoras de rendimiento y mantenibilidad (`concurrency`, variables globales).

Genera una lista de recomendaciones y la versión final optimizada del workflow.

> **Nota final:** Tras la revisión del Prompt 6, el fichero `.github/workflows/pipeline.yml` se ajustó aplicando las recomendaciones finales: `concurrency` con cancelación de runs en progreso, `permissions: contents: read`, variable global `ARTIFACT_NAME`, `timeout-minutes` por job, `retention-days` en el artifact, `environment: production` en deploy, `--exclude '.env'` en rsync para no borrar el `.env` de EC2, y limpieza de la clave SSH al finalizar. Los jobs `test` y `build` funcionan sin secrets; el job `deploy` fallará hasta configurarlos. Si el deploy falla por falta de secrets, se puede relanzar con **Re-run failed jobs** en GitHub Actions una vez configurados. Con el **Prompt 7 (extra)**, el deploy también instala automáticamente Node.js, PM2, Docker y PostgreSQL si no están presentes en EC2.

### Secrets a configurar en GitHub

Configurar en el fork: **Settings → Secrets and variables → Actions → New repository secret**.

| Secret | Obligatorio para | Descripción | Ejemplo |
|---|---|---|---|
| `EC2_HOST` | `deploy` | IP pública o DNS de la instancia EC2 | `54.123.45.67` |
| `EC2_USER` | `deploy` | Usuario SSH de la instancia | `ec2-user` (Amazon Linux) / `ubuntu` |
| `EC2_SSH_KEY` | `deploy` | Contenido completo de la clave privada `.pem` | `-----BEGIN RSA PRIVATE KEY-----...` |
| `EC2_PATH` | `deploy` | Ruta de despliegue en el servidor | `/home/ec2-user/lti-backend` |
| `DATABASE_URL` | `deploy` (con Prompt 7 extra) | Cadena de conexión PostgreSQL; crea `.env` y levanta contenedor local si apunta a `localhost` | `postgresql://LTIdbUser:password@localhost:5432/LTIdb` |

Con el **Prompt 7 (extra)** ya no es necesario instalar manualmente Node.js, PM2, PostgreSQL ni `.env` en EC2. Solo debe existir la instancia EC2 con acceso SSH y permisos `sudo`.

---

## Prompt 7 (extra): Bootstrap automático de servicios en EC2

Actúa como un DevOps Senior experto en AWS y GitHub Actions.

Amplía el job `deploy` del workflow para que, **antes de copiar el artifact**, instale automáticamente en EC2 los servicios que falten:

* **Node.js 20** — si no está instalado o no es la versión 20.
* **PM2** — si no está instalado globalmente.
* **Docker** — si no está instalado (necesario para PostgreSQL local).
* **PostgreSQL** — contenedor Docker `lti-postgres` si `DATABASE_URL` apunta a `localhost` y el contenedor no existe.
* **`.env`** — crear en `EC2_PATH` solo si no existe, usando el secret `DATABASE_URL` (nunca subir `.env` desde el repo).
* **Directorio `uploads/`** — crear en el directorio padre de `EC2_PATH` para subida de CVs.

Requisitos adicionales:

* Detectar Amazon Linux / RHEL / Ubuntu para usar el gestor de paquetes correcto.
* Pasar `DATABASE_URL` al servidor de forma segura (por ejemplo, codificada en base64).
* Mantener `--exclude '.env'` en rsync para no sobrescribir el `.env` existente.
* Añadir `npx prisma migrate deploy` y `pm2 save` tras el despliegue.
* Aumentar `timeout-minutes` del job `deploy` (bootstrap puede tardar varios minutos).

Actualiza la tabla de secrets indicando que `DATABASE_URL` pasa a ser **obligatorio para `deploy`** cuando se usa bootstrap automático.

Devuélveme el YAML actualizado y la documentación de los secrets necesarios.

> **Nota:** Este prompt es **extra** y va más allá del ejercicio base. Automatiza el aprovisionamiento inicial de la instancia, pero la EC2 debe existir previamente con acceso SSH y el usuario debe tener permisos `sudo`. Si `DATABASE_URL` apunta a una base de datos externa (RDS, etc.), el script omite la creación del contenedor PostgreSQL local.
