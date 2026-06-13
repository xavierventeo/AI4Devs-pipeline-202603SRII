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
