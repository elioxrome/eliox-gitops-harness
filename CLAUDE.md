# Contexto: ecosistema GitOps de Eliezer

Este repo no tiene código propio que correr. Es el mapa de referencia entre los
repos reales y de laboratorio que conforman un mismo flujo GitOps
(Bitbucket/GitHub → Jenkins/GitHub Actions → eliox-platform-config → ArgoCD →
clusters). Léelo antes de tocar cualquiera de los repos listados abajo si no
tienes ya este contexto cargado en la conversación.

Hay **dos flujos que coexisten y no deben mezclarse**: el flujo real de la
empresa (parcialmente reproducido en local) y el laboratorio nuevo
(`argocd-kind-lab`). Ver la sección "Dos flujos" más abajo antes de asumir cuál
aplica a lo que te estén pidiendo.

## Mapa de repos

| Repo | Rol | Ubicación local | Remoto |
|---|---|---|---|
| **mc-user-fastapi** | Microservicio real (FastAPI). Tiene su propio `Jenkinsfile` (usa `eliox-jenkins-shared-library`) y un workflow de GitHub Actions que build+push la imagen y abre PR a `eliox-platform-config`. | `~/Escritorio/mc-user-fastapi` (clon activo, el que se edita) · `~/works/devops/kind_cluster/mc-user-fastapi` (clon viejo de mayo 2026, prototipo de un solo cluster `local-cluster`, no editar) | `github.com/elioxrome/mc-user-fastapi` |
| **eliox-platform-config** | Config real de la plataforma: un chart Helm por microservicio bajo `charts/`, values por entorno bajo `environments/<env>/`, `Applications` de ArgoCD bajo `argocd/applications/`. | `~/works/devops/kind_cluster/eliox-platform-config` | `github.com/elioxrome/eliox-platform-config` (branches: `main`, `gitops`, `feature/argocd`, PRs automáticos `pr-gitops-dev-*`) |
| **eliox-jenkins-shared-library** | Librería compartida de Jenkins (`vars/enterprisePipeline.groovy`) que consumen los `Jenkinsfile` de cada microservicio vía `@Library('eliox-jenkins-shared-library@...')`. | `~/works/devops/kind_cluster/eliox-jenkins-shared-library` · `~/works/sabadell/eliox-jenkins-shared-library` | `github.com/elioxrome/eliox-jenkins-shared-library` |
| **devops-workflows** | Workflows reutilizables de GitHub Actions (CI build+push a GHCR, CD a VPS por Docker Compose) para otros proyectos de Eliezer. No forma parte directa del flujo de mc-user-fastapi, pero sigue la misma filosofía "build once, deploy many". | `~/works/devops-workflows` | `github.com/elioxrome/devops-workflows` |
| **generic-charts-ms** | Chart Helm genérico (`microservice`) para cualquier microservicio stateless: Deployment+Service+HPA+PDB+NetworkPolicy+Ingress opcional. Reemplaza, en el laboratorio, al chart por-servicio de `eliox-platform-config`. Tiene su propio pipeline de GitHub Actions que publica cada versión a Artifactory usando un runner self-hosted en este laptop. | `~/Escritorio/microservice-chart` | `github.com/elioxrome/generic-charts-ms` |
| **argocd-kind-lab** | Laboratorio local: kind multi-cluster (`hub` + un spoke por "tipo") + una instancia de ArgoCD por Helm por tipo + Artifactory, todo como IaC versionado. Reproduce la arquitectura real de la empresa (5 instancias ArgoCD por tipo: IT4T/DIGITAL/EOP/PSC/BCC) reducida a `it4t`+`digital` para no gastar RAM. | `~/Escritorio/kind_cluster_test` | `github.com/elioxrome/argocd-kind-lab` |

## Dos flujos — no mezclar

**1. Flujo real de la empresa (parcialmente reproducido):**
`mc-user-fastapi` → build (Jenkins vía `eliox-jenkins-shared-library`, o el
workflow de GitHub Actions `main.yml`) → push de imagen a Docker Hub → PR
automático a `eliox-platform-config` (`environments/dev/mc-user-fastapi-values.yaml`,
branch `gitops`) → merge → ArgoCD aplica. El chart vive **dentro** de
`eliox-platform-config` (`charts/mc-user-fastapi`), un chart por servicio.

**2. Flujo del laboratorio (`argocd-kind-lab`):**
kind con un cluster `hub` + spokes por tipo (hoy solo `it4t` + `digital`
activos; `eop`/`psc`/`bcc` desactivados en `kind/delete/` para ahorrar RAM) +
una instancia ArgoCD Helm por tipo en el hub + Artifactory local (IaC en
`artifactory/`). El chart **ya no es específico de mc-user-fastapi**: es
`generic-charts-ms`, publicado por su propio pipeline de GitHub Actions
(runner self-hosted, label `kind-local`, instalado como systemd en este
laptop: `actions.runner.elioxrome-generic-charts-ms.laptop-eliezer-romero.service`)
a Artifactory. Cada microservicio solo aporta un `values.yaml` de pocas
líneas; se empaqueta como "release chart" (`artifactory/publish-release-chart.sh`)
que trae el chart genérico vendorizado como dependencia Helm real (no copia de
templates). El `ApplicationSet` de ArgoCD en el lab
(`argocd/applicationsets/mc-user-fastapi-appset.yaml`) apunta a ese release
chart en Artifactory, no a `eliox-platform-config`.

## Decisiones y gotchas ya resueltos (no los vuelvas a investigar)

- Los CRDs de ArgoCD (`applications.argoproj.io`, etc.) son cluster-scoped:
  solo una instancia Helm puede tener `crds.install: true` (la tiene `it4t`;
  las demás van con `false`) o falla por conflicto de ownership.
- ArgoCD multi-source **no soporta** un chart Helm como fuente `ref` para
  `$values` (solo repos git sirven como `ref`). Por eso el lab usa un
  "release chart" con dependencia Helm vendorizada en vez de separar chart y
  values en dos fuentes.
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
- `.generated/artifactory-admin.env` en `argocd-kind-lab` tiene el password
  viejo de Artifactory (devuelve 401 si se usa para probar algo a mano); el
  secret `ARTIFACTORY_PASSWORD` en GitHub (`generic-charts-ms`) ya está
  actualizado y el pipeline de CI funciona con el valor correcto.
- `~/works/devops/kind_cluster/mc-user-fastapi` y su `eliox-platform-config`
  son clones desactualizados (mayo/feb 2026) de un prototipo anterior de un
  solo cluster (`local-cluster`, ver `kind-cluster.yaml`/`user.yaml` en esa
  carpeta); no son la fuente de verdad, úsalos solo como referencia histórica.

## Cómo usar este repo con Claude Code

Si vas a pedirle a Claude que haga algo sobre cualquiera de los repos de este
mapa, abre (o pégale el contenido de) este `CLAUDE.md` primero, o menciona
"mira el harness" / pasa la ruta `~/Escritorio/eliox-gitops-harness`. Mantenlo
actualizado cuando algo del mapa cambie (repo nuevo, flujo que se reemplaza,
gotcha nuevo) en vez de dejar que cada sesión lo redescubra.
