# Evaluación del repositorio de InfraLab v2

Fecha de auditoría: 2026-11-02  
Repositorio auditado: raíz de este repositorio  
Repositorio de referencia, inspeccionado únicamente en modo lectura: copia upstream local no versionada

> [!NOTE]
> Esta evaluación conserva la línea base de la auditoría inicial. [`ROADMAP.md`](ROADMAP.md) es la fuente oficial para el estado actual del proyecto. La comparación con la copia upstream se realizó únicamente en modo lectura y fuera del contenido versionado.

## A. Resumen ejecutivo

La copia auditada contiene los cinco componentes requeridos de la aplicación: UI, Catalog, Cart, Orders y Checkout. Los fuentes de esos componentes coinciden con el repositorio upstream en la comparación estática realizada. La copia ya presenta una limpieza parcial respecto del upstream: no contiene workflows bajo `.github/workflows/`, `samples/` ni `terraform/`. No se encontraron archivos `*.tf` ni una implementación de Terraform en la copia auditada.

Todavía permanecen implementaciones DevOps provistas por el proyecto original:

- 5 Dockerfiles de servicios y 3 Dockerfiles auxiliares.
- 8 archivos Docker Compose.
- 6 charts Helm, incluidos el chart agregador y los charts de los cinco servicios.
- 2 Helmfiles, 1 Tiltfile y 7 plantillas Helmfile `*.gotmpl`.
- Scripts para producir distribuciones Compose/Kubernetes, manifests multi-arquitectura, E2E sobre Compose/kind y análisis de imágenes.
- Configuraciones de Renovate, Release Please y Mergify.
- Configuraciones mixtas de Nx, mise y Lefthook que combinan tareas de aplicación con Docker, Helm, Terraform, Kubernetes y artefactos que ya no existen.

La estrategia recomendada es retirar únicamente las implementaciones DevOps inequívocas, conservar íntegros los fuentes, contratos, tests, datos, migraciones, wrappers y atribuciones, y revisar los archivos mixtos antes de editarlos. Los backends opcionales incluidos en el código (DynamoDB, SQS, Bedrock, metadatos AWS, OpenSearch, Redis, PostgreSQL, MySQL y RabbitMQ) son capacidades runtime del código de negocio; no son infraestructura como código y se deben conservar salvo una decisión funcional posterior, fuera del alcance de esta limpieza.

No se ejecutaron compilaciones, tests, instalaciones de dependencias ni comandos que generasen artefactos durante esta fase. La única modificación realizada es este documento.

Los directorios vacíos `.agents/`, `.aws/`, `.codex/` y `.git/` son puntos de montaje/control del entorno de trabajo, no aparecen en el upstream comparado y no se trataron como código del repositorio. No se deben borrar durante la limpieza. En particular, el `.git/` visible no contiene metadata utilizable desde este entorno.

## B. Inventario

### B.1 Componentes principales

| Componente | Ruta real | Stack | Responsabilidad | Persistencia/backend por defecto | Alternativas presentes |
|---|---|---|---|---|---|
| UI | `src/ui/` | Java 21, Spring Boot 3.5.16, Thymeleaf/WebFlux | Frontend server-side y agregador de llamadas a APIs | Mocks cuando no se configuran endpoints | Catalog, Cart, Orders, Checkout y Recommendations remotos; chat Bedrock/OpenAI/mock |
| Catalog | `src/catalog/` | Go 1.25, Gin, GORM | Consulta de productos, tags y búsqueda | Datos locales/in-memory | MySQL y OpenSearch |
| Cart | `src/cart/` | Java 21, Spring Boot 3.5.16 | Carritos por cliente | In-memory | DynamoDB |
| Orders | `src/orders/` | Java 21, Spring Boot 3.5.16, Flyway | Creación y consulta de órdenes; publicación de eventos | In-memory/H2 y mensajería in-memory | PostgreSQL; RabbitMQ o SQS |
| Checkout | `src/checkout/` | Node.js, TypeScript, NestJS 11 | Estado del checkout y orquestación de creación de órdenes | In-memory y Orders mock | Redis y Orders remoto |

### B.2 Módulos auxiliares y compartidos

| Módulo | Ruta | Función | Observación |
|---|---|---|---|
| Agregación de despliegue | `src/app/` | Compose agregado, tracing, Helmfile, Tilt y chart padre | Implementación DevOps original; candidata a retiro completo |
| Contrato Recommendations | `src/recommendations/` | OpenAPI y documentación del servicio opcional | No contiene implementación, pero `src/ui/pom.xml` lo consume para generar/compilar el cliente de UI |
| Estilo Java compartido | `src/misc/style/java/checkstyle.xml` | Reglas Checkstyle | Referenciado por los tres `pom.xml` Java; necesario para lint |
| E2E funcional | `src/e2e/` | Cypress y page objects | Los tests son útiles; su Dockerfile y runner Docker son separables |
| Generador de carga | `src/load-generator/` | Artillery, escenario y helpers | Herramienta auxiliar, no uno de los cinco servicios; requiere decisión explícita |
| Atribución OSS | `oss/` y `src/*/ATTRIBUTION.md` | Generación y conservación de atribuciones | Las atribuciones existentes deben conservarse; el generador basado en Docker requiere revisión |
| Orquestación monorepo | `package.json`, `yarn.lock`, `.yarn/`, `.yarnrc.yml`, `nx.json`, `src/*/project.json` | Nx/Yarn para build, test, lint y herramientas | Mezcla tareas nativas útiles con container/Helm/manifests y referencias obsoletas |
| Documentación | `README.md`, `DEVELOPER_GUIDE.md`, `docs/`, READMEs de servicios | Arquitectura, features y operación | Contiene instrucciones Docker/Kubernetes/Terraform que quedarían obsoletas después de la limpieza |

### B.3 Archivos necesarios para compilar, probar y ejecutar

| Componente | Build y dependencias | Tests | Runtime/configuración que se debe conservar |
|---|---|---|---|
| UI | `src/ui/pom.xml`, `src/ui/mvnw`, `src/ui/mvnw.cmd`, `src/ui/.mvn/`, `src/ui/src/`; además `src/{catalog,cart,orders,checkout,recommendations}/openapi.yml` para Kiota | `src/ui/src/test/` | `src/ui/src/main/resources/`, `application.yml`, `application-prod.yml`, assets, templates, datos mock y clientes generados |
| Catalog | `src/catalog/go.mod`, `src/catalog/go.sum`, `src/catalog/main.go` y paquetes `api/`, `config/`, `controller/`, `httputil/`, `middleware/`, `model/`, `repository/` | `src/catalog/test/`; sus pruebas levantan MySQL mediante Testcontainers | `repository/products.json`, `repository/tags.json`, configuración por variables de entorno y `openapi.yml` |
| Cart | `src/cart/pom.xml`, wrappers Maven y `src/cart/src/main/` | `src/cart/src/test/`; la integración con DynamoDB usa Testcontainers | `application.yml`, `application-prod.yml`, metadata Spring y `openapi.yml` |
| Orders | `src/orders/pom.xml`, wrappers Maven, `src/orders/src/main/`, clases de eventos generadas | `src/orders/src/test/`; la integración PostgreSQL usa Testcontainers | `application*.yml`, `db/migration/V1__Initial.sql`, `events/*.schema.json`, `openapi.yml` |
| Checkout | `src/checkout/package.json`, `yarn.lock`, `.yarn/`, `.yarnrc.yml`, `nest-cli.json`, `tsconfig*.json`, `src/` | `src/checkout/test/`, `test/jest-e2e.json`; la prueba principal usa Redis mediante Testcontainers | `src/config/configuration.ts`, cliente Orders generado bajo `src/clients/orders/`, `openapi.yml` |

Archivos compartidos adicionales:

- `src/misc/style/java/checkstyle.xml` es necesario para `./mvnw checkstyle:checkstyle` en UI, Cart y Orders.
- `src/recommendations/openapi.yml` es necesario aun cuando Recommendations no sea uno de los cinco servicios desplegados.
- Los `ATTRIBUTION.md` y `LICENSE` no son insumos de compilación nativa, pero son obligatorios para licencia y atribución.
- Los `project.json` y la configuración Nx son opcionales para compilación nativa, pero actualmente constituyen la interfaz uniforme documentada del monorepo.

### B.4 Inventario DevOps existente

| Tipo | Rutas encontradas |
|---|---|
| Dockerfiles de servicio | `src/ui/Dockerfile`, `src/catalog/Dockerfile`, `src/cart/Dockerfile`, `src/orders/Dockerfile`, `src/checkout/Dockerfile` |
| Dockerfiles auxiliares | `src/e2e/Dockerfile.run`, `src/load-generator/Dockerfile`, `src/load-generator/Dockerfile.run` |
| Compose | `src/app/docker-compose.yml`, `src/app/compose.override.yaml`, `src/app/docker-compose.tracing.yml`, y `src/{ui,catalog,cart,orders,checkout}/docker-compose.yml` |
| Helm | `src/app/chart/` y `src/{ui,catalog,cart,orders,checkout}/chart/` |
| Helmfile/Tilt | `src/app/helmfile.yaml`, `src/app/helmfile.slim.yaml`, `src/app/templates/`, `src/app/Tiltfile` |
| Observabilidad de despliegue | `src/app/otel-collector-config.yml`, `src/e2e/scripts/assert-traces.sh` |
| Distribución/orquestación | `scripts/compose-dist.sh`, `scripts/kubernetes-dist.sh`, `scripts/create-manifest.sh`, `scripts/e2e-compose.sh`, `scripts/e2e-kind.sh`, `scripts/e2e.sh` |
| Seguridad de imágenes | `scripts/generate-security-report.sh` |
| Automatización de repositorio | `.mergify.yml`, `renovate.json`, `release-please-config.json`, `.release-please-manifest.json` |
| CI/CD | No existe `.github/` ni se encontraron workflows en la copia auditada |
| Terraform | No existe `terraform/` ni se encontraron archivos `*.tf`, `*.tf.json` o `*.tfvars` en la copia auditada |

## C. Arquitectura actual

### C.1 Comunicación entre servicios

```text
Navegador
   |
   v
UI (Spring Boot/Thymeleaf)
   |---- HTTP ----> Catalog ------> in-memory/MySQL ----> OpenSearch opcional
   |---- HTTP ----> Cart ---------> in-memory/DynamoDB
   |---- HTTP ----> Orders -------> in-memory/PostgreSQL
   |                                  |--> in-memory/RabbitMQ/SQS (eventos)
   `---- HTTP ----> Checkout -----> in-memory/Redis
                         |
                         `---- HTTP ----> Orders, o mock si no hay endpoint

UI ---- HTTP opcional ----> Recommendations, o mock si no hay endpoint
```

Los servicios escuchan en el puerto `8080` por defecto. Los puertos `8081`, `8082`, `8083`, `8085` y `8888` son decisiones de los Compose existentes, no requisitos del código. La comunicación se configura mediante variables de entorno:

- UI: `RETAIL_UI_ENDPOINTS_CATALOG`, `RETAIL_UI_ENDPOINTS_CARTS`, `RETAIL_UI_ENDPOINTS_ORDERS`, `RETAIL_UI_ENDPOINTS_CHECKOUT` y endpoint opcional de Recommendations por binding Spring.
- Checkout: `RETAIL_CHECKOUT_ENDPOINTS_ORDERS`.
- Cada servicio selecciona su backend con `RETAIL_*_PERSISTENCE_PROVIDER`; Orders selecciona mensajería con `RETAIL_ORDERS_MESSAGING_PROVIDER`.

Con los valores por defecto, los servicios pueden iniciar con implementaciones in-memory/mock. Para una prueba integrada real de los cinco servicios se deben configurar explícitamente los endpoints de UI y Checkout.

### C.2 Dependencias relevantes

- UI genera clientes desde los contratos OpenAPI hermanos. Retirar un `openapi.yml`, incluido Recommendations, puede romper `mvn package` aunque el servicio remoto no se despliegue.
- Checkout incluye en el árbol de fuentes un cliente Orders generado. Es código necesario para compilar; los archivos de metadata del generador proporcionan trazabilidad.
- Orders requiere su migración Flyway y sus schemas/clases de eventos; no son artefactos DevOps.
- Los datos JSON de Catalog y los datos/assets/templates de UI sostienen los modos in-memory/mock y no deben confundirse con samples descartables.
- OpenTelemetry, Actuator/Prometheus, endpoints de health y código de chaos están integrados en los servicios. Son comportamiento de aplicación/observabilidad y no manifiestos de despliegue.
- Las pruebas de Catalog, Cart, Orders y Checkout usan Testcontainers. Retirar los Dockerfiles del proyecto no las invalida, pero ejecutarlas requiere un runtime Docker compatible y acceso a las imágenes de prueba.

## D. Matriz de decisiones

`KEEP` significa conservar sin cambios durante la limpieza. `REMOVE` significa retirar únicamente después de aprobación. `REVIEW` identifica archivos mixtos, auxiliares o inconsistentes cuya acción exacta debe aprobarse antes de modificar.

| Ruta real | Clasificación | Justificación / decisión propuesta |
|---|---|---|
| `LICENSE` | KEEP | Licencia MIT-0 original; conservación obligatoria. |
| `.agents/`, `.aws/`, `.codex/`, `.git/` | KEEP | Directorios vacíos/controlados por el entorno, fuera del código fuente. No modificar ni usar como candidatos de limpieza. El `.git/` visible no permite operar el repositorio. |
| `src/{ui,catalog,cart,orders,checkout}/ATTRIBUTION.md` | KEEP | Atribución de dependencias por servicio; necesaria para cumplimiento. |
| `CHANGELOG.md`, `CODE_OF_CONDUCT.md` | KEEP | Preservan historia y contexto del proyecto; no implementan DevOps. |
| `README.md`, `DEVELOPER_GUIDE.md`, `CONTRIBUTING.md` | REVIEW | Contienen referencias a Docker, Kubernetes, Terraform, ECR y procesos upstream. Propuesta: conservar atribución y contenido de aplicación, y actualizar solamente instrucciones obsoletas después de aprobar el alcance. |
| `docs/` | REVIEW | Arquitectura/features e imágenes útiles, pero parte del contenido describe plataformas upstream. Revisar texto; no eliminar imágenes funcionales sin comprobar referencias. |
| `.gitignore` | KEEP | Protege secretos locales, dependencias, builds y caches. Las entradas de Terraform/Kubernetes no son una implementación y pueden servir para la futura plataforma. |
| `.prettierrc` | KEEP | Formato de fuentes Java/XML. |
| `.prettierignore` | REVIEW | Incluye charts y Terraform que se retirarían; propuesta: depurar referencias obsoletas sin cambiar reglas relevantes a fuentes. |
| `.mise.toml` | REVIEW | Java, Node, Maven y Go son útiles; Terraform, kubectl, Helm, Helmfile, AWS CLI, tflint, kind y Tilt pertenecen a la plataforma original. Propuesta: retener sólo toolchain de aplicación y utilidades justificadas. |
| `package.json`, `yarn.lock`, `.yarn/`, `.yarnrc.yml` | REVIEW | Nx/Yarn soportan build y herramientas, pero incluyen plugins/commands de containers y Helm. Propuesta conservadora: mantener Nx para build/test y retirar únicamente dependencias/scripts DevOps, actualizando el lockfile de manera controlada. |
| `nx.json` | REVIEW | Contiene defaults útiles de build/test/lint/serve, pero también `@nx-tools/nx-container`, Helm, manifests, ECR y una referencia a `go.work` que no existe. Requiere edición selectiva. |
| `lefthook.yml` | REVIEW | Prettier/gofmt son útiles; Terraform/tflint y `samples/` son referencias obsoletas. Propuesta: conservar hooks de código y retirar los obsoletos. |
| `.mergify.yml`, `renovate.json`, `release-please-config.json`, `.release-please-manifest.json` | REMOVE | Automatización de PRs, dependencias y releases del upstream; se reconstruirá CI/CD sin reutilizar la solución original. |
| `scripts/add-licenses.sh`, `scripts/misc/license-header.txt` | KEEP | Mantienen headers de licencia de los fuentes. |
| `scripts/generate-codescan-report.sh` | KEEP | Análisis estático de código fuente, independiente de la infraestructura de despliegue. |
| `scripts/compose-dist.sh`, `scripts/kubernetes-dist.sh`, `scripts/create-manifest.sh` | REMOVE | Generan las distribuciones Compose/Kubernetes y manifests de imagen del upstream. |
| `scripts/e2e-compose.sh`, `scripts/e2e-kind.sh`, `scripts/e2e.sh` | REMOVE | Acoplan E2E a Compose/kind y al runner Docker original. Los tests Cypress se conservan por separado. |
| `scripts/generate-security-report.sh` | REMOVE | Implementación de análisis de imágenes del upstream; además referencia `scripts/build-image.sh`, archivo que no existe en esta copia. |
| `oss/attribution/` | KEEP | Fuente de la herramienta de atribución y workspace asociado; protege trazabilidad legal. |
| `oss/ort/`, `oss/run.sh`, `src/{ui,catalog,cart,orders,checkout}/scripts/oss.source` | REVIEW | Generador de atribución basado en Docker/ORT. Conservar los `ATTRIBUTION.md` es obligatorio; decidir si también se conserva esta herramienta auxiliar o se reconstruye. |
| `src/app/` | REMOVE | Agregación íntegramente DevOps: Compose, Helm, Helmfile, Tilt, templates, tracing y generación de password. No contiene lógica de los cinco servicios. |
| `src/{ui,catalog,cart,orders,checkout}/Dockerfile` | REMOVE | Imágenes provistas por AWS que se desean reconstruir. Los builds nativos están definidos por Maven, Go y Yarn. |
| `src/{ui,catalog,cart,orders,checkout}/.dockerignore` | REMOVE | Sólo da soporte a los Dockerfiles originales. |
| `src/{ui,catalog,cart,orders,checkout}/docker-compose.yml` | REMOVE | Orquestación y backends locales originales que se reconstruirán. |
| `src/{ui,catalog,cart,orders,checkout}/chart/` | REMOVE | Charts y manifiestos Kubernetes/Helm originales que se reconstruirán. |
| `src/ui/src/main/` | KEEP | Lógica, clientes generados, configuración, datos mock, templates y assets del frontend. |
| `src/ui/src/test/` | KEEP | Tests del servicio UI. |
| `src/ui/pom.xml`, `src/ui/.mvn/`, `src/ui/mvnw`, `src/ui/mvnw.cmd` | KEEP | Build reproducible de UI. `pom.xml` también documenta la dependencia en contratos OpenAPI hermanos. |
| `src/ui/project.json` | REVIEW | Targets nativos útiles, pero hereda container/Helm/manifests, posee tag `chart` y referencia `samples/` ausente. Editar selectivamente si se conserva Nx. |
| `src/ui/README.md` | REVIEW | Documentación funcional útil con sección Docker obsoleta después de la limpieza. |
| `src/ui/scripts/oss.source` | REVIEW | Parte del flujo ORT indicado arriba. |
| `src/ui/src/main/resources/static/assets/.DS_Store`, `src/ui/src/main/resources/static/assets/img/.DS_Store`, `src/ui/src/main/resources/static/assets/img/products/.DS_Store` | REMOVE | Metadata de macOS sin uso de build/runtime; ya está ignorada globalmente. |
| `src/catalog/{api,config,controller,httputil,middleware,model,repository}/`, `src/catalog/main.go` | KEEP | Código y datos del servicio Catalog. |
| `src/catalog/test/` | KEEP | Tests de Catalog; requieren MySQL/Testcontainers para el paquete de integración. |
| `src/catalog/go.mod`, `src/catalog/go.sum`, `src/catalog/openapi.yml` | KEEP | Dependencias reproducibles y contrato consumido por UI. |
| `src/catalog/project.json` | REVIEW | Build/serve útiles, pero `test` y `lint` son no-op; contiene Helm/manifest y `update-samples` apunta a `samples/` ausente. |
| `src/catalog/README.md` | REVIEW | Documentación funcional útil con instrucciones Docker. |
| `src/catalog/scripts/oss.source` | REVIEW | Parte del flujo ORT. |
| `src/cart/src/main/`, `src/cart/src/test/` | KEEP | Código, recursos y tests de Cart. |
| `src/cart/pom.xml`, `src/cart/.mvn/`, `src/cart/mvnw`, `src/cart/mvnw.cmd`, `src/cart/openapi.yml` | KEEP | Build reproducible y contrato consumido por UI. |
| `src/cart/scripts/generate-openapi-spec.sh` | KEEP | Regenera el contrato a partir del servicio; no despliega infraestructura. Requiere Maven, `yq` y credenciales dummy locales. |
| `src/cart/project.json` | REVIEW | Build/test/lint/serve útiles; retirar targets/tags heredados de manifest/Helm si se conserva Nx. |
| `src/cart/README.md` | REVIEW | Documentación funcional útil con instrucciones Docker. |
| `src/cart/scripts/oss.source` | REVIEW | Parte del flujo ORT. |
| `src/orders/src/main/`, `src/orders/src/test/`, `src/orders/events/` | KEEP | Código, migración, eventos, recursos y tests de Orders. |
| `src/orders/pom.xml`, `src/orders/.mvn/`, `src/orders/mvnw`, `src/orders/mvnw.cmd`, `src/orders/openapi.yml` | KEEP | Build reproducible y contrato consumido por UI/Checkout. |
| `src/orders/scripts/generate-events.sh`, `src/orders/scripts/generate-openapi-spec.sh` | KEEP | Herramientas de codegen de contratos/eventos; no son despliegue. `generate-events.sh` es destructivo sobre clases generadas y sólo debe ejecutarse de forma deliberada. |
| `src/orders/project.json` | REVIEW | Build/test/lint/serve útiles; retirar targets/tags de manifest/Helm si se conserva Nx. |
| `src/orders/README.md` | REVIEW | Documentación funcional útil, pero contiene instrucciones Docker y nombra erróneamente `RETAIL_CHECKOUT_PERSISTENCE_PROVIDER` en vez de la configuración Orders observada en código. |
| `src/orders/scripts/oss.source` | REVIEW | Parte del flujo ORT. |
| `src/checkout/src/`, `src/checkout/test/` | KEEP | Lógica, cliente Orders generado y tests del servicio Checkout. |
| `src/checkout/package.json`, `src/checkout/yarn.lock`, `src/checkout/.yarn/`, `src/checkout/.yarnrc.yml`, `src/checkout/nest-cli.json`, `src/checkout/tsconfig*.json`, `src/checkout/eslint.config.mjs`, `src/checkout/.prettierrc` | KEEP | Build, ejecución, lint y test reproducibles del servicio. |
| `src/checkout/openapi.yml` | KEEP | Contrato consumido por UI. |
| `src/checkout/.env` | KEEP LOCAL / DO NOT COMMIT | Archivo local no requerido por el código observado. Está excluido por Git y por el contexto Docker; su valor no se reproduce en este informe. |
| `src/checkout/src/clients/orders/git_push.sh` | REMOVE | Boilerplate del generador para publicar el cliente en otro repositorio; no participa en build/runtime. |
| `src/checkout/src/clients/orders/` excepto `git_push.sh` | KEEP | Cliente generado necesario para compilar y llamar Orders; mantener código y metadata de generación. |
| `src/checkout/scripts/generate-clients.sh`, `src/checkout/scripts/generate-openapi-spec.sh` | REVIEW | Son herramientas de codegen, pero actualmente presentan referencias obsoletas: no existe el script npm `generate:clients` y el OpenAPI se intenta leer desde `/v2/api-docs`, mientras Nest publica Swagger bajo `/api`. No ejecutar ni eliminar sin decisión. |
| `src/checkout/project.json` | REVIEW | Orquesta install/build/test/lint, pero hereda targets de package/Nx y agrega Helm/manifest. Debe depurarse si se conserva Nx. |
| `src/checkout/README.md` | REVIEW | Indica `yarn start`, script que no existe; los scripts reales son `serve`, `serve:dev` y `serve:prod`. También contiene instrucciones Docker. |
| `src/checkout/scripts/oss.source` | REVIEW | Parte del flujo ORT. |
| `src/recommendations/openapi.yml`, `src/recommendations/README.md` | KEEP | El contrato es dependencia de build de UI; README explica el componente opcional. |
| `src/misc/style/java/checkstyle.xml` | KEEP | Dependencia de lint compartida por los servicios Java. |
| `src/e2e/cypress/`, `src/e2e/cypress.config.js`, `src/e2e/package.json`, `src/e2e/yarn.lock`, `src/e2e/.yarn/`, `src/e2e/.yarnrc.yml`, `src/e2e/project.json` | KEEP | Suite funcional independiente que validará la aplicación cuando exista un entorno ejecutable. |
| `src/e2e/Dockerfile.run`, `src/e2e/scripts/run-docker.sh` | REMOVE | Runner/container E2E original; Cypress puede ejecutarse de forma nativa o integrarse luego en la plataforma propia. |
| `src/e2e/scripts/assert-traces.sh`, `src/e2e/README.md` | REVIEW | La aserción depende de Jaeger y del overlay de tracing que se propone retirar; conservarla sólo si se quiere reutilizar el criterio funcional, no la infraestructura. Actualizar README si se conserva. |
| `src/load-generator/Dockerfile`, `src/load-generator/Dockerfile.run`, `src/load-generator/scripts/run-docker.sh`, `src/load-generator/scripts/run-test.sh` | REMOVE | Implementación Docker y acoplamiento a la red Compose original. |
| `src/load-generator/helpers.js`, `src/load-generator/scenario.yml`, `src/load-generator/package.json`, `src/load-generator/yarn.lock`, `src/load-generator/project.json`, `src/load-generator/README.md` | REVIEW | Prueba de carga útil pero auxiliar y fuera de los cinco servicios. Recomendación: conservar escenario/helpers como validación y actualizar su ejecución a una forma propia más adelante. |

## E. Estructura objetivo

Árbol propuesto después de aprobar y ejecutar la limpieza. Los módulos marcados como opcionales dependen de las decisiones `REVIEW`; no se crearán directorios nuevos de infraestructura en esta fase.

```text
infralab-v2/
├── LICENSE
├── README.md                         # ajustado al repositorio de fuentes
├── CHANGELOG.md
├── CODE_OF_CONDUCT.md
├── CONTRIBUTING.md                  # revisado
├── DEVELOPER_GUIDE.md               # revisado
├── REPOSITORY_ASSESSMENT.md
├── .gitignore
├── .mise.toml                       # sólo toolchain de aplicación aprobado
├── .prettierrc
├── .prettierignore
├── .yarn/
├── .yarnrc.yml
├── package.json                     # Nx/tooling sin container/Helm
├── yarn.lock
├── nx.json                          # build/test/lint/serve, sin despliegue
├── lefthook.yml                     # sólo hooks de fuentes aprobados
├── docs/                            # documentación de aplicación revisada
├── oss/
│   └── attribution/                 # y ORT sólo si se aprueba conservarlo
├── scripts/
│   ├── add-licenses.sh
│   ├── generate-codescan-report.sh
│   └── misc/license-header.txt
└── src/
    ├── ui/
    │   ├── .mvn/  mvnw  mvnw.cmd  pom.xml  project.json
    │   ├── ATTRIBUTION.md  README.md
    │   └── src/
    ├── catalog/
    │   ├── go.mod  go.sum  main.go  openapi.yml  project.json
    │   ├── api/ config/ controller/ httputil/ middleware/ model/ repository/
    │   └── test/
    ├── cart/
    │   ├── .mvn/  mvnw  mvnw.cmd  pom.xml  openapi.yml  project.json
    │   └── src/
    ├── orders/
    │   ├── .mvn/  mvnw  mvnw.cmd  pom.xml  openapi.yml  project.json
    │   ├── events/
    │   └── src/
    ├── checkout/
    │   ├── .yarn/  .yarnrc.yml  package.json  yarn.lock
    │   ├── nest-cli.json  tsconfig.json  tsconfig.build.json
    │   ├── openapi.yml  project.json
    │   ├── src/
    │   └── test/
    ├── recommendations/
    │   └── openapi.yml
    ├── misc/style/java/checkstyle.xml
    ├── e2e/                          # Cypress sin runner Docker
    └── load-generator/               # opcional; Artillery sin Docker
```

Ausencias deliberadas en el árbol objetivo: `src/app/`, Dockerfiles, archivos Compose, charts Helm, Helmfiles, Tiltfile, manifiestos Kubernetes originales, Terraform upstream y automatización CI/CD/release upstream.

## F. Plan de ejecución

1. Obtener aprobación explícita de esta evaluación y confirmar que los comandos se ejecutan desde la raíz de la copia de trabajo. Resolver además cada grupo `REVIEW`, especialmente Nx, documentación, ORT, load generator y trace assertions.
2. Antes de borrar, generar en `/tmp` un inventario y checksums de todos los archivos `KEEP`, y repetir la comparación de sólo lectura contra una copia upstream local. No escribir en upstream.
3. Ejecutar una validación baseline de builds/tests nativos según la sección H. Registrar cualquier fallo preexistente; no atribuirlo a la limpieza.
4. Retirar sólo los paths `REMOVE` aprobados. Hacerlo por grupos pequeños: primero `src/app/`; luego Dockerfiles/Compose/charts por servicio; luego scripts DevOps; finalmente automatización y archivos auxiliares.
5. Editar únicamente los paths `REVIEW` aprobados. Mantener build/test/lint/serve nativos; quitar targets, plugins, tags y documentación DevOps obsoletos. No actualizar versiones de dependencias.
6. Comprobar que no se modificaron fuentes, tests, contratos, migraciones, datos, assets, lockfiles de servicio o atribuciones salvo una excepción expresamente aprobada.
7. Repetir los builds/tests nativos y comparar resultados con la baseline. Ejecutar smoke tests in-memory y, donde haya Docker disponible, integraciones Testcontainers.
8. Ejecutar Cypress contra los cinco procesos configurados por endpoints, sin usar los Compose retirados. La futura implementación propia podrá automatizar esta etapa.
9. Confirmar por búsqueda que ya no permanecen Dockerfiles, Compose, charts/Helmfile/Tilt, Terraform ni automatización upstream no aprobada.
10. Actualizar exclusivamente este documento con las secciones J–N: cambios, validaciones, diferencias, estado final y próximos pasos.

## G. Riesgos

| Riesgo | Impacto | Mitigación |
|---|---|---|
| Ambigüedad del directorio de trabajo | Limpiar la copia equivocada | Confirmar la raíz del repositorio antes de cualquier operación. |
| `.git/` aparece vacío/no utilizable en la copia auditada; `git status` devuelve “not a git repository” | No hay diff/status local confiable ni rollback por Git en este entorno | Usar checksums y comparación con upstream antes/después; no iniciar ni alterar Git sin aprobación. |
| Configuración Nx mezclada | Eliminar `package.json`, `nx.json` o `project.json` puede perder la interfaz uniforme de build/test | Editar selectivamente o validar primero todos los comandos nativos. |
| Contratos OpenAPI hermanos | UI puede dejar de compilar si se elimina cualquier contrato, especialmente `src/recommendations/openapi.yml` | Clasificar todos los contratos como KEEP y verificar build UI. |
| Clientes generados | Borrar `src/ui/.../client/` o `src/checkout/src/clients/orders/` rompe compilación/runtime | Conservar el código generado; separar sólo boilerplate no usado. |
| Migración y eventos de Orders | Pérdida de esquema o incompatibilidad de mensajería | Conservar `db/migration/`, `events/` y clases generadas. |
| Datos/assets in-memory | UI/Catalog pueden iniciar pero fallar al navegar si se eliminan JSON, imágenes o templates | Conservar `repository/*.json` y todos los recursos UI salvo `.DS_Store`. |
| Testcontainers | Integraciones fallan sin Docker, red o acceso a imágenes aunque el código sea correcto | Separar build/unit de integración y documentar `SKIPPED` por entorno cuando corresponda. |
| Cobertura declarada engañosa | `src/catalog/project.json` define `test` y `lint` como `exit 0`; `src/checkout/package.json` define `test` como `exit 0` | Ejecutar los comandos reales `go test -v ./test/...` y `yarn test:integration`; no usar el no-op como evidencia. |
| Scripts obsoletos | `generate-security-report.sh` referencia un archivo ausente; codegen Checkout referencia script/endpoint inexistentes | No ejecutarlos; retirar o reparar sólo con aprobación específica. |
| Documentación obsoleta | Checkout documenta `yarn start` inexistente y Orders documenta una variable incorrecta | Actualizar documentación de servicio sin cambiar lógica. El README raíz ya diferencia la plataforma actual de la planificada. |
| Versiones divergentes | `.mise.toml` fija Node 22; Dockerfile Checkout usaba Node 20; README indica Node >=16 | Adoptar como baseline el toolchain nativo declarado/aprobado y registrar versiones reales en validación. |
| Go/CGO | Catalog depende indirectamente de `go-sqlite3`; ciertos entornos pueden requerir CGO y toolchain C | Verificar `go build` en el entorno objetivo antes de concluir que la limpieza causó el fallo. |
| Atribución/licencias | Eliminar `LICENSE`, `ATTRIBUTION.md` o tooling sin retener outputs compromete cumplimiento | Mantener licencia y atribuciones como invariantes. |
| Archivos `.env` locales | Pueden inducir a versionar o incorporar configuración sensible a imágenes | Mantenerlos ignorados por Git y Docker, usar `.env.example` con placeholders y nunca copiar valores al informe. |
| Código AWS opcional | Confundir SDKs/backends runtime con IaC puede llevar a borrar funcionalidad | Conservar dependencias/código; la limpieza se limita a implementaciones de plataforma. |
| Baseline ya parcialmente limpiada | Los workflows originales, `samples/` y `terraform/` ya faltan; algunos targets aún los referencian | Tratar referencias como deuda de configuración, no restaurar esas carpetas. |

## H. Plan de validación

Los siguientes comandos son el plan; **no fueron ejecutados en la Fase 1**. Deben correrse primero como baseline y luego de la limpieza, registrando código de salida y errores reales.

### H.1 Preflight y trazabilidad

```bash
cd /path/to/infralab-v2
java -version
mvn -version
go version
node --version
yarn --version

diff -qr \
  --exclude=.git --exclude=.agents --exclude=.aws --exclude=.codex \
  ../retail-store-upstream \
  .
```

Criterios:

- Toolchain compatible con Java 21, Go 1.25 y la versión Node/Yarn finalmente aprobada.
- Antes de limpiar, la comparación debe mostrar sólo diferencias conocidas: `.github/`, `samples/`, `terraform/` y este assessment.
- Después de limpiar, toda diferencia nueva debe corresponder a un path aprobado.

Para fuentes críticos, comparar de forma específica:

```bash
diff -qr ../retail-store-upstream/src/ui/src       src/ui/src
diff -qr ../retail-store-upstream/src/catalog     src/catalog     --exclude=chart --exclude=Dockerfile --exclude=.dockerignore --exclude=docker-compose.yml --exclude=project.json --exclude=README.md --exclude=.DS_Store
diff -qr ../retail-store-upstream/src/cart/src     src/cart/src
diff -qr ../retail-store-upstream/src/orders/src  src/orders/src
diff -qr ../retail-store-upstream/src/checkout/src src/checkout/src --exclude=git_push.sh
```

Criterio: ninguna diferencia de lógica de negocio, tests, recursos, migraciones o clientes generados, salvo cambios aprobados de antemano.

### H.2 Build y tests por servicio

UI:

```bash
cd src/ui
./mvnw --no-transfer-progress clean verify
./mvnw checkstyle:checkstyle
```

Criterio: exit code 0; tests y Checkstyle aprobados; JAR generado; los cinco contratos enumerados (Catalog, Cart, Orders, Checkout y Recommendations) resuelven correctamente.

Catalog:

```bash
cd src/catalog
go build -o /tmp/infralab-v2-catalog ./main.go
go test -v ./test/...
```

Criterio: binario generado fuera del repositorio; tests aprobados. El segundo comando requiere Docker/Testcontainers y MySQL; si el entorno no lo permite, registrar `SKIPPED` con causa, no `PASS`.

Cart:

```bash
cd src/cart
./mvnw --no-transfer-progress clean test -DexcludedGroups=integration
AWS_ACCESS_KEY_ID=dummy AWS_SECRET_ACCESS_KEY=dummy ./mvnw --no-transfer-progress test -Dgroups=integration
./mvnw checkstyle:checkstyle
```

Criterio: unitarios y lint con exit code 0; integración con DynamoDB/Testcontainers aprobada o documentada como `SKIPPED` por falta de runtime.

Orders:

```bash
cd src/orders
./mvnw --no-transfer-progress clean test -DexcludedGroups=integration
./mvnw --no-transfer-progress test -Dgroups=integration
./mvnw checkstyle:checkstyle
```

Criterio: unitarios y lint con exit code 0; integración PostgreSQL/Testcontainers aprobada o `SKIPPED` justificado; Flyway carga `V1__Initial.sql`.

Checkout:

```bash
cd src/checkout
yarn install --immutable
yarn lint
yarn build
yarn test:integration
```

Criterio: install no modifica `yarn.lock`; lint/build con exit code 0; integración Jest aprobada o `SKIPPED` justificado si Redis/Testcontainers no puede ejecutarse. `yarn test` no es un criterio porque actualmente es un no-op.

Validación opcional de la orquestación Nx, sólo si se aprueba conservarla y depurarla:

```bash
cd /path/to/infralab-v2
yarn install --immutable
yarn nx run-many -t build --projects=ui,catalog,cart,orders,checkout
```

Criterio: los cinco builds terminan con exit code 0 sin invocar Docker, Helm ni manifest generation.

### H.3 Smoke test de los cinco servicios

Iniciar cada proceso en terminal separada usando backends in-memory y puertos distintos; no usar los Compose originales:

```bash
cd src/catalog && PORT=8081 go run main.go
cd src/cart && PORT=8082 ./mvnw spring-boot:run
cd src/orders && PORT=8083 ./mvnw spring-boot:run
cd src/checkout && PORT=8085 RETAIL_CHECKOUT_ENDPOINTS_ORDERS=http://127.0.0.1:8083 yarn serve
cd src/ui && PORT=8888 \
  RETAIL_UI_ENDPOINTS_CATALOG=http://127.0.0.1:8081 \
  RETAIL_UI_ENDPOINTS_CARTS=http://127.0.0.1:8082 \
  RETAIL_UI_ENDPOINTS_ORDERS=http://127.0.0.1:8083 \
  RETAIL_UI_ENDPOINTS_CHECKOUT=http://127.0.0.1:8085 \
  ./mvnw spring-boot:run
```

Verificaciones mínimas:

```bash
curl --fail http://127.0.0.1:8081/health
curl --fail http://127.0.0.1:8082/actuator/health
curl --fail http://127.0.0.1:8083/actuator/health
curl --fail http://127.0.0.1:8085/health
curl --fail http://127.0.0.1:8888/actuator/health
curl --fail http://127.0.0.1:8888/home
```

Criterios:

- Los cinco procesos alcanzan estado healthy.
- UI puede listar catálogo, manipular un carrito, completar checkout y consultar la orden mediante los endpoints configurados.
- No hay dependencia en un archivo Docker/Compose/Helm retirado.
- Los logs no exponen credenciales ni valores sensibles.

### H.4 E2E funcional

Con los cinco procesos activos:

```bash
cd src/e2e
yarn install --immutable
CYPRESS_BASE_URL=http://127.0.0.1:8888 yarn execute
```

Criterio: todas las specs aplicables de home, catalog, cart, checkout y search terminan satisfactoriamente. Si search queda deshabilitado por configuración, registrar el alcance exacto en vez de declarar cobertura total.

### H.5 Comprobación de limpieza

```bash
cd /path/to/infralab-v2
find . -type f \( -name 'Dockerfile' -o -name 'Dockerfile.*' \)
find . -type f \( -iname '*compose*.yml' -o -iname '*compose*.yaml' \)
find . -type d -name chart
find . -type f \( -name 'helmfile*.yaml' -o -name 'Tiltfile' -o -name '*.tf' -o -name '*.tf.json' -o -name '*.tfvars' \)
find . -type f -path '*/.github/workflows/*'
```

Criterio: sin resultados para implementaciones originales, excepto cualquier path que el usuario haya aprobado explícitamente conservar. Las referencias textuales históricas o de atribución deben evaluarse por contexto y no eliminarse automáticamente.

## I. Estado

**BASELINE COMPLETADA**

Esta evaluación documenta la línea base inicial. Las propuestas de limpieza y los elementos `REVIEW` no autorizan cambios por sí mismos; el estado vigente y las fases aprobadas se mantienen en [`ROADMAP.md`](ROADMAP.md).
