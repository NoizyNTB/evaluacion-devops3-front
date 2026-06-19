# Sistema de Gestión de Despachos y Ventas

##  Descripción del Proyecto

Este es un sistema web de gestión de **despachos y ventas** con arquitectura de **microservicios**. La aplicación permite la administración integral de operaciones comerciales divididas en dos módulos principales:

- **Módulo de Ventas**: Gestión de productos, pedidos y transacciones comerciales
- **Módulo de Despachos**: Administración de envíos, seguimiento y logística

El proyecto utiliza una arquitectura **desacoplada** donde cada servicio es independiente, facilita el mantenimiento, escalabilidad y despliegue continuo. El sistema está diseñado con propósitos **académicos y educativos** para demostrar buenas prácticas en desarrollo de aplicaciones modernas y DevOps.

---

##  Tecnologías Utilizadas

### **Frontend**
- **React 18** - Librería de UI
- **Vite** - Herramienta de build rápida
- **Tailwind CSS** - Framework de estilos
- **JavaScript/JSX** - Lenguaje de programación

### **Backend**
- **Spring Boot** - Framework Java para microservicios (dos instancias)
- **RESTful API** - Arquitectura de APIs
- **MySQL** - Base de datos (Ambos backend)

### **DevOps & Infraestructura**
- **Docker** - Containerización de aplicaciones
- **Docker Compose** - Orquestación local de contenedores
- **Nginx** - Servidor web y reverse proxy
- **GitHub Actions** - Pipeline CI/CD
- **AWS EC2** - Infraestructura en la nube
- **AWS ECR** - Registro privado de imágenes Docker
- **AWS Systems Manager (SSM)** - Despliegue remoto en EC2

---

##  Arquitectura del Proyecto

La aplicación está estructurada con **Nginx como reverse proxy** que enruta las peticiones a los diferentes servicios backend:

- **Frontend (React)**: Interfaz de usuario servida en puerto 80
- **Backend Ventas (Spring Boot)**: API en puerto 8080, accesible vía `/api/v1/ventas/*`
- **Backend Despachos (Spring Boot)**: API en puerto 8081, accesible vía `/api/v1/despachos/*`
- **Base de Datos**: MySQL accesible desde ambos backends

Nginx actúa como punto de entrada único, enrutando cada petición al servicio correspondiente según la ruta solicitada.


---

##  Pipeline CI/CD (GitHub Actions)

### **¿Cómo funciona?**

El pipeline está definido en [.github/workflows/main.yml](.github/workflows/main.yml) y se ejecuta automáticamente al hacer `git push` a la rama `main`.

### **Flujo del Pipeline**

El pipeline se ejecuta automáticamente al hacer `git push` a la rama `main` con dos etapas principales:

1. **Build & Push**: Construye la imagen Docker y la publica en AWS ECR
2. **Deploy to EC2**: Descarga la imagen de ECR y la ejecuta en la instancia EC2

### **Descripción de cada etapa**

#### **1. Build & Push (build-and-push)**

**Qué hace:**
- Descarga el código del repositorio
- Autentica con AWS usando credenciales
- **Construye** la imagen Docker usando el `Dockerfile`
- **Etiqueta** la imagen con:
  - Hash del commit (`github.sha`)
  - `latest` (última versión)
- **Publica** ambas versiones en AWS ECR

**Dockerfile usado:**
```dockerfile
# Stage 1: Build
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Producción
FROM nginx:1.27-alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**Beneficios del Multi-stage:**
-  Reduce tamaño de imagen (~100MB → ~50MB)
-  No incluye herramientas de build en producción
-  Mejora seguridad (menos paquetes = menos vulnerabilidades)

#### **2. Deploy to EC2 (deploy-to-ec2)**

**Qué hace:**
- Espera a que termine el build
- Autentica con AWS
- Usa **AWS Systems Manager (SSM)** para ejecutar comandos remotos en EC2:
  1. Crea directorio `/home/ec2-user/frontend`
  2. Autentica Docker con ECR
  3. **Descarga** la nueva imagen desde ECR
  4. **Detiene** el contenedor anterior (`react-frontend`)
  5. **Elimina** el contenedor viejo
  6. **Ejecuta** nuevo contenedor mapeando puerto 80
  7. **Limpia** imágenes de Docker para liberar espacio

**Comando en EC2:**
```bash
docker run -d --name react-frontend -p 80:80 \
  [AWS_ACCOUNT_ID].dkr.ecr.[REGION].amazonaws.com/[REPOSITORY]:latest
```

### **Variables de Entorno (Secrets en GitHub)**

Configura estos **GitHub Secrets** en el repositorio:

```
AWS_ACCESS_KEY_ID          → Tu Access Key de AWS
AWS_SECRET_ACCESS_KEY      → Tu Secret Key de AWS
AWS_SESSION_TOKEN          → Token temporal (si aplica)
AWS_REGION                 → Región (ej: us-east-1)
AWS_ACCOUNT_ID             → ID de tu cuenta AWS
AWS_ECR_REPOSITORY         → Nombre del repositorio ECR
EC2_INSTANCE_ID            → ID de la instancia EC2
```

**Cómo configurarlos:**
1. Ve a tu repositorio en GitHub
2. Settings → Secrets and variables → Actions
3. Click en "New repository secret"
4. Agrega cada variable

---

##  Despliegue en AWS EC2

### **Infraestructura en AWS**

El sistema se ejecuta en una instancia **EC2** con Docker instalado. Las imágenes se almacenan en **AWS ECR (Elastic Container Registry)** y se descargan automáticamente durante el despliegue.

### **Pasos para desplegar manualmente**

#### **1. Conectar a la instancia EC2**

```bash
ssh -i tu-clave.pem ec2-user@[IP-PUBLICA-EC2]
```

#### **2. Instalar Docker (si no está instalado)**

```bash
sudo yum update -y
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
```

#### **3. Configurar credenciales de AWS**

```bash
aws configure
# Ingresa: Access Key, Secret Key, Región, Formato (json)
```

#### **4. Ejecutar la imagen (manual)**

```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin [AWS_ACCOUNT_ID].dkr.ecr.us-east-1.amazonaws.com

docker pull [AWS_ACCOUNT_ID].dkr.ecr.us-east-1.amazonaws.com/frontend:latest

docker run -d --name react-frontend -p 80:80 \
  [AWS_ACCOUNT_ID].dkr.ecr.us-east-1.amazonaws.com/frontend:latest
```

#### **5. Verificar que el contenedor está corriendo**

```bash
docker ps
```

**Nota:** Las peticiones desde el Frontend van a través de Nginx, que actúa como proxy y las redirige a los backends correspondientes.
