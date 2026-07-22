# GraphQL Actions — NestJS + Docker + GitHub Actions

API GraphQL con NestJS, empaquetada en una imagen Docker multi-stage y publicada automáticamente en Docker Hub mediante un workflow de GitHub Actions con versionado semántico.

## Contenido

- [Descripción del proyecto](#descripción-del-proyecto)
- [Stack tecnológico](#stack-tecnológico)
- [Estructura del repositorio](#estructura-del-repositorio)
- [Dockerfile (multi-stage)](#dockerfile-multi-stage)
- [GitHub Actions — CI/CD](#github-actions--cicd)
- [Versionado semántico](#versionado-semántico)
- [Secretos necesarios](#secretos-necesarios)
- [Ejecución local](#ejecución-local)
- [Ejecución con Docker](#ejecución-con-docker)
- [API GraphQL](#api-graphql)

---

## Descripción del proyecto

Aplicación NestJS que expone un endpoint GraphQL (Apollo) con:

- Módulo **HelloWorld** (queries de ejemplo)
- Módulo **Todo** (CRUD en memoria + agregaciones)

Además se configuró:

1. **Dockerfile multi-stage** para construir una imagen de producción ligera.
2. **`.dockerignore`** para no enviar a Docker archivos innecesarios.
3. **GitHub Actions workflow** que, en cada push/PR a `main`:
   - Calcula una versión semántica a partir de los commits
   - Hace login en Docker Hub
   - Construye la imagen etiquetada con esa versión
   - La publica en Docker Hub

Imagen publicada:

```text
eduhernandez/docker-graphql:<versión>
```

---

## Stack tecnológico

| Tecnología | Uso |
|---|---|
| NestJS 9 | Framework backend |
| GraphQL + Apollo | API GraphQL |
| TypeScript | Lenguaje |
| Yarn | Gestor de dependencias |
| Docker (multi-stage) | Empaquetado y despliegue |
| GitHub Actions | CI/CD |
| PaulHatch/semantic-version | Versionado automático |
| Docker Hub | Registro de imágenes |

---

## Estructura del repositorio

```text
graphql-actions/
├── .github/
│   └── workflows/
│       └── docker-image.yml   # Pipeline CI/CD
├── src/
│   ├── hello-world/           # Queries de ejemplo
│   ├── todo/                  # CRUD GraphQL de todos
│   ├── app.module.ts
│   ├── main.ts
│   └── schema.gql             # Schema generado automáticamente
├── Dockerfile                 # Build multi-stage (prod)
├── .dockerignore
├── package.json
├── yarn.lock
└── README.md
```

---

## Dockerfile (multi-stage)

El `Dockerfile` usa **4 etapas** sobre `node:19-alpine3.15` para separar dependencias, compilación y runtime:

| Stage | Nombre | Qué hace |
|---|---|---|
| 1 | `dev-deps` | Instala todas las dependencias (`yarn install --frozen-lockfile`) |
| 2 | `builder` | Copia `node_modules` + código y ejecuta `yarn build` |
| 3 | `prod-deps` | Instala solo dependencias de producción (`--prod`) |
| 4 | `prod` | Imagen final: `node_modules` de prod + `dist`, expone el puerto **3000** |

Flujo simplificado:

```text
package.json ──► dev-deps (yarn install)
                      │
                      ▼
                 builder (yarn build) ──► dist/
                      │
package.json ──► prod-deps (yarn install --prod)
                      │
                      ▼
                 prod (node dist/main.js)
```

Comando de arranque en producción:

```bash
node dist/main.js
```

### `.dockerignore`

Se excluyen del contexto de build:

```text
node_modules
.git
.github
dist
```

Así el build es más rápido y la imagen no arrastra artefactos locales ni el historial de Git.

---

## GitHub Actions — CI/CD

Archivo: [`.github/workflows/docker-image.yml`](.github/workflows/docker-image.yml)

### Disparadores

El workflow se ejecuta cuando hay:

- `push` a la rama `main`
- `pull_request` hacia `main`

### Jobs y pasos

Un solo job (`build`) en `ubuntu-latest`:

| Paso | Acción | Descripción |
|---|---|---|
| 1 | `actions/checkout@v4` | Clona el repo con `fetch-depth: 0` (historial completo, necesario para versionar) |
| 2 | `PaulHatch/semantic-version@v4.0.3` | Calcula la versión semántica según los mensajes de commit |
| 3 | Docker login | Autenticación en Docker Hub con secrets |
| 4 | Build Docker Image | `docker build -t eduhernandez/docker-graphql:$NEW_VERSION .` |
| 5 | Push Docker Image | `docker push eduhernandez/docker-graphql:$NEW_VERSION` |

Diagrama del pipeline:

```text
push / PR → main
        │
        ▼
  Checkout (historial completo)
        │
        ▼
  Semantic Version  →  NEW_VERSION
        │
        ▼
  docker login (secrets)
        │
        ▼
  docker build  →  eduhernandez/docker-graphql:$NEW_VERSION
        │
        ▼
  docker push   →  Docker Hub
```

---

## Versionado semántico

Se usa la action [PaulHatch/semantic-version@v4.0.3](https://github.com/marketplace/actions/git-semantic-version?version=v4.0.3).

Configuración actual:

```yaml
major_pattern: "major:"
minor_pattern: "feat:"
format: "${major}.${minor}.${patch}-prerelease${increment}"
```

### Cómo impactan los commits

| Prefijo en el mensaje de commit | Efecto |
|---|---|
| `major: ...` | Incrementa la versión **major** |
| `feat: ...` | Incrementa la versión **minor** |
| Otros commits | Incrementan el **patch** / contador de prerelease |

Ejemplo de tag generado:

```text
1.2.3-prerelease4
```

Esa versión se usa como tag de la imagen Docker:

```text
eduhernandez/docker-graphql:1.2.3-prerelease4
```

> Tip: usa mensajes de commit claros (`feat:`, `major:`) para controlar cómo sube la versión publicada.

---

## Secretos necesarios

En el repositorio de GitHub → **Settings → Secrets and variables → Actions**, crear:

| Secret | Descripción |
|---|---|
| `DOCKER_USER` | Usuario de Docker Hub |
| `DOCKER_PASSWORD` | Password o Access Token de Docker Hub |

El workflow los usa así:

```yaml
- name: Docker login
  env:
    DOCKER_USER: ${{ secrets.DOCKER_USER }}
    DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
  run: |
    docker login -u $DOCKER_USER -p $DOCKER_PASSWORD
```

Sin estos secrets, el login y el push fallarán.

---

## Ejecución local

### Requisitos

- Node.js (compatible con Nest 9)
- Yarn

### Instalación y arranque

```bash
yarn install

# desarrollo (watch)
yarn start:dev

# producción (tras compilar)
yarn build
yarn start:prod
```

La app queda en:

```text
http://localhost:3000/graphql
```

---

## Ejecución con Docker

### Build local

```bash
docker build -t eduhernandez/docker-graphql:local .
```

### Run

```bash
docker run --rm -p 3000:3000 eduhernandez/docker-graphql:local
```

### Pull de una imagen publicada por el CI

```bash
docker pull eduhernandez/docker-graphql:<versión>
docker run --rm -p 3000:3000 eduhernandez/docker-graphql:<versión>
```

---

## API GraphQL

Playground / landing de Apollo disponible en `/graphql`.

### Queries de ejemplo (HelloWorld)

- `hello`
- `randomNumber`
- `randomFromZeroTo(to: Int)`

### Módulo Todo

| Operación | Tipo | Descripción |
|---|---|---|
| `todos` | Query | Lista todos (filtro por status) |
| `todo(id)` | Query | Obtiene un todo por id |
| `createTodo` | Mutation | Crea un todo |
| `updateTodo` | Mutation | Actualiza un todo |
| `removeTodo` | Mutation | Elimina un todo |
| `totalTodos` / `pendingTodos` / `completedTodos` | Query | Agregaciones |
| `aggregations` | Query | Totales agrupados |

Ejemplo:

```graphql
query {
  hello
  todos {
    id
    description
    done
  }
  totalTodos
}
```

---

## Resumen de lo implementado

1. Proyecto NestJS con API GraphQL (HelloWorld + Todo).
2. `Dockerfile` multi-stage orientado a producción (`dev-deps` → `builder` → `prod-deps` → `prod`).
3. `.dockerignore` para optimizar el contexto de build.
4. Workflow de GitHub Actions en `main` para build + push a Docker Hub.
5. Versionado semántico automático según mensajes de commit (`feat:`, `major:`).
6. Publicación de imágenes como `eduhernandez/docker-graphql:<versión>`.

---

## Licencia

UNLICENSED (proyecto privado / de práctica).
