# otsitohub-api en el lab de Kubernetes

Despliegue del API headless de Otsitohub (FastAPI + PostgreSQL + Redis) en el
laboratorio personal, gestionado por el Argo CD del lab desde este repo.

Es una copia adaptada de los manifiestos de `gitea_admin/gitops-apps`
(`apps/otsitohub-api/`), que apuntan al cluster compartido `talos-prod`. Las
diferencias respecto de aquellos:

| Aspecto | gitops-apps (talos-prod) | Este repo (lab) |
|---|---|---|
| Imagen | `gitea-teotl.amsiops.io/amsidev/otsitohub-api` (privada) | `ghcr.io/alesdev17/otsitohub-api:lab` (publica) |
| Pull secret | `gitea-registry-credentials` | ninguno |
| Exposicion | Service ClusterIP + ingress del cluster | Service NodePort `30080` |
| StorageClass | `local-path` fijo | la default del cluster |
| PVC postgres | 10Gi | 5Gi |
| Overlays | dev / staging / production | solo `lab` |

## Estructura

```
apps/otsitohub-api/
  base/            manifiestos comunes (namespace, postgres, redis, api, service)
  overlays/lab/    ajustes del lab (imagen GHCR, NodePort, 5Gi, ENVIRONMENT=lab)
argocd/otsitohub-api-application.yaml   Application de Argo CD
```

## Requisitos previos

1. **Imagen publicada** en `ghcr.io/alesdev17/otsitohub-api:lab` y el paquete
   marcado como publico en GitHub (si es privado, el pull falla con
   `ImagePullBackOff` porque el overlay no define `imagePullSecrets`).

   Desde `~/Dev/OtsitoHub/otsitohub-api`:

   ```bash
   echo "$GHCR_PAT" | docker login ghcr.io -u AlesDev17 --password-stdin
   docker build --platform linux/amd64 -t ghcr.io/alesdev17/otsitohub-api:lab .
   docker push ghcr.io/alesdev17/otsitohub-api:lab
   ```

   `--platform linux/amd64` es obligatorio si construyes desde un Mac con Apple
   Silicon y los nodos del lab son x86: sin eso el pod arranca y muere con
   `exec format error`.

2. **Secrets** en el namespace `otsitohub`. No estan en git y hay que crearlos
   antes del primer sync, porque `otsitohub-api-secrets` esta marcado
   `optional: false` y sin el los pods se quedan en `CreateContainerConfigError`:

   ```bash
   kubectl create namespace otsitohub --dry-run=client -o yaml | kubectl apply -f -

   kubectl create secret generic postgres-credentials -n otsitohub \
     --from-literal=POSTGRES_PASSWORD="$(openssl rand -base64 24)"

   kubectl create secret generic otsitohub-api-secrets -n otsitohub \
     --from-literal=JWT_SECRET="$(openssl rand -base64 48)"
   ```

   `ENVIRONMENT=lab` (distinto de `development`) activa el guard de
   `config.py`: si `JWT_SECRET` sigue siendo el default de desarrollo, la app se
   niega a arrancar. Es intencional.

## Despliegue

```bash
export KUBECONFIG=~/.kube/jorge-lab.kubeconfig

# Render local, sin tocar el cluster
kubectl kustomize apps/otsitohub-api/overlays/lab

# Registrar la Application (una sola vez)
kubectl apply -f argocd/otsitohub-api-application.yaml
```

La Application tiene `automated: {prune: true, selfHeal: true}`: a partir de
aqui, cada push a `main` de este repo se sincroniza solo.

## Verificacion

```bash
kubectl -n otsitohub get pods,svc,pvc
kubectl -n otsitohub logs deploy/otsitohub-api -c run-migrations   # alembic upgrade head
curl http://<ip-de-cualquier-nodo>:30080/health
```

`/health` debe responder 200 con `environment: lab`. El NodePort responde en
cualquier nodo del cluster, no solo en el que corre el pod.

## Notas operativas

- **Sin default StorageClass**: si `kubectl get sc` no muestra ninguna marcada
  `(default)`, el PVC de postgres queda `Pending`. Descomenta el bloque
  `storageClassName` en `overlays/lab/patches/postgres-storage.yaml`.
- **volumeClaimTemplates es inmutable**: cambiar el tamano del PVC despues del
  primer despliegue exige borrar el StatefulSet con
  `kubectl delete sts postgres -n otsitohub --cascade=orphan` y volver a
  sincronizar.
- **Actualizar la imagen**: el tag `lab` es mutable y el pull es `Always`, asi
  que basta con `docker push` y `kubectl -n otsitohub rollout restart deploy/otsitohub-api`.
- **Este despliegue no sustituye a la tienda**: el modelo operativo de la caja
  sigue siendo docker compose en LAN. El lab es para validar el flujo GitOps y
  el acceso remoto.
