# Pipeline CI/CD con GitHub Actions, Ansible y GCP

## Despliegue Automatizado de Crypto Chatbot (Frontend + Backend)

---

## 1. Arquitectura General

### Flujo del Pipeline

```
Developer Push (main)
        ↓
GitHub Actions
  ├─ Build imagen backend (Python)
  ├─ Build imagen frontend (Nginx)
  ├─ Push ambas imágenes → Artifact Registry
  └─ SSH → Control Node
              ↓
        Control Node (GCP VM) 
          └─ ansible-playbook deploy.yml
                    ↓
              VM de Despliegue
                ├─ agent-backend  (puerto 80:80)
                └─ agent-frontend (puerto 80:80)
```


### Componentes Involucrados

| Componente | Rol | Ubicación |
|---|---|---|
| GitHub Actions | CI — build y push + SSH al control | Cloud (GitHub) | 
| Artifact Registry | Registro de imágenes Docker | GCP (`us-central1`) |
| Control Node | Ejecuta Ansible | GCP VM |
| VM de Despliegue | Hospeda los contenedores | GCP VM |

---

## 2. Prerrequisitos

Antes de comenzar, asegúrate de tener lo siguiente:

- Cuenta de GCP con un proyecto activo y facturación habilitada.
- `gcloud` CLI instalado y configurado localmente.
- Repositorio en GitHub con el código del chatbot (frontend + backend).
- Control Node en GCP ya configurado con Ansible, Docker y acceso SSH a la VM de despliegue.
- La VM de despliegue con Docker instalado y accesible desde el Control Node.

Configura las variables de entorno para los comandos que siguen:

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1
```

---

## 3. Paso 1 — Service Account de CI en GCP

Creamos una service account dedicada exclusivamente para GitHub Actions. Esta cuenta se usará para autenticarse contra GCP y subir las imágenes Docker al Artifact Registry.

### Crear la Service Account

```bash
gcloud iam service-accounts create github-ci \
  --display-name="GitHub Actions CI"
```

### Asignar Rol de Escritura en Artifact Registry

```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:github-ci@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"
```

### Generar Clave JSON

```bash
gcloud iam service-accounts keys create github-ci-key.json \
  --iam-account=github-ci@${PROJECT_ID}.iam.gserviceaccount.com
```

> ⚠️ **Importante:** Guarda este archivo JSON de forma segura. Lo necesitarás para configurar los secretos de GitHub. No lo subas al repositorio.

---

## 4. Paso 2 — Artifact Registry

Crear el repositorio Docker en Artifact Registry (una sola vez):

```bash
gcloud artifacts repositories create agent-repo \
  --repository-format=docker \
  --location=$REGION \
  --description="Repositorio de imágenes del Crypto Chatbot"
```

La ruta del registro será:

```
us-central1-docker.pkg.dev/PROJECT_ID/agent-repo
```

Verificar que se ha creado correctamente:

```bash
gcloud artifacts repositories list --location=$REGION
```
Lo hacemos público (para no tener que crear service accounts en los nodos clientes en esta práctica)

```bash
gcloud artifacts repositories add-iam-policy-binding agent-repo \
  --location=us-central1 \
  --member="allUsers" \
  --role="roles/artifactregistry.reader"
```

---

## 5. Paso 3 — Permisos del Control Node para Artifact Registry

El Control Node necesita poder **descargar (pull)** las imágenes desde Artifact Registry para desplegarlas en la VM final. Para ello, la service account `ansible-sa` del Control Node necesita el rol de **lector** del registro.

### Roles Existentes de `ansible-sa`

La service account ya tiene asignados los siguientes roles para gestión de infraestructura:

```bash
for role in \
  roles/compute.instanceAdmin.v1 \
  roles/compute.networkAdmin \
  roles/compute.securityAdmin \
  roles/iam.serviceAccountUser; do
  gcloud projects add-iam-policy-binding "$PROJECT_ID" \
    --member="serviceAccount:ansible-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="$role"
done
```

### Añadir Rol de Lectura en Artifact Registry

Añadimos el rol necesario para pull de imágenes:

```bash
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:ansible-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.reader"
```

> 💡 **Tip:** Si prefieres mantener la consistencia, puedes añadirlo al bloque `for` anterior para tener todos los roles documentados en un solo lugar:
>
> ```bash
> for role in \
>   roles/compute.instanceAdmin.v1 \
>   roles/compute.networkAdmin \
>   roles/compute.securityAdmin \
>   roles/iam.serviceAccountUser \
>   roles/artifactregistry.reader; do
>   gcloud projects add-iam-policy-binding "$PROJECT_ID" \
>     --member="serviceAccount:ansible-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
>     --role="$role"
> done
> ```

### Configurar Docker en el Control Node para Autenticarse

Conéctate al Control Node por SSH y ejecuta:

```bash
gcloud auth configure-docker us-central1-docker.pkg.dev
```

Esto configura Docker para usar `gcloud` como credential helper al hacer pull desde Artifact Registry.

### Verificar el Acceso

Desde el Control Node, prueba que puede acceder al registro:

```bash
docker pull us-central1-docker.pkg.dev/$PROJECT_ID/agent-repo/agent-backend:latest
```

> 💡 **Nota:** Si la VM de despliegue es distinta del Control Node, Ansible se encargará de autenticarse con Docker en la VM remota usando la clave JSON de la service account (ver Paso 9).

---



## 6. Paso 6 — Acceso SSH desde GitHub Actions al Control Node

Usamos el modelo de **trigger por SSH**: GitHub Actions, tras construir y subir las imágenes, se conecta por SSH al Control Node para ejecutar Ansible.

### Generar Par de Claves en el Control Node

```bash
ssh-keygen -t ed25519 -f ~/.ssh/github-deploy -C "github-actions-deploy"
```

Esto genera:

- `~/.ssh/github-deploy` — clave privada (irá a GitHub Secrets)
- `~/.ssh/github-deploy.pub` — clave pública (se queda en el Control Node)

### Autorizar la Clave Pública

En el Control Node, añadir la clave pública al archivo de claves autorizadas:

```bash
cat ~/.ssh/github-deploy.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### Verificar la Configuración SSH

Asegúrate de que el servicio SSH está activo y que el firewall de GCP permite el tráfico en el puerto 22 (o el puerto que uses):

```bash
sudo systemctl status sshd
```

---

## 7. Paso 7 — Secretos en GitHub

Ve a tu repositorio en GitHub → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**.

Configura los siguientes secretos:

| Secreto | Descripción | Valor |
|---|---|---|
| `GCP_SA_KEY` | JSON completo de `github-ci-key.json` | Contenido del archivo JSON |
| `PROJECT_ID` | ID del proyecto de GCP | `tu-project-id` |
| `GOOGLE_API_KEY` | Clave de API de Google (Gemini) | `AIza...` |
| `CONTROL_HOST` | IP pública o hostname del Control Node | `34.xxx.xxx.xxx` |
| `CONTROL_HOST_USER` | Usuario SSH del Control Node | `ansible` o tu usuario |
| `CONTROL_HOST_SSH_KEY` | Clave privada SSH (`github-deploy`) | Contenido de `~/.ssh/github-deploy` |
| `CONTROL_HOST_PORT` | Puerto SSH del Control Node | `22` |

> ⚠️ Para `CONTROL_HOST_SSH_KEY`, pega el contenido completo de la clave privada, incluyendo las líneas `-----BEGIN OPENSSH PRIVATE KEY-----` y `-----END OPENSSH PRIVATE KEY-----`.

---

## 8. Paso 8 — GitHub Actions Workflow

### `.github/workflows/build-deploy.yml`

```yaml
name: Build and Deploy Crypto Chatbot

on:
  push:
    branches: [main]

env:
  REGION: us-central1
  REGISTRY: us-central1-docker.pkg.dev
  REPO: agent-repo

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      # ─────────────────────────────────
      # Checkout del código
      # ─────────────────────────────────
      - name: Checkout repo
        uses: actions/checkout@v4

      # ─────────────────────────────────
      # Crear .env del backend dinámicamente
      # ─────────────────────────────────
      - name: Create backend .env
        run: |
          echo "GOOGLE_API_KEY=${{ secrets.GOOGLE_API_KEY }}" > backend/.env

      # ─────────────────────────────────
      # Autenticación en GCP
      # ─────────────────────────────────
      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          credentials_json: '${{ secrets.GCP_SA_KEY }}'

      - name: Configure Docker for Artifact Registry
        run: |
          gcloud auth configure-docker ${{ env.REGISTRY }}

      # ─────────────────────────────────
      # Build y Push — Backend (Python)
      # ─────────────────────────────────
      - name: Build backend image
        run: |
          docker build \
            -t ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-backend:${{ github.sha }} \
            -t ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-backend:latest \
            backend/

      - name: Push backend image
        run: |
          docker push ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-backend:${{ github.sha }}
          docker push ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-backend:latest

      # ─────────────────────────────────
      # Build y Push — Frontend (Nginx)
      # ─────────────────────────────────
      - name: Build frontend image
        run: |
          docker build \
            -t ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-frontend:${{ github.sha }} \
            -t ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-frontend:latest \
            frontend/

      - name: Push frontend image
        run: |
          docker push ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-frontend:${{ github.sha }}
          docker push ${{ env.REGISTRY }}/${{ secrets.PROJECT_ID }}/${{ env.REPO }}/agent-frontend:latest

      # ─────────────────────────────────
      # Deploy vía SSH al Control Node
      # ─────────────────────────────────
      - name: Deploy via Control Node
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.CONTROL_HOST }}
          username: ${{ secrets.CONTROL_HOST_USER }}
          key: ${{ secrets.CONTROL_HOST_SSH_KEY }}
          port: ${{ secrets.CONTROL_HOST_PORT }}
          script: |
            source ~/ansible-venv/bin/activate
            cd ~/ansible-gcp
            ansible-playbook -i inventory/gcp.yml \
              playbooks/deploy-bot.yml \
              --extra-vars "image_tag=${{ github.sha }}"
```

### Notas sobre el Workflow

- Se etiquetan las imágenes tanto con el SHA del commit (trazabilidad) como con `latest` (conveniencia).
- En producción se debería hacer un `git pull` en el Control Node asegurando que los playbooks y el inventario estén siempre actualizados. En esta práctica, al usar el nodo de control también como repositorio git para hacer push, no lo complicamos mas.
- Se usa `google-github-actions/auth@v2` (versión actualizada).

---

## 9. Paso 9 — Playbook de Ansible (Deploy)

Este playbook se ejecuta en el Control Node y despliega los contenedores en la VM de destino.

### `ansible/deploy-bot.yml`

```yaml
---
- name: Deploy Crypto Chatbot containers
  hosts: all
  become: true

  vars:
    registry_base: "us-central1-docker.pkg.dev/banded-pad-481109-q1/agent-repo"
    docker_network: "chatbot-network"

  tasks:

    - name: Crear red Docker
      community.docker.docker_network:
        name: "{{ docker_network }}"
        state: present

    - name: Desplegar backend
      community.docker.docker_container:
        name: agent-backend
        image: "{{ registry_base }}/agent-backend:{{ image_tag }}"
        state: started
        restart_policy: always
        pull: true
        recreate: true
        networks:
          - name: "{{ docker_network }}"
        ports:
          - "8000:80"

    - name: Desplegar frontend
      community.docker.docker_container:
        name: agent-frontend
        image: "{{ registry_base }}/agent-frontend:{{ image_tag }}"
        state: started
        restart_policy: always
        pull: true
        recreate: true
        networks:
          - name: "{{ docker_network }}"
        ports:
          - "80:80"

    - name: Verificar contenedores
      command: docker ps --format "{{ '{{' }}.Names{{ '}}' }} - {{ '{{' }}.Status{{ '}}' }} - {{ '{{' }}.Image{{ '}}' }}"
      register: docker_status

    - name: Mostrar estado
      debug:
        msg: "{{ docker_status.stdout_lines }}"
```

### Notas del Playbook

- Se usa `delegate_to: localhost` para leer la clave de SA desde el Control Node (no desde la VM remota).
- `recreate: true` fuerza la recreación del contenedor si la imagen ha cambiado.
- `pull: false` porque ya hemos hecho el pull explícito (más control sobre errores).
- Se incluye una tarea de **limpieza** para evitar acumulación de imágenes antiguas.
- La **verificación final** confirma que ambos contenedores están corriendo.

---



---

## 10. Paso 10 — Verificación y Troubleshooting

### Verificar el Pipeline Completo

**1. Verificar Artifact Registry (desde local o Cloud Shell):**

```bash
gcloud artifacts docker images list \
  $REGION-docker.pkg.dev/$PROJECT_ID/agent-repo
```


**2. Verificar contenedores en la VM de despliegue:**

```bash
# En la VM de despliegue
docker ps
docker logs agent-backend
docker logs agent-frontend
```

**3. Test funcional del chatbot:**

```bash
# Desde la VM
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"input": "What is the price of Bitcoin?"}'

# Desde fuera (IP pública de la VM)
 http://EXTERNAL_IP

```



------------------------------------------------------------------------------------------------------------------




### Problemas Comunes

| Problema | Causa Probable | Solución |
|---|---|---|
| `Permission denied` al hacer pull | SA sin rol `artifactregistry.reader` | Paso 3 — asignar rol |
| `docker login` falla en la VM | Docker no configurado con credential helper | Ejecutar `gcloud auth configure-docker` |
| Frontend no conecta con backend | Contenedores en redes distintas | Verificar que ambos están en `chatbot-network` |
| SSH desde GitHub Actions falla | Clave SSH incorrecta o firewall | Verificar secreto y regla de firewall en puerto 22 |
| `git pull` falla en Control Node | Autenticación Git no configurada | Configurar SSH key o token para el repo |
| Contenedor se reinicia en bucle | Error en la app o `.env` mal generado | `docker logs agent-backend` para diagnosticar |

---

## 14. Consideraciones de Seguridad

### Estado Actual (Válido para Demo/Proyectos Pequeños)

- El `.env` se inyecta en la imagen Docker durante el build. Esto significa que el secreto (`GOOGLE_API_KEY`) vive dentro de la capa de la imagen.
- La clave JSON de la service account se almacena en el Control Node.
- Se usa autenticación por clave SSH para la conexión GitHub → Control Node.

### Mejoras para Producción

| Mejora | Descripción |
|---|---|
| **Secret Manager** | Usar Google Secret Manager para inyectar secretos en runtime en vez de bake en la imagen |
| **Workload Identity** | Eliminar la necesidad de claves JSON asociando la SA directamente a la VM |
| **Variables de entorno runtime** | Pasar secretos como `-e` en `docker run` en vez de `COPY .env` |
| **Webhook en vez de SSH** | Implementar un listener webhook en el Control Node para mayor seguridad |
| **OIDC Federation** | Usar GitHub OIDC token en vez de clave JSON para autenticación en GCP |
| **Network policies** | Restringir tráfico entre contenedores solo a lo necesario |
| **HTTPS/TLS** | Añadir certificado SSL con Let's Encrypt + Nginx como proxy |

---

## 15. Resumen del Flujo Completo

```
┌─────────────────────────────────────────────────────────────────┐
│                        DEVELOPER                                 │
│                     git push main                                │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                    GITHUB ACTIONS                                │
│                                                                  │
│  1. Checkout código                                              │
│  2. Generar backend/.env con GOOGLE_API_KEY                      │
│  3. Autenticar en GCP (github-ci SA)                             │
│  4. Build agent-backend → Push a Artifact Registry               │
│  5. Build agent-frontend → Push a Artifact Registry              │
│  6. SSH → Control Node                                           │
└──────────────────────┬──────────────────────────────────────────┘
                       │ SSH
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CONTROL NODE (GCP VM)                          │
│                                                                  │
│  1. git pull (actualiza playbooks)                               │
│  2. ansible-playbook deploy.yml --extra-vars "image_tag=SHA"     │
└──────────────────────┬──────────────────────────────────────────┘
                       │ SSH + Ansible
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                    VM DE DESPLIEGUE (GCP VM)                      │
│                                                                  │
│  1. Docker login en Artifact Registry                            │
│  2. Pull agent-backend:SHA                                       │
│  3. Pull agent-frontend:SHA                                      │
│  4. Recrear contenedores en red chatbot-network                  │
│  5. Backend → puerto 8000:80                                     │
│  6. Frontend → puerto 80:80 (proxy reverso a backend)            │
│                                                                  │
│  ┌─────────────┐     ┌──────────────┐                           │
│  │  Frontend    │────▶│   Backend    │                           │
│  │  (nginx:80)  │     │  (python:80) │                           │
│  └─────────────┘     └──────────────┘                           │
│        ▲                                                         │
│        │ puerto 80                                               │
└────────┼────────────────────────────────────────────────────────┘
         │
    Usuario Final
```

---

## ✅ Checklist Final

- [ ] Service Account `github-ci` creada con rol `artifactregistry.writer`
- [ ] Artifact Registry `agent-repo` creado en `us-central1`
- [ ] Service Account del Control Node con rol `artifactregistry.reader`
- [ ] Docker configurado en Control Node para Artifact Registry
- [ ] Par de claves SSH generado y clave pública en `authorized_keys` del Control Node
- [ ] Todos los secretos configurados en GitHub (7 secretos)
- [ ] Workflow `.github/workflows/build-deploy.yml` creado
- [ ] Playbook `deploy.yml` en el repositorio del Control Node (`~/ansible-gcp`)
- [ ] Inventario con la IP de la VM de despliegue
- [ ] Red Docker `chatbot-network` creada (o se crea automáticamente con el playbook)
- [ ] Reglas de firewall: puerto 80 (HTTP) y 22 (SSH) abiertos según necesidad
- [ ] Test funcional del chatbot tras el primer despliegue

---

> 📝 **Versión:** 1.0  
> **Proyecto:** Crypto Chatbot Master UCM  
> **Stack:** Python (FastAPI/LangGraph) + Nginx + Docker + Ansible + GitHub Actions + GCP


OJO ------------------------   EN la VM 

gcloud auth configure-docker us-central1-docker.pkg.dev



```bash
gcloud artifacts repositories add-iam-policy-binding agent-repo \
  --location=us-central1 \
  --member="allUsers" \
  --role="roles/artifactregistry.reader"

```

ansible-playbook -i inventory/gcp.yml playbooks/deploy-bot.yml --extra-vars "image_tag=latest"



```yaml
---
# deploy-test.yml
# Test playbook - ejecutar desde el Control Node:
#   ansible-playbook deploy-test.yml -i inventory/hosts --extra-vars "image_tag=latest"

- name: Deploy Crypto Chatbot containers
  hosts: all
  become: true

  vars:
    registry_base: "us-central1-docker.pkg.dev/banded-pad-481109-q1/agent-repo"
    docker_network: "chatbot-network"

  tasks:

    - name: Crear red Docker
      community.docker.docker_network:
        name: "{{ docker_network }}"
        state: present

    - name: Desplegar backend
      community.docker.docker_container:
        name: agent-backend
        image: "{{ registry_base }}/agent-backend:{{ image_tag }}"
        state: started
        restart_policy: always
        pull: true
        recreate: true
        networks:
          - name: "{{ docker_network }}"
        ports:
          - "8000:80"

    - name: Desplegar frontend
      community.docker.docker_container:
        name: agent-frontend
        image: "{{ registry_base }}/agent-frontend:{{ image_tag }}"
        state: started
        restart_policy: always
        pull: true
        recreate: true
        networks:
          - name: "{{ docker_network }}"
        ports:
          - "80:80"

    - name: Verificar contenedores
      command: docker ps --format "{{ '{{' }}.Names{{ '}}' }} - {{ '{{' }}.Status{{ '}}' }} - {{ '{{' }}.Image{{ '}}' }}"
      register: docker_status

    - name: Mostrar estado
      debug:
        msg: "{{ docker_status.stdout_lines }}"

```


docker network rm chatbot-network
