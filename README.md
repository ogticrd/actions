# ogticrd/actions

Repositorio central de GitHub Actions reutilizables para los proyectos de OGTIC.

## Sobre este repositorio

Aquí se mantienen las composite actions que cualquier repositorio de la organización puede utilizar en sus pipelines de CI/CD. La idea es tener un solo lugar donde vivan las operaciones comunes de despliegue, en vez de duplicar configuraciones entre proyectos.

Actualmente el foco está en despliegues a **GCP Cloud Run**, pero el repositorio se irá expandiendo con actions para otros proveedores y servicios según las necesidades lo requieran.

## Estructura

```
actions/
├── docker/
│   └── build/              Build y push de imágenes Docker (GAR, GHCR, Docker Hub)
└── gcp/
    ├── auth/                Autenticación a GCP (JSON key o Workload Identity)
    └── cloudrun/
        ├── deploy/          Despliegue a Cloud Run
        ├── iam/             Control de acceso público/privado
        └── cleanup/         Eliminación de servicios
```

## Inicio rápido

```yaml
# .github/workflows/deploy.yml en tu repositorio
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4

      - name: Build y push de imagen
        id: build
        uses: ogticrd/actions/actions/docker/build@main
        with:
          image-name: ${{ vars.GOOGLE_ARTIFACT_REGISTRY }}/${{ vars.APP_NAME }}
          registry-username: _json_key
          registry-password: ${{ secrets.GAR_JSON_KEY }}

      - name: Autenticación a GCP
        uses: ogticrd/actions/actions/gcp/auth@main
        with:
          project-id: ${{ vars.GOOGLE_PROJECT_ID }}
          credentials-json: ${{ secrets.GAR_JSON_KEY }}

      - name: Deploy a Cloud Run
        uses: ogticrd/actions/actions/gcp/cloudrun/deploy@main
        with:
          image: ${{ steps.build.outputs.primary-image }}
          service: ${{ vars.APP_NAME }}
          region: ${{ vars.GOOGLE_CLOUD_REGION }}

      - name: Permitir acceso público
        uses: ogticrd/actions/actions/gcp/cloudrun/iam@main
        with:
          service: ${{ vars.APP_NAME }}
          region: ${{ vars.GOOGLE_CLOUD_REGION }}
```

---

## Actions disponibles

### `docker/build`

Construye y sube imágenes Docker a cualquier registry compatible con OCI.

```yaml
- uses: ogticrd/actions/actions/docker/build@main
  with:
    registry: us-docker.pkg.dev           # default
    registry-username: _json_key
    registry-password: ${{ secrets.GAR_JSON_KEY }}
    image-name: project-id/repo/app
    target: runner                        # opcional, para multi-stage
    build-args: |                         # opcional
      NEXT_PUBLIC_API_URL=${{ vars.API_URL }}
    environment: production               # opcional, agrega tag
    cache-strategy: registry              # registry (default), gha, none
```

| Output | Descripción |
|---|---|
| `image` | Todos los tags generados |
| `primary-image` | Tag con SHA corto (para usar en deploy) |
| `digest` | Digest de la imagen |
| `image-tag` | SHA corto (7 caracteres) |

---

### `gcp/auth`

Autenticación a Google Cloud. Soporta JSON key y Workload Identity Federation.

```yaml
# JSON key (método actual)
- uses: ogticrd/actions/actions/gcp/auth@main
  with:
    project-id: ${{ vars.GOOGLE_PROJECT_ID }}
    credentials-json: ${{ secrets.GAR_JSON_KEY }}
```

```yaml
# Workload Identity Federation
- uses: ogticrd/actions/actions/gcp/auth@main
  with:
    project-id: ${{ vars.GOOGLE_PROJECT_ID }}
    workload-identity-provider: projects/123/locations/global/workloadIdentityPools/pool/providers/provider
    service-account: sa@project.iam.gserviceaccount.com
```

---

### `gcp/cloudrun/deploy`

Despliega una imagen a Cloud Run con escalabilidad y variables de entorno configurables.

```yaml
- uses: ogticrd/actions/actions/gcp/cloudrun/deploy@main
  with:
    image: us-docker.pkg.dev/project/repo/app:abc1234
    service: my-service
    region: us-east1
    project-id: my-project
    env-vars: |
      NODE_ENV=production
      DATABASE_URL=${{ secrets.DATABASE_URL }}
    min-instances: "1"
    max-instances: "100"
    cpu: "2"
    memory: 1Gi
    vpc-connector: projects/my-project/locations/us-east1/connectors/us-east1
```

| Output | Descripción |
|---|---|
| `url` | URL del servicio desplegado |
| `revision` | Nombre de la revisión |

---

### `gcp/cloudrun/iam`

Controla si el servicio es público o privado.

```yaml
# Público (default)
- uses: ogticrd/actions/actions/gcp/cloudrun/iam@main
  with:
    service: my-service
    region: us-east1

# Privado
- uses: ogticrd/actions/actions/gcp/cloudrun/iam@main
  with:
    service: my-service
    region: us-east1
    allow-unauthenticated: "false"
```

---

### `gcp/cloudrun/cleanup`

Elimina un servicio de Cloud Run. Verifica que el servicio exista antes de intentar borrarlo.

```yaml
- uses: ogticrd/actions/actions/gcp/cloudrun/cleanup@main
  with:
    service: my-app-pr-42
    region: us-east1
```

---

## Variables de entorno

Las variables de entorno para Cloud Run se pasan en formato `KEY=VALUE`, una por línea. Se pueden incluir comentarios y líneas vacías:

```yaml
env-vars: |
  # Configuración general
  NODE_ENV=production

  # Secrets del repositorio
  DATABASE_URL=${{ secrets.DATABASE_URL }}
  API_KEY=${{ secrets.API_KEY }}

  # Variables del repositorio
  SENTRY_DSN=${{ vars.SENTRY_DSN }}
```

La action se encarga de convertir esto al formato que Cloud Run necesita. En los logs se imprimen los nombres de las variables (no los valores) para facilitar el debugging.

---

## Registries

### Google Artifact Registry (default)

```yaml
- uses: ogticrd/actions/actions/docker/build@main
  with:
    registry-username: _json_key
    registry-password: ${{ secrets.GAR_JSON_KEY }}
    image-name: project-id/repo/app
```

### GitHub Container Registry

```yaml
- uses: ogticrd/actions/actions/docker/build@main
  with:
    registry: ghcr.io
    registry-username: ${{ github.actor }}
    registry-password: ${{ secrets.GITHUB_TOKEN }}
    image-name: ${{ github.repository }}/app
    cache-strategy: gha
```

---

## Configuración del repositorio

### Variables

| Variable | Descripción | Ejemplo |
|---|---|---|
| `APP_NAME` | Nombre de la aplicación | `my-app` |
| `GOOGLE_PROJECT_ID` | ID del proyecto en GCP | `ogtic-project-123` |
| `GOOGLE_CLOUD_REGION` | Región de Cloud Run | `us-east1` |
| `GOOGLE_ARTIFACT_REGISTRY` | Path del registry en GAR | `us-docker.pkg.dev/project-id/repo` |

### Secretos

| Secreto | Descripción |
|---|---|
| `GAR_JSON_KEY` | JSON key del service account de GCP |

---

## Ejemplos

En la carpeta [`examples/`](./examples) hay workflows completos listos para copiar:

- [`simple-deploy.yml`](./examples/simple-deploy.yml) — Despliegue mínimo en un solo job
- [`full-pipeline.yml`](./examples/full-pipeline.yml) — Build y deploy con variables de entorno, scaling y VPC
- [`multi-environment.yml`](./examples/multi-environment.yml) — Pipeline con dev, staging y producción, incluyendo cleanup de PRs

---

## Licencia

MIT
