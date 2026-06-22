# Evaluación DevOps 3 - Frontend (Deploy Branch)

## Descripción General

Este repositorio contiene la aplicación **frontend React** del sistema de gestión de ventas y despachos. La rama `deploy` está configurada con un pipeline CI/CD automático que:

1. Construye la aplicación React
2. Empaqueta en contenedor Docker
3. Publica la imagen en AWS ECR (Elastic Container Registry)
4. Despliega automáticamente en Kubernetes (EKS)

**Nota Importante**: Esta es una aplicación **frontend-only**. El backend de ventas y despachos se ejecuta en servicios separados. El frontend consume APIs REST desde esos servicios a través de un proxy Nginx.

---

## Arquitectura

### Capas de la Aplicación

```
┌────────────────────────────────────────────────┐
│  React SPA (Single Page Application)           │  ← Este repositorio
│  - Componentes React                           │
│  - Gestión de estado                           │
│  - Interfaz de usuario con Tailwind CSS        │
└──────────────┬───────────────────────────────┘
               │
               ▼ (HTTP via Nginx proxy)
        ┌──────────────────────┐
        │   Nginx Reverse      │
        │   Proxy Server       │  ← Docker Container
        └──────────────────────┘
        │                   │
   ┌────▼─────┐       ┌─────▼───────┐
   │ Backend   │       │  Backend    │
   │ Ventas    │       │  Despachos  │
   │ :8080     │       │  :8081      │
   └───────────┘       └─────────────┘
```

### Componentes Clave

| Componente | Tecnología | Propósito |
|-----------|-----------|----------|
| **Frontend Framework** | React 18.2.0 | UI interactiva y gestión de estado |
| **Build Tool** | Vite 5.2.0 | Empaquetado y optimización |
| **Estilos** | Tailwind CSS 3.4.3 | Diseño responsivo |
| **HTTP Client** | Axios 1.6.8 | Llamadas a APIs backend |
| **Enrutador** | React Router 6.24.1 | Navegación SPA |
| **Web Server** | Nginx 1.27 | Servir frontend y proxy APIs |
| **Contenedor** | Docker | Empaquetar aplicación |
| **Orquestador** | Kubernetes (EKS) | Ejecutar y escalar en AWS |

---

## Estructura del Proyecto

```
evaluacion-devops3-front/
│
├── src/                          # Código fuente de React
│   ├── main.jsx                  # Punto de entrada de la aplicación
│   ├── index.css                 # Estilos globales
│   ├── Routes/                   # Configuración de rutas
│   └── componentes/              # Componentes React
│       ├── Layouts/              # Componentes de estructura
│       └── CrudAdmin/            # CRUD de ventas y despachos
│           ├── TableCompras.jsx      # Tabla de pedidos sin despacho
│           ├── TableDespachos.jsx    # Tabla de despachos
│           ├── FormDespacho.jsx      # Formulario crear despacho
│           └── FormCierreDespacho.jsx # Formulario cerrar despacho
│
├── k8s/                          # Manifiestos Kubernetes
│   ├── frontend-deployment.yaml  # Deployment de frontend
│   └── front-service.yaml        # Service LoadBalancer
│
├── .github/workflows/            # CI/CD Pipeline
│   └── main.yml                  # GitHub Actions workflow
│
├── public/                       # Archivos estáticos
├── package.json                  # Dependencias Node.js
├── vite.config.js               # Configuración de Vite
├── dockerfile                   # Definición de imagen Docker
├── default.conf                 # Configuración Nginx
├── tailwind.config.js           # Configuración Tailwind CSS
└── README.md                    # Documentación original
```

---

## 🔌 Endpoints de API

El frontend consume las siguientes APIs del backend (accedidas a través del proxy Nginx):

### 1. **Obtener Ventas**
- **Endpoint**: `GET /api/v1/ventas`
- **Propósito**: Lista todas las compras sin despacho asignado
- **Respuesta**: Array de objetos venta
- **Campos**: `idVenta`, `direccionCompra`, `valorCompra`, `fechaCompra`, `despachoGenerado`
- **Usado en**: `TableCompras.jsx`

### 2. **Crear Despacho**
- **Endpoint**: `POST /api/v1/despachos`
- **Propósito**: Crear nueva orden de despacho desde una venta
- **Body**: 
  ```json
  {
    "idVenta": "uuid",
    "direccionDespacho": "dirección",
    "patenteCamion": "patente"
  }
  ```
- **Usado en**: `FormDespacho.jsx`

### 3. **Marcar Venta Despachada**
- **Endpoint**: `PUT /api/v1/ventas/{ventaId}`
- **Propósito**: Marcar que una venta ya tiene despacho asignado
- **Usado en**: `FormDespacho.jsx` (después de crear despacho)

### 4. **Obtener Despachos**
- **Endpoint**: `GET /api/v1/despachos`
- **Propósito**: Lista todas las órdenes de despacho
- **Respuesta**: Array de objetos despacho
- **Campos**: `idDespacho`, `idCompra`, `direccionCompra`, `fechaDespacho`, `patenteCamion`, `entregado`, `intento`
- **Usado en**: `TableDespachos.jsx`

### 5. **Actualizar Despacho**
- **Endpoint**: `PUT /api/v1/despachos/{despachoId}`
- **Propósito**: Actualizar estado de despacho (intentos, entregado, etc.)
- **Usado en**: `FormCierreDespacho.jsx`

### Headers HTTP Comunes
```javascript
{
  'Content-Type': 'application/json',
  'Accept': 'application/json'
}
```

---

## Docker & Contenedorización

### Dockerfile Multi-Stage

El proyecto usa un **Dockerfile de dos etapas** para optimizar el tamaño de la imagen final:

```dockerfile
# ETAPA 1: BUILD
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci                    # Instala dependencias exactas
RUN npm run build             # Genera carpeta /dist

# ETAPA 2: PRODUCTION
FROM nginx:1.27-alpine
COPY --from=0 /app/dist /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

**Beneficios**:
- Imagen final de ~50MB (sin Node.js ni código fuente)
- Seguridad mejorada (no incluye herramientas de build)
- Tiempo de despliegue más rápido
- Menor consumo de recursos

### Configuración Nginx (default.conf)

El archivo `default.conf` configura Nginx como:

1. **Servidor estático** para la SPA React
2. **Reverse proxy** para APIs backend

**Proxies configurados**:
```nginx
upstream backend_ventas {
    server ventas-service:8080;           # Servicio backend ventas
}

upstream backend_despachos {
    server despacho-service:8081;         # Servicio backend despachos
}

# Ruta raíz: Sirve React SPA
location / {
    try_files $uri $uri/ /index.html;     # SPA routing
}

# Proxy a backend ventas
location /api/v1/ventas {
    proxy_pass http://backend_ventas;
}

# Proxy a backend despachos
location /api/v1/despachos {
    proxy_pass http://backend_despachos;
}
```

**Nombres de servicios esperados**:
- `ventas-service` (hostname DNS en Kubernetes)
- `despacho-service` (hostname DNS en Kubernetes)

---

## CI/CD Pipeline (GitHub Actions)

### Flujo de Despliegue

La rama `deploy` tiene un workflow automático que se ejecuta en cada push:

```
┌─────────────────────────────────────┐
│ Push a rama 'deploy'                │
└────────────────┬────────────────────┘
                 │
      ┌──────────▼──────────┐
      │ JOB 1: Build & Push │
      └──────────┬──────────┘
                 │
    ┌────────────┴───────────────────┐
    │                                │
1.  npm ci                           │
2.  npm run build                    │
3.  docker build (multi-stage)       │
4.  docker push to ECR               │
    - Tag 1: {commit_sha}            │
    - Tag 2: latest                  │
    │                                │
    └────────────┬────────────────────┘
                 │
      ┌──────────▼──────────────┐
      │ JOB 2: Deploy a EKS     │
      │ (depende de Job 1)      │
      └──────────┬──────────────┘
                 │
1.  aws eks update-kubeconfig        │
2.  kubectl apply -f k8s/            │
3.  kubectl set image deployment/... │
4.  kubectl rollout status           │
5.  kubectl get pods                 │
```

### Archivos de Configuración

**`.github/workflows/main.yml`**:
- Triggers: Push a `deploy` o manual `workflow_dispatch`
- Variables de entorno: AWS credentials desde GitHub Secrets
- Outputs: `image_tag` (commit SHA para trazabilidad)

### Secretos Requeridos en GitHub

Para que el pipeline funcione, debes configurar estos **GitHub Secrets**:

| Secret | Valor Ejemplo | Propósito |
|--------|--------------|----------|
| `AWS_ACCESS_KEY_ID` | `AKIA...` | Autenticación AWS |
| `AWS_SECRET_ACCESS_KEY` | `wJal...` | Autenticación AWS |
| `AWS_SESSION_TOKEN` | `(opcional)` | Token de sesión temporal |
| `AWS_REGION` | `us-east-1` | Región AWS para ECR y EKS |
| `AWS_ACCOUNT_ID` | `123456789012` | ID de cuenta AWS |
| `AWS_ECR_REPOSITORY` | `frontend` | Nombre repositorio ECR |
| `EKS_CLUSTER_NAME` | `mi-cluster-eks` | Nombre del cluster EKS |
| `EKS_NAMESPACE` | `production` | Namespace de Kubernetes |
| `K8S_DEPLOYMENT_NAME` | `frontend` | Nombre del Deployment |
| `K8S_CONTAINER_NAME` | `frontend-app` | Nombre del contenedor |

---

## Kubernetes (EKS) Despliegue

### Manifiestos Kubernetes

#### **frontend-deployment.yaml**
Define cómo ejecutar la aplicación en Kubernetes:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend              # Deployment name
spec:
  replicas: 1                 # Una sola réplica (configurable)
  selector:
    matchLabels:
      app: frontend
  template:
    spec:
      containers:
        - name: frontend-app  # Nombre del contenedor
          image: nginx:alpine # Placeholder, reemplazado por CI/CD
          ports:
            - containerPort: 80
```

**Notas**:
- La `image` es un placeholder. El workflow de GitHub la reemplaza con la imagen ECR
- El nombre `frontend-app` debe coincidir con `K8S_CONTAINER_NAME` en secrets
- Replicas = 1 (sin autoescalado activo)

#### **front-service.yaml**
Expone la aplicación al tráfico externo:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: front-service
spec:
  type: LoadBalancer              # Crea AWS NLB
  selector:
    app: frontend                 # Apunta a pods con esta etiqueta
  ports:
    - port: 80                    # Puerto externo
      targetPort: 80              # Puerto del contenedor
```

**Tipo LoadBalancer**:
- AWS crea automáticamente un **Network Load Balancer (NLB)**
- Asigna una IP pública para acceso desde internet
- Distribuye tráfico a los pods `frontend`

### Despliegue Manual (sin GitHub Actions)

Si necesitas desplegar manualmente:

```bash
# 1. Configurar kubeconfig
aws eks update-kubeconfig --region us-east-1 --name mi-cluster-eks

# 2. Aplicar manifiestos
kubectl apply -f k8s/ -n production

# 3. Actualizar imagen (si ya existe)
kubectl set image deployment/frontend \
  frontend-app=123456789012.dkr.ecr.us-east-1.amazonaws.com/frontend:commit-sha \
  -n production

# 4. Verificar despliegue
kubectl rollout status deployment/frontend -n production

# 5. Ver pods
kubectl get pods -n production

# 6. Obtener URL del LoadBalancer
kubectl get service front-service -n production
```

---

## Desarrollo Local

### Requisitos Previos

- **Node.js** v18+
- **npm** v9+ (o yarn)
- **Git**

### Instalación y Ejecución

```bash
# 1. Clonar repositorio
git clone https://github.com/NoizyNTB/evaluacion-devops3-front.git
cd evaluacion-devops3-front

# 2. Cambiar a rama deploy
git checkout deploy

# 3. Instalar dependencias
npm ci                    # npm clean install (preferible a npm install)

# 4. Ejecutar servidor de desarrollo
npm run dev

# Abre http://localhost:5173 en el navegador
```

### Scripts Disponibles

| Script | Comando | Propósito |
|--------|---------|----------|
| Dev | `npm run dev` | Servidor Vite con hot reload |
| Build | `npm run build` | Construir para producción (/dist) |
| Preview | `npm run preview` | Vista previa de build local |
| Lint | `npm run lint` | Validar código con ESLint |

### Desarrollo contra Backend Local

En `vite.config.js` ya está configurado un proxy para desarrollo:

```javascript
server: {
  proxy: {
    '/api': {
      target: 'https://qic534o8o0.execute-api.us-east-1.amazonaws.com',
      changeOrigin: true
    }
  }
}
```

Para apuntar a tu backend local, modificar el `target` en tiempo de desarrollo.

---

## AWS ECR (Elastic Container Registry)

### Push Manual de Imagen (sin GitHub Actions)

```bash
# 1. Construir imagen
docker build -t frontend:latest .

# 2. Autenticarse en ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# 3. Etiquetar imagen
docker tag frontend:latest \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/frontend:latest

# 4. Push a ECR
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/frontend:latest
```

### Estructura de Tags en ECR

El workflow de GitHub empuja dos tags:

- `{COMMIT_SHA}` - Identificador único de la versión
- `latest` - Última versión compilada

Esto permite:
- Trazabilidad: Saber qué commit se está ejecutando
- Rollback fácil: Volver a versión anterior si hay problemas

---

## Variables de Entorno

### Frontend (no aplica - React compila en build time)

React no tiene un archivo `.env` de runtime. Las variables de build se incluyen en tiempo de compilación.

### Backend (servicios externos)

Los servicios backend (`ventas-service`, `despacho-service`) requieren configuración:

- **MYSQL_HOST**: Hostname del servicio MySQL
- **MYSQL_PORT**: Puerto MySQL (default 3306)
- **MYSQL_USER**: Usuario de base de datos
- **MYSQL_PASSWORD**: Contraseña de base de datos
- **MYSQL_DATABASE**: Nombre de la base de datos

Estas variables se configuran en los Deployments de los servicios backend, no en el frontend.

---

##Troubleshooting

### Problema: "Connection refused" en APIs

**Causa**: Los servicios backend no están accesibles  
**Solución**:
```bash
# Verificar que los servicios existen en K8s
kubectl get svc -n production

# Verificar conexión de red
kubectl run debug -it --image=curlimages/curl -- sh
curl http://ventas-service:8080/health
```

### Problema: Nginx retorna 502 Bad Gateway

**Causa**: Nginx no puede resolver los nombres de los servicios backend  
**Solución**:
- Verificar que `ventas-service` y `despacho-service` existen
- Verificar que están en el mismo namespace
- Revisar logs de Nginx:
```bash
kubectl logs -l app=frontend -n production
```

### Problema: Imagen no se actualiza en Kubernetes

**Causa**: Pod sigue usando imagen anterior  
**Solución**:
```bash
# Forzar new deployment
kubectl rollout restart deployment/frontend -n production

# O eliminar pods para que se recreen con nueva imagen
kubectl delete pod -l app=frontend -n production
```

### Problema: npm install falla durante build de Docker

**Causa**: Lock file desincronizado  
**Solución**: El workflow usa `npm ci` en lugar de `npm install` (más confiable)

---

## Monitoreo y Logs

### Verificar Estado del Despliegue

```bash
# Status del deployment
kubectl describe deployment frontend -n production

# Logs de un pod
kubectl logs -l app=frontend -n production

# Logs en tiempo real
kubectl logs -f -l app=frontend -n production

# Eventos recientes
kubectl get events -n production --sort-by='.lastTimestamp'
```

### Acceder a la Aplicación

```bash
# Obtener URL pública
kubectl get svc front-service -n production
# La URL está en la columna EXTERNAL-IP

# O hacer port-forward local
kubectl port-forward svc/front-service 8080:80 -n production
# Luego acceder a http://localhost:8080
```

---

## Workflow de Despliegue Recomendado

### Para Desarrolladores

1. **Desarrollo**: Rama `main` o feature branches
2. **Testing**: Validar localmente con `npm run dev`
3. **Merge a deploy**: Push a rama `deploy` cuando esté listo
4. **CI/CD automático**: GitHub Actions construye y despliega
5. **Verificación**: Validar en EKS que la app está corriendo

### Comandos Típicos

```bash
# Desarrollo local
git checkout -b feature/nueva-funcionalidad
npm run dev

# Cuando está listo
git commit -am "Agregar nueva funcionalidad"
git push origin feature/nueva-funcionalidad

# En GitHub: hacer PR a rama deploy
# Después de review y merge:

# La rama deploy tendrá el nuevo commit
# GitHub Actions se activará automáticamente
# La imagen se construirá y desplegará en EKS

# Verificar despliegue
kubectl get pods -n production
kubectl logs -l app=frontend -n production
```

---
