# store-gitops

Repositorio DevOps de `store-app`: contiene los manifiestos de Kubernetes (Kustomize) y las Applications de Argo CD. **Argo CD vigila este repositorio** y sincroniza el clúster EKS con lo que hay en la rama `main`.

```
Push a store-app (main) → GitHub Actions: test, build y push a ECR (tag = SHA corto)
  → la CI hace "kustomize edit set image" en este repo y commit → Argo CD detecta el cambio
  → sincroniza el clúster → los nodos hacen pull de la imagen desde ECR
```

## Estructura

```
apps/
├── frontend/            Deployment (2 réplicas) + Service LoadBalancer (80 → 8080)
├── products-service/    Deployment + Service ClusterIP (8000)
├── orders-service/      Deployment + Service ClusterIP (8080)
├── products-db/         StatefulSet PostgreSQL + PVC gp3 + Service headless
└── orders-db/           StatefulSet PostgreSQL + PVC gp3 + Service headless
argocd/
├── root-app.yaml        Application raíz (app-of-apps): se aplica a mano una sola vez
└── apps/                Una Application por componente (apps/<nombre>)
```

Todo vive en el namespace `store`. Cada componente tiene su `kustomization.yaml`; se puede revisar sin clúster con `kubectl kustomize apps/<nombre>`.

- **Probes:** los tres Deployments usan `/health` (readiness y liveness). Las bases usan `pg_isready`.
- **`APP_VERSION`:** siempre es igual al tag de la imagen. Kustomize lo copia del campo `image` (bloque `replacements`), así que no se edita a mano y el pie de página de la aplicación muestra la versión desplegada.
- **Imágenes de marcador:** mientras la CI no haya corrido, las imágenes son `000000000000.dkr.ecr.us-east-1.amazonaws.com/<nombre>:0000000`. Los pods quedarán en `ImagePullBackOff` hasta la primera ejecución de la CI; es lo esperado.
- **Réplicas:** el frontend tiene 2 (para ver el rolling update); los servicios 1, porque dos instancias creando a la vez la tabla y los datos de ejemplo en el primer arranque podrían chocar.
- **Cada servicio usa solo su base de datos** (database-per-service).

## Requisitos previos

La infraestructura de `store-app/infra/terraform` ya aplicada (clúster, StorageClass `gp3`, Argo CD) y `kubectl` apuntando al clúster:

```bash
aws eks update-kubeconfig --name store-eks --region us-east-1
kubectl get nodes
kubectl get storageclass          # gp3 debe aparecer como (default)
kubectl get pods -n argocd
```

## Despliegue paso a paso

### 1. Namespace y Secrets de las bases de datos

Las contraseñas **no están en git**: se generan en el momento y viven solo en el clúster. Los Secrets deben existir antes de la `root-app`.

```bash
kubectl create namespace store

# Contraseñas aleatorias (hexadecimal: seguras para usar dentro de una URL)
PRODUCTS_DB_PASSWORD=$(openssl rand -hex 16)
ORDERS_DB_PASSWORD=$(openssl rand -hex 16)

kubectl create secret generic products-db-credentials -n store \
  --from-literal=POSTGRES_PASSWORD="$PRODUCTS_DB_PASSWORD" \
  --from-literal=DATABASE_URL="postgresql+psycopg://products:${PRODUCTS_DB_PASSWORD}@products-db:5432/products"

kubectl create secret generic orders-db-credentials -n store \
  --from-literal=POSTGRES_PASSWORD="$ORDERS_DB_PASSWORD" \
  --from-literal=DATABASE_URL="postgres://orders:${ORDERS_DB_PASSWORD}@orders-db:5432/orders?sslmode=disable"

unset PRODUCTS_DB_PASSWORD ORDERS_DB_PASSWORD
```

Cada Secret lleva `POSTGRES_PASSWORD` (lo lee la base) y `DATABASE_URL` (la lee el servicio). Si cambias la contraseña de un Secret **después** de que la base se haya inicializado, PostgreSQL no la actualiza: habría que borrar el PVC de esa base (se pierden sus datos).

### 2. Registrar el repositorio en Argo CD

- **Repositorio público:** Argo CD lo lee sin credenciales. Puedes registrarlo igualmente desde la UI (*Settings → Repositories → Connect Repo*, método HTTPS, URL `https://github.com/J0sePs/store-gitops.git`).
- **Repositorio privado:** crea un token de GitHub de solo lectura (*Contents: read*) y regístralo con un Secret. El token no se guarda en git:

  ```bash
  read -rs GITHUB_TOKEN     # pega el token y pulsa Enter (no se muestra)
  kubectl create secret generic store-gitops-repo -n argocd \
    --from-literal=type=git \
    --from-literal=url=https://github.com/J0sePs/store-gitops.git \
    --from-literal=username=J0sePs \
    --from-literal=password="$GITHUB_TOKEN"
  kubectl label secret store-gitops-repo -n argocd argocd.argoproj.io/secret-type=repository
  unset GITHUB_TOKEN
  ```

Si tu repositorio tiene otra URL, cámbiala en `argocd/root-app.yaml` y en `argocd/apps/*.yaml` (6 archivos):

```bash
sed -i 's#https://github.com/J0sePs/store-gitops.git#https://github.com/<USUARIO>/store-gitops.git#' argocd/root-app.yaml argocd/apps/*.yaml
```

### 3. Aplicar la Application raíz (una sola vez)

Antes, sube este repositorio a GitHub (rama `main`).

```bash
kubectl apply -f argocd/root-app.yaml
```

La raíz crea las cinco Applications y estas sincronizan sus carpetas `apps/<nombre>`. A partir de aquí todo se gestiona desde git.

### 4. Comprobar

```bash
kubectl get applications -n argocd            # Sync: Synced / Health: Healthy
kubectl get pods -n store -w                  # bases en Running; los servicios, tras la primera imagen de la CI
kubectl get svc frontend -n store             # EXTERNAL-IP (hostname del balanceador; tarda unos minutos)

URL=$(kubectl get svc frontend -n store -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl "http://$URL/health"                     # {"status":"ok","version":"<tag>"}
curl "http://$URL/api/products/health"
curl "http://$URL/api/orders/health"
```

Abre `http://<hostname del balanceador>` en el navegador: el pie de página muestra la versión del frontend y la de cada servicio.

## Interfaz de Argo CD

```bash
# Contraseña inicial del usuario admin (no está en git ni en las salidas de Terraform)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo

# Abrir la interfaz en https://localhost:8080 (usuario: admin)
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

El navegador avisará del certificado autofirmado: es normal. Argo CD no se expone con un balanceador (ahorra costo).

## Cómo actualiza la CI las imágenes

Después de subir una imagen a ECR, la CI ejecuta dentro de `apps/<nombre>` (donde `NAME` es `frontend`, `products-service` u `orders-service`, `ECR_REGISTRY` es `<cuenta>.dkr.ecr.us-east-1.amazonaws.com` y `SHORT_SHA` el SHA corto del commit):

```bash
cd "apps/$NAME"
kustomize edit set image "$NAME=$ECR_REGISTRY/$NAME:$SHORT_SHA"
```

y hace commit. Solo cambia el bloque `images` del `kustomization.yaml`; `APP_VERSION` se actualiza solo. Reglas:

- El tag es el **SHA corto** del commit. La etiqueta `latest` está prohibida.
- Se usa `kustomize edit`, no `sed`: entiende tags que parecen números (los entrecomilla). Además reformatea el archivo y elimina sus comentarios tras la primera ejecución: es normal.
- La CI **nunca** ejecuta `kubectl apply`: el despliegue lo hace Argo CD.

## Operación

- **Volver a una versión anterior:** `git revert` del commit de la CI y push; Argo CD sincroniza solo.
- **Forzar una sincronización:** desde la UI (*Sync*) o `kubectl -n argocd annotate application <nombre> argocd.argoproj.io/refresh=hard --overwrite`.
- **Cambios manuales en el clúster:** `selfHeal` los revierte al estado de git. Para cambiar algo, cámbialo aquí.
- **Versión desplegada:** `curl http://<balanceador>/api/products/health` (y `/orders`).
- Los volúmenes EBS están atados a una zona de disponibilidad: cada base solo puede reprogramarse en nodos de su zona.

## Limpieza antes de `terraform destroy`

Los balanceadores y los volúmenes EBS los crea Kubernetes fuera de Terraform. Si se quedan, el `destroy` de la VPC falla y siguen cobrando.

```bash
# 1. Borrar las Applications (el finalizador elimina sus recursos, incluido el Service LoadBalancer)
kubectl delete applications --all -n argocd

# 2. Los PVC de las bases no los borra Argo CD: bórralos tú
kubectl delete pvc --all -n store

# 3. Comprobar (solo lectura) que no queda nada
kubectl get svc,pvc -n store
aws elb describe-load-balancers --region us-east-1 --query 'LoadBalancerDescriptions[].LoadBalancerName'
aws ec2 describe-volumes --region us-east-1 --filters Name=status,Values=available --query 'Volumes[].VolumeId'
```

Después sigue los pasos de `destroy` de `store-app/infra/terraform/README.md`.

## Seguridad

- Este repositorio **no contiene secretos**: ni contraseñas de bases de datos, ni tokens, ni credenciales de AWS. El `.gitignore` ignora `secret.yaml`, `*.secret.yaml`, `.env` y similares.
- Los contenedores corren sin root, sin escalada de privilegios y sin capabilities.
- La CI usa OIDC para entrar a AWS (sin llaves estáticas).
