# eliox-gitops-harness

Mapa de referencia entre los repos reales y de laboratorio que forman el
mismo flujo GitOps (microservicio → build → config de plataforma → ArgoCD →
cluster). No tiene código que correr: es contexto para que cualquier sesión
de Claude Code (o cualquier persona nueva en esto) entienda de una vez cómo
se conectan `mc-user-fastapi`, `eliox-platform-config`,
`eliox-jenkins-shared-library`, `generic-charts-ms` y `argocd-kind-lab`, sin
tener que redescubrirlo cada vez.

El contenido real vive en [CLAUDE.md](./CLAUDE.md): tabla de repos (rol,
ubicación local, remoto), los dos flujos que coexisten (el real de la
empresa y el del laboratorio nuevo), los gotchas ya resueltos (CRDs de
ArgoCD, límite de multi-source con charts Helm, licencia de Artifactory OSS,
runner self-hosted, etc.) y los pendientes conocidos.

## Uso

Antes de pedirle a Claude algo sobre cualquiera de esos repos, dale el
contexto de este harness (abrir el repo, pasar la ruta, o decir "mira el
harness"). Cuando algo del ecosistema cambie — un repo nuevo, un flujo que se
reemplaza, un gotcha nuevo — actualiza `CLAUDE.md` en el mismo cambio, en vez
de dejarlo desincronizado.
