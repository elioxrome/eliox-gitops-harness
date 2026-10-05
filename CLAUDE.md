# Contexto: ecosistema GitOps de Eliezer

Este repo no tiene código propio que correr. Es el mapa de referencia entre los
repos reales y de laboratorio que conforman un mismo flujo GitOps
(Bitbucket/GitHub → Jenkins/GitHub Actions → eliox-platform-config → ArgoCD →
clusters). Léelo antes de tocar cualquiera de los repos listados abajo si no
tienes ya este contexto cargado en la conversación.

Diagramas del entramado (actualízalos junto con este archivo cuando algo del
mapa cambie — si no, se desincronizan rápido):
- [`arquitectura-devops.mmd`](arquitectura-devops.mmd): flujo CI/CD de los
  dos flujos (qué repo dispara qué, qué commitea/publica qué).
- [`infraestructura-lab.mmd`](infraestructura-lab.mmd): qué corre dónde en
  el laptop (clusters kind, namespaces, runners self-hosted) — el "mapa
  físico", separado del flujo.

Hay **dos flujos que no deben mezclarse conceptualmente**: el flujo real de la
empresa (parcialmente reproducido en local) y el laboratorio nuevo
(`argocd-kind-lab`). Ver la sección "Dos flujos" más abajo antes de asumir cuál
aplica a lo que te estén pidiendo.

Desde 2026-10-05 **ambos corren a la vez, en vivo, dentro del mismo
`argocd-it4t`** (namespace `argocd-it4t` del cluster `hub`, ver "Cómo levantar
el laboratorio"): `mc-user-fastapi-dev` (lab, namespace `apps-dev`) y
`mc-user-fastapi-platform-dev` (flujo real, namespace `apps`). Coexisten sin
chocar porque tienen nombres y namespaces de destino distintos — pero
cualquier `Application`/recurso nuevo que agregues tiene que seguir evitando
esa colisión a mano, el namespace de ArgoCD no lo hace por ti.

## Mapa de repos

| Repo | Rol | Ubicación local | Remoto |
|---|---|---|---|
| **mc-user-fastapi** | Microservicio real (FastAPI). Tiene su propio `Jenkinsfile` (usa `eliox-jenkins-shared-library`) y `main.yml`: build+push de la imagen (tag `dev-<run>-<sha8>`) + PR a `eliox-platform-config`. Ya **no** tiene workflow propio para el lab — ese paso lo dispara `eliox-platform-config` (ver "Automatización del laboratorio"). | `~/works/gitops-eliox/mc-user-fastapi` (clon activo, el que se edita) · `~/works/mc-user-fastapi` (clon viejo de mayo 2026, prototipo de un solo cluster `local-cluster`, no editar) | `github.com/elioxrome/mc-user-fastapi` |
| **eliox-platform-config** | Config real de la plataforma: un chart Helm por microservicio bajo `charts/`, values por entorno bajo `environments/<env>/`, `Applications` de ArgoCD bajo `argocd/applications/`. | `~/works/gitops-eliox/eliox-platform-config` | `github.com/elioxrome/eliox-platform-config` (branches: `main`, `gitops`, `feature/argocd`, PRs automáticos `pr-gitops-dev-*`) |
| **eliox-jenkins-shared-library** | Librería compartida de Jenkins (`vars/enterprisePipeline.groovy`) que consumen los `Jenkinsfile` de cada microservicio vía `@Library('eliox-jenkins-shared-library@...')`. | `~/works/sabadell/eliox-jenkins-shared-library` (clon activo) · `~/works/devops/kind_cluster/eliox-jenkins-shared-library` (clon viejo de feb 2026, no editar) | `github.com/elioxrome/eliox-jenkins-shared-library` |
| **devops-workflows** | Workflows reutilizables de GitHub Actions (CI build+push a GHCR, CD a VPS por Docker Compose) para otros proyectos de Eliezer. No forma parte directa del flujo de mc-user-fastapi, pero sigue la misma filosofía "build once, deploy many". | `~/works/devops-workflows` (clon activo) | `github.com/elioxrome/devops-workflows` |
| **generic-charts-ms** | Chart Helm genérico (`microservice`) para cualquier microservicio stateless: Deployment+Service+HPA+PDB+NetworkPolicy+Ingress opcional. Reemplaza, en el laboratorio, al chart por-servicio de `eliox-platform-config`. Tiene su propio pipeline de GitHub Actions que publica cada versión a Artifactory usando un runner self-hosted en este laptop. | `~/works/gitops-eliox/generic-charts-ms` | `github.com/elioxrome/generic-charts-ms` |
| **argocd-kind-lab** | Laboratorio local: kind multi-cluster (`hub` + un spoke por "tipo") + una instancia de ArgoCD por Helm por tipo + Artifactory, todo como IaC versionado. Reproduce la arquitectura real de la empresa (5 instancias ArgoCD por tipo: IT4T/DIGITAL/EOP/PSC/BCC) reducida a `it4t`+`digital` para no gastar RAM. | `~/works/gitops-eliox/argocd-kind-lab` | `github.com/elioxrome/argocd-kind-lab` |

## Dos flujos — no mezclar

**1. Flujo real de la empresa (parcialmente reproducido):**
`mc-user-fastapi` → build (Jenkins vía `eliox-jenkins-shared-library`, o el
workflow de GitHub Actions `main.yml`) → push de imagen a Docker Hub → PR
automático a `eliox-platform-config` (`environments/dev/mc-user-fastapi-values.yaml`,
branch `gitops`) → **merge manual del PR (gobernanza: nadie te obliga, pero
el merge es el punto de revisión/control)** → ArgoCD aplica. El chart vive
**dentro** de `eliox-platform-config` (`charts/mc-user-fastapi`), un chart
por servicio. Como ArgoCD lee los values en vivo de `gitops` en cada sync
(no hay paso de empaquetado/versionado de por medio), cualquier cambio en
ese archivo *solo* se refleja si pasó por ese PR+merge — es la fuente de
verdad real, no un espejo de lo desplegado.

La `Application` (`argocd/applications/mc-user-fastapi-dev.yaml`) está
**aplicada y corriendo** desde 2026-10-05 en `argocd-it4t` con el nombre
`mc-user-fastapi-platform-dev` (namespace destino `apps`) — el archivo trae
`namespace: argocd`/`name: mc-user-fastapi-dev` originales, que no sirven
tal cual en este lab (no existe el namespace `argocd`, y ese nombre choca
con la `Application` del flujo del lab); se corrigieron a mano en el propio
YAML (en `main` y `gitops`) y así hay que mantenerlo si se reinstala desde
cero.

**2. Flujo del laboratorio (`argocd-kind-lab`):**
kind con un cluster `hub` + spokes por tipo (hoy solo `it4t` + `digital`
activos; `eop`/`psc`/`bcc` desactivados en `kind/delete/` para ahorrar RAM) +
una instancia ArgoCD Helm por tipo en el hub + Artifactory local (IaC en
`artifactory/`). El chart **ya no es específico de mc-user-fastapi**: es
`generic-charts-ms`, publicado por su propio pipeline de GitHub Actions
(runner self-hosted, label `kind-local`, instalado como systemd en este
laptop: `actions.runner.elioxrome-generic-charts-ms.laptop-eliezer-romero.service`)
a Artifactory. Cada microservicio aporta un `values.yaml` de pocas líneas
(ver más abajo **de dónde** sale ese archivo — es importante, no es una copia
propia del lab); se empaqueta como "release chart"
(`artifactory/publish-release-chart.sh`) que trae el chart genérico
vendorizado como dependencia Helm real (no copia de templates). El
`ApplicationSet` de ArgoCD en el lab
(`argocd/applicationsets/mc-user-fastapi-appset.yaml`) apunta a ese release
chart en Artifactory, no a `eliox-platform-config`. Corre directo en el hub
vía la instancia `argocd-it4t`, sin spoke, con un namespace por entorno
(`apps-{env}`) — generador `list`, solo `dev` por ahora.

### Automatización del laboratorio — un solo `values.yaml` (2026-10-05)

El paso de "publicar release chart + instalar/actualizar en ArgoCD" está
automatizado end-to-end, probado en vivo. **Decisión clave: no hay un
`values.yaml` separado para el lab.** La primera versión de esta
automatización sí mantenía uno propio (en `mc-user-fastapi` y luego en
`argocd-kind-lab`) — se descartó a propósito: dos archivos editables para lo
mismo es exactamente la duplicación que no se quería. La única fuente
editable es la que ya existía para el flujo real:
`eliox-platform-config/environments/dev/mc-user-fastapi-values.yaml`.

- **Dónde vive**: `eliox-platform-config/.github/workflows/deploy-lab.yml`
  (no en `mc-user-fastapi` ni en `argocd-kind-lab` — vive donde vive el
  archivo que lo dispara). Trigger: `push` a la branch **`gitops`** con
  filtro de `paths` sobre ese archivo exacto, **y** `workflow_dispatch`. No
  dispara con el PR abierto, solo cuando se mergea (eso es lo que mueve la
  branch `gitops`).
- **No construye ninguna imagen propia.** Reusa la que ya construyó y pusheó
  `mc-user-fastapi/main.yml` — mismo tag `dev-<run>-<sha8>` para los dos
  despliegues (real y lab). Ya no existe el tag `lab-dev-*` de la primera
  versión de esta automatización.
- Pasos: checkout de `argocd-kind-lab` y `generic-charts-ms` (ambos
  públicos, sin token — a diferencia de la primera versión, acá nadie
  necesita permiso de push a ningún repo ajeno) → copia
  `environments/dev/mc-user-fastapi-values.yaml` a un archivo **temporal,
  nunca commiteado**, solo para inyectarle con `yq` el campo `nameOverride`
  que exige `generic-charts-ms` (ver gotcha) y que no tiene sentido guardar
  en el archivo real (ese campo no aplica al chart propio de
  `charts/mc-user-fastapi` del flujo real) → `publish-release-chart.sh` →
  `kubectl apply` del `ApplicationSet` → `argocd.argoproj.io/refresh=hard`
  (necesario, ver gotcha de cacheo de versión más abajo).
- **Runner**: self-hosted dedicado para `eliox-platform-config`
  (`actions.runner.elioxrome-eliox-platform-config.laptop-eliezer-romero-eliox-platform-config.service`,
  label `kind-local`). Un runner self-hosted solo puede registrarse contra
  un repo a la vez (sin Organización de GitHub detrás no hay forma de
  compartirlo entre repos personales), así que es uno nuevo, no el de
  `generic-charts-ms`.
- **Secrets** en `eliox-platform-config`: `ARTIFACTORY_USER` /
  `ARTIFACTORY_PASSWORD` (copiados a mano desde
  `argocd-kind-lab/.generated/artifactory-admin.env`).
- **Pendiente de limpiar**: el runner self-hosted
  `actions.runner.elioxrome-mc-user-fastapi.laptop-eliezer-romero-mc-user-fastapi.service`
  (registrado para la primera versión, cuando `deploy-lab.yml` vivía en
  `mc-user-fastapi`) quedó **sin ningún workflow que lo use** — `main.yml`
  corre en `ubuntu-latest` (GitHub-hosted), no self-hosted. Se puede parar y
  desregistrar (`./config.sh remove` desde `~/actions-runners/mc-user-fastapi/`,
  más `sudo ./svc.sh uninstall`), no se hizo todavía.

Además existe `argocd/applicationsets/digital-guestbook-appset.yaml`: un
segundo ejemplo, sobre la instancia `argocd-digital`, que usa el generador
`clusters` built-in de ArgoCD (detecta los spokes registrados vía
`register-clusters.sh` por su label `role: spoke`) contra el repo público
`argoproj/argocd-example-apps`. Es el patrón hub+spoke "de verdad" (a
diferencia de `mc-user-fastapi`, que no usa spoke); sirve para practicar ese
generador, no para desplegar nada real.

## Cómo levantar el laboratorio (`argocd-kind-lab`) de punta a punta

Todo esto se corre **desde `~/works/gitops-eliox/argocd-kind-lab`**. Son los
mismos pasos que ya probamos a mano; quedan aquí centralizados para no
redescubrirlos cada vez. Orden estricto — cada fase depende de la anterior.

**Requisitos:** `docker`, `kind`, `kubectl`, `helm`, `openssl`, `python3`
(con `pyyaml`, lo usa `publish-release-chart.sh`).

**Fase 0 — ¿ya está algo levantado?**
```bash
kind get clusters                 # ¿existen ya "hub"/"digital"?
./argocd/status.sh                # resumen de instancias ArgoCD en el hub
```
Si ya existen, salta las fases que ya estén hechas — los scripts son
idempotentes (`up.sh`/`install.sh` detectan lo que ya existe).

**Fase 1 — clusters kind + instancias de ArgoCD**
```bash
./up.sh                                  # crea hub + spoke "digital"
./argocd/install.sh                      # instala argocd-it4t + argocd-digital en el hub
./argocd/register-clusters.sh digital    # registra el spoke "digital" (solo si vas a usarlo; mc-user-fastapi NO lo necesita, va directo a it4t)
./argocd/status.sh                       # confirma que todo esté arriba
```
UI/password (opcional, en terminal aparte):
```bash
./argocd/port-forward.sh it4t 8443       # https://localhost:8443
./argocd/admin-password.sh it4t
```

**Fase 2 — Artifactory**
```bash
./artifactory/install.sh                 # una sola vez; si ya existe, no lo reinstala (no toca su volumen)
```
Primer login (`./artifactory/port-forward.sh` → http://localhost:8082,
`admin`/`password`) obliga a cambiar el password. Después, crea a mano
`.generated/artifactory-admin.env` (gitignored, no se genera solo):
```bash
cat > .generated/artifactory-admin.env <<'EOF'
ARTIFACTORY_USER=admin
ARTIFACTORY_PASSWORD=<el password real que pusiste en el login>
EOF
```
⚠️ Si este archivo ya existe, **no asumas que el password es el correcto** —
ver gotcha de `.generated/artifactory-admin.env` desactualizado más abajo.

Registra Artifactory como repo Helm dentro de la instancia que lo vaya a usar
(para `mc-user-fastapi`, es `it4t`):
```bash
./artifactory/register-argocd-repo.sh it4t
```

**Fases 3 y 4, automatizadas:** una vez hechas las Fases 0-2 (clusters +
ArgoCD + Artifactory con su repo registrado en `it4t`), publicar el chart de
`mc-user-fastapi` e instalarlo/actualizarlo en ArgoCD ya **no hace falta
hacerlo a mano** — corre solo con cada push a `main` de `mc-user-fastapi`, o
a demanda con el botón "Run workflow" en
`github.com/elioxrome/mc-user-fastapi/actions/workflows/deploy-lab.yml`. Ver
"Automatización del laboratorio" más arriba. Lo manual de abajo sigue
sirviendo para otros microservicios que todavía no tengan su propio
`deploy-lab.yml`, o para depurar el pipeline paso a paso.

**Fase 3 — publicar los charts (manual)**
```bash
# 1. Chart genérico (una vez, o cada vez que cambie generic-charts-ms)
./artifactory/publish-chart.sh ~/works/gitops-eliox/generic-charts-ms

# 2. Release chart por microservicio/entorno: un values.yaml a mano + nombre + versión
#    (edita artifactory/releases/<microservicio>/values.yaml con el tag de imagen
#    que quieras — una carpeta por microservicio, no reusar la de otro)
./artifactory/publish-release-chart.sh \
  ~/works/gitops-eliox/generic-charts-ms \
  ./artifactory/releases/mc-user-fastapi/values.yaml \
  mc-user-fastapi 0.1.0-dev
```
Repite el paso 2 (con nueva versión o el mismo `0.1.0-dev` sobrescrito) cada
vez que cambie el `image.tag`/`image.repository` que quieres desplegar.

**Fase 4 — instalar el microservicio en ArgoCD (manual)**
```bash
kubectl --context kind-hub apply -f argocd/applicationsets/mc-user-fastapi-appset.yaml
kubectl --context kind-hub -n argocd-it4t get applications
kubectl --context kind-hub -n apps-dev get pods
```

**Opcional — levantar también el flujo real (`eliox-platform-config`) en el
mismo `argocd-it4t`:**
```bash
kubectl --context kind-hub apply \
  -f ~/works/gitops-eliox/eliox-platform-config/argocd/applications/mc-user-fastapi-dev.yaml
kubectl --context kind-hub -n argocd-it4t get applications   # debe verse mc-user-fastapi-dev Y mc-user-fastapi-platform-dev
kubectl --context kind-hub -n apps get pods
```
`eliox-platform-config` es público, así que no hace falta registrar
credenciales de git para esto.

**Apagar todo**
```bash
./down.sh
```
⚠️ Esto borra **todos** los clusters kind (`hub` + spokes activos) y
`.generated/` — se va ArgoCD, Artifactory y todo su contenido. No hay
"apagar solo una parte"; si solo quieres liberar RAM temporalmente, para los
pods en vez de borrar los clusters.

**Parar/arrancar los runners self-hosted** (no hace falta borrar nada para
esto — parar un runner no lo desregistra de GitHub, solo deja de escuchar
jobs; si algo dispara un workflow mientras está parado, el job queda
"Queued" hasta que lo vuelvas a arrancar, no se pierde):
```bash
# parar (p.ej. para no consumir mientras no trabajas en esto)
sudo systemctl stop actions.runner.elioxrome-generic-charts-ms.laptop-eliezer-romero.service
sudo systemctl stop actions.runner.elioxrome-eliox-platform-config.laptop-eliezer-romero-eliox-platform-config.service

# arrancar de nuevo
sudo systemctl start actions.runner.elioxrome-generic-charts-ms.laptop-eliezer-romero.service
sudo systemctl start actions.runner.elioxrome-eliox-platform-config.laptop-eliezer-romero-eliox-platform-config.service

# ver estado de los tres (incluye el huerfano de mc-user-fastapi)
systemctl list-units --type=service --all | grep actions.runner
```
Los runners en sí son livianos estando parados — lo que de verdad pesa en
RAM son los clusters kind y Artifactory, que siguen corriendo aunque pares
los runners (pararlos no afecta esos).

Para **desregistrar por completo** uno que ya no se usa (ej. el huérfano de
`mc-user-fastapi`, ver "Pendiente de limpiar" en automatización del lab):
```bash
cd ~/actions-runners/<repo>
sudo ./svc.sh stop
sudo ./svc.sh uninstall
./config.sh remove --token <token-de-baja>   # gh api -X POST repos/elioxrome/<repo>/actions/runners/remove-token --jq .token
cd .. && rm -rf <repo>
```

## Decisiones y gotchas ya resueltos (no los vuelvas a investigar)

- Los CRDs de ArgoCD (`applications.argoproj.io`, etc.) son cluster-scoped:
  solo una instancia Helm puede tener `crds.install: true` (la tiene `it4t`;
  las demás van con `false`) o falla por conflicto de ownership.
- ArgoCD multi-source **no soporta** un chart Helm como fuente `ref` para
  `$values` (solo repos git sirven como `ref`). Por eso el lab usa un
  "release chart" con dependencia Helm vendorizada en vez de separar chart y
  values en dos fuentes.
- El vendoring del chart genérico dentro del release chart (`publish-release-chart.sh`)
  **no es evitable publicando sin vendorizar y dejando que ArgoCD resuelva la
  dependencia al sincronizar** — probado en vivo (2026-10-05) y confirmado que
  falla antes de eso: `helm package` se niega en seco a empaquetar un chart
  con una dependencia declarada en `Chart.yaml` que no esté en `charts/`
  (`Error: found in Chart.yaml, but missing in charts/ directory: <nombre>`).
  Es un requisito duro del CLI de Helm al empaquetar, no una limitación de
  ArgoCD ni de la URL in-cluster de Artifactory — no lo vuelvas a intentar.
- ArgoCD cachea el chart de un `Application`/`ApplicationSet` por
  `targetRevision` (versión del chart). Si se republica contenido nuevo bajo
  la **misma** versión (p.ej. `mc-user-fastapi-0.1.0-dev` apuntando a otro
  tag de imagen), ArgoCD no lo nota solo — hay que forzar
  `kubectl annotate application <nombre> argocd.argoproj.io/refresh=hard --overwrite`
  (así lo hace `deploy-lab.yml`). Bump de versión en cada publish evitaría
  esto, pero el `ApplicationSet` del lab usa `targetRevision` fijo
  (`0.1.0-{{.env}}`), así que por ahora se resuelve con el refresh forzado.
- Un runner self-hosted **no puede usar `sudo`** dentro de un step de
  workflow (no hay terminal/askpass, falla con "a terminal is required").
  Cualquier instalación de herramientas en un job que corra en
  `[self-hosted, kind-local]` tiene que ir a una ruta sin privilegios
  (`$HOME/.local/bin` + `$GITHUB_PATH`), nunca `/usr/local/bin` por `sudo`.
- Esta laptop ya tenía un `/usr/bin/yq` instalado (el wrapper en Python de
  `jq` para YAML) que **no** es compatible con la sintaxis `yq -i '.path = x'`
  que usan `main.yml`/`deploy-lab.yml` (esa es la del binario de Go de
  mikefarah/yq). En un runner self-hosted hay que instalar el yq correcto en
  una ruta que quede *antes* en el `PATH` (ver punto anterior) — en
  GitHub-hosted (`ubuntu-latest`) no pasa, ahí no viene preinstalado ningún
  `yq` conflictivo.
- Artifactory OSS no permite crear repos nuevos vía la API de administración
  (ese endpoint es Pro-only, devuelve 400 "available only in Artifactory
  Pro") — el lab reutiliza el repo genérico por defecto `example-repo-local`
  bajo la ruta `charts-values`.
- Artifactory 7.161.x exige PostgreSQL (Derby ya no es soportado) y un
  Secret de Kubernetes con master-key + join-key (`global.masterKeySecretName`
  / `joinKeySecretName`), si no, los pods crashloopean.
- Un runner de GitHub Actions self-hosted en este laptop es obligatorio para
  que el pipeline de `generic-charts-ms` alcance Artifactory dentro del kind
  cluster vía `kubectl port-forward` — un runner hosteado por GitHub no tiene
  esa red.
- `nameOverride` es obligatorio al consumir `generic-charts-ms`: si se omite,
  todos los labels `app.kubernetes.io/name` quedan como `microservice` en vez
  del nombre real de la app.
- El `Jenkinsfile` de `mc-user-fastapi` fija
  `@Library('eliox-jenkins-shared-library@develop')`: el clon activo de la
  librería (`~/works/sabadell/eliox-jenkins-shared-library`) vive en la
  branch `develop`, no `main` — tenlo en cuenta antes de asumir que los
  cambios deben ir a `main` ahí.
- `eliox-platform-config` trae, además del chart-por-servicio, scaffolding de
  calidad/compliance que no es específico de `mc-user-fastapi`: workflows
  `helm-ci.yml`/`security.yml`/`release.yml`, policies de Conftest/OPA en
  `policies/conftest/`, scripts `scripts/{validate,package,promote}.sh` y
  ADRs/runbooks bajo `docs/`. El `README.md` de ese repo menciona un chart
  `bs-fastapi-repo` en su diagrama de estructura que no existe en el repo
  (solo está `mc-user-fastapi` bajo `charts/`); es un desfase de la doc de
  ese repo, no del mapa.

## Convenciones

- **Sin** trailer `Co-Authored-By: Claude` en los commits de los repos
  personales/de laboratorio de Eliezer (`mc-user-fastapi`, `argocd-kind-lab`,
  `generic-charts-ms`, y por extensión este propio repo harness). Si en algún
  momento se trabaja en un repo claramente de equipo/empresa, preguntar antes
  de asumir la misma preferencia.
- `argocd-kind-lab` y `generic-charts-ms` son públicos en GitHub bajo
  `elioxrome` y pensados para mostrar/practicar el patrón. `eliox-platform-config`,
  `mc-user-fastapi` y `eliox-jenkins-shared-library` son el código real de la
  empresa/proyecto — tratarlos con más cuidado (no forzar pushes, no inventar
  branches nuevas sin que el usuario lo pida).
- Nunca borrar bases de datos ni volúmenes de otros sistemas que corran en el
  mismo Docker/kind del laptop al experimentar con Artifactory u otra pieza
  nueva — instrucción explícita del usuario.

## Pendientes conocidos (al momento de escribir esto)

- El `README.md` de `argocd-kind-lab` todavía no documenta Artifactory ni el
  pipeline de publicación de `generic-charts-ms`.
- `stg`/`prd` están comentados en el `ApplicationSet` de `mc-user-fastapi`
  hasta que existan las imágenes `1.0.0-rc` / `1.0.0` en Docker Hub.
- `.generated/artifactory-admin.env` en `argocd-kind-lab` no se genera solo
  (hay que crearlo a mano, ver "Cómo levantar el laboratorio"); si ya existe
  de una sesión anterior, puede tener un password viejo de Artifactory
  (devuelve 401 al publicar) — no asumas que coincide con el actual sin
  confirmarlo. El secret `ARTIFACTORY_PASSWORD` en GitHub (`generic-charts-ms`)
  es otra copia independiente, actualizada y funcionando en el pipeline de CI.
- `~/works/devops/kind_cluster/mc-user-fastapi` y su `eliox-platform-config`
  son clones desactualizados (mayo/feb 2026) de un prototipo anterior de un
  solo cluster (`local-cluster`, ver `kind-cluster.yaml`/`user.yaml` en esa
  carpeta); no son la fuente de verdad, úsalos solo como referencia histórica.
- `~/works/gitops-eliox-workflows` es un clon duplicado de `devops-workflows`
  (sin cambios sin pushear); el clon activo con el que se trabaja es
  `~/works/devops-workflows`, que puede ir 1+ commits adelante de `origin`.

## Cómo usar este repo con Claude Code

Si vas a pedirle a Claude que haga algo sobre cualquiera de los repos de este
mapa, abre (o pégale el contenido de) este `CLAUDE.md` primero, o menciona
"mira el harness" / pasa la ruta `~/works/gitops-eliox/eliox-gitops-harness`. Mantenlo
actualizado cuando algo del mapa cambie (repo nuevo, flujo que se reemplaza,
gotcha nuevo) en vez de dejar que cada sesión lo redescubra.
