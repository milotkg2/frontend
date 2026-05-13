# alumnos-web

Frontend del sistema **Alumnos**, construido con **React 18** y **Vite 6**. Interfaz CRUD completa con soporte para importacion/exportacion CSV, servida por **Nginx** como proxy inverso hacia la capa App.

---

## Tecnologias

| Tecnologia              | Version  | Uso                              |
| ----------------------- | -------- | -------------------------------- |
| React                   | 18.3     | Framework UI                     |
| Vite                    | 6.0      | Build tool y dev server          |
| Axios                   | 1.7      | Cliente HTTP                     |
| Nginx                   | 1.27     | Servidor estatico + proxy inverso |
| Vitest                  | 2.1      | Test runner                      |
| Testing Library / React | 16.1     | Tests de componentes             |
| MSW (Mock Service Worker) | 2.7    | Mocking de API en tests          |
| Node.js                 | 20       | Entorno de build                 |

---

## Estructura del proyecto

```
alumnos-web/
├── src/
│   ├── api/
│   │   └── alumnos.js          # Cliente Axios — llamadas al backend
│   ├── components/
│   │   ├── AlumnoForm.jsx      # Formulario crear/editar alumno
│   │   ├── AlumnoTable.jsx     # Tabla con acciones editar/eliminar
│   │   └── CsvPanel.jsx        # Panel importar/exportar CSV
│   ├── hooks/
│   │   └── useAlumnos.js       # Hook con toda la logica de estado
│   ├── test/
│   │   ├── api/                # Tests del cliente HTTP
│   │   ├── components/         # Tests de componentes React
│   │   ├── hooks/              # Tests del hook useAlumnos
│   │   ├── mocks/              # Handlers MSW y servidor mock
│   │   ├── App.test.jsx        # Test de integracion del App
│   │   └── setup.js            # Configuracion global de tests
│   ├── App.jsx                 # Componente raiz
│   ├── index.css               # Estilos globales
│   └── main.jsx                # Entry point
├── Dockerfile                  # Build multi-etapa (Node + Nginx)
├── docker-compose.yml          # Servicio standalone con red compartida
├── nginx.conf                  # Config Nginx con proxy dinamico
├── vite.config.js              # Config Vite + Vitest + cobertura
├── package.json
└── README.md
```

---

## Requisitos

| Herramienta    | Version minima |
| -------------- | -------------- |
| Docker         | 20.10+         |
| Docker Compose | 2.0+           |
| Node.js        | 20+ (solo para desarrollo local sin Docker) |
| npm            | 10+ |

---

## Inicio rapido con Docker

### Prerequisitos

Este compose asume que `alumnos-app` ya esta corriendo en la red `alumnos-network`.

**1. Crear la red compartida (solo la primera vez):**
```bash
docker network create alumnos-network
```

**2. Levantar la base de datos:**
```bash
cd ../alumnos-datos && docker compose up -d
```

**3. Levantar el backend:**
```bash
cd ../alumnos-app && docker compose up -d --build
```

### Levantar el frontend

```bash
docker compose up -d --build
```

### Verificar que esta corriendo

```bash
docker compose ps
```

Resultado esperado:

```
NAME          STATUS       PORTS
alumnos-web   Up X seconds 0.0.0.0:80->80/tcp
```

### Abrir en el navegador

```
http://localhost
```

### Detener

```bash
docker compose down
```

---

## Desarrollo local sin Docker

### 1. Instalar dependencias

```bash
npm install
```

### 2. Levantar el dev server

```bash
npm run dev
```

La app queda disponible en `http://localhost:3000`.

El dev server incluye un proxy configurado en `vite.config.js`:

```
/alumnos  →  http://localhost:8080
```

Requiere que `alumnos-app` este corriendo en `localhost:8080`.

### 3. Build de produccion

```bash
npm run build
# Salida en: dist/
```

### 4. Preview del build

```bash
npm run preview
```

---

## Funcionalidades

### CRUD de alumnos

| Accion    | Descripcion                                      |
| --------- | ------------------------------------------------ |
| Listar    | Tabla con todos los alumnos registrados          |
| Crear     | Formulario con campos nombre y apellido          |
| Editar    | Formulario pre-cargado con datos del alumno      |
| Eliminar  | Confirmacion antes de borrar                     |

### Panel CSV

| Accion   | Descripcion                                           |
| -------- | ----------------------------------------------------- |
| Exportar | Descarga `alumnos.csv` con todos los registros        |
| Importar | Pega CSV en el textarea y sube multiples alumnos      |

Formato CSV esperado (sin encabezado):
```
Juan,Perez
Ana,Lopez
Carlos,Soto
```

### Stats en tiempo real

- Total de alumnos registrados
- Cantidad de apellidos unicos

---

## Proxy Nginx

En produccion, Nginx actua como proxy inverso hacia el backend. El hostname del backend se inyecta en runtime via la variable de entorno `BACKEND_HOST`:

```nginx
location /alumnos {
    proxy_pass http://${BACKEND_HOST}:8080/alumnos;
}
```

| Entorno       | BACKEND_HOST       | Ejemplo                        |
| ------------- | ------------------ | ------------------------------ |
| Docker local  | `alumnos-app`      | nombre del contenedor          |
| AWS EC2       | IP privada         | `10.0.1.45`                    |

El reemplazo se realiza con `envsubst` al iniciar el contenedor, sin necesidad de rebuild.

---

## Tests

### Ejecutar todos los tests

```bash
npm test
```

### Ejecutar con cobertura

```bash
npm run test:coverage
# Reporte en: coverage/index.html
```

Umbral minimo requerido: **90% en lineas, funciones, ramas y sentencias**. Si no se alcanza, el build de Docker falla en la etapa de tests.

### Modo watch (desarrollo)

```bash
npm run test:watch
```

### Interfaz visual de tests

```bash
npm run test:ui
```

### Cobertura por modulo

| Modulo           | Descripcion                              |
| ---------------- | ---------------------------------------- |
| `src/api/`       | Tests del cliente Axios con MSW          |
| `src/components/`| Tests de renderizado e interaccion       |
| `src/hooks/`     | Tests del hook `useAlumnos`              |
| `src/App.jsx`    | Test de integracion del componente raiz  |

---

## Imagen Docker

| Propiedad    | Valor                  |
| ------------ | ---------------------- |
| Base build   | `node:20-alpine`       |
| Base runtime | `nginx:1.27-alpine`    |
| Imagen ECR   | `alumnos-web:latest`   |
| Puerto       | `80`                   |
| Healthcheck  | `GET http://localhost` |

### Build multi-etapa

El Dockerfile usa 4 etapas para garantizar calidad y minimizar la imagen final:

| Etapa    | Base              | Descripcion                                      |
| -------- | ----------------- | ------------------------------------------------ |
| `deps`   | node:20-alpine    | Instala dependencias npm                         |
| `test`   | deps              | Ejecuta tests con cobertura (falla si < 90%)     |
| `build`  | test              | Genera el bundle de produccion con Vite          |
| `runtime`| nginx:1.27-alpine | Sirve los assets estaticos con proxy al backend  |

> Si los tests fallan, el build se detiene y no se genera la imagen.

### Construir la imagen manualmente

```bash
docker build -t alumnos-web:latest .
```

### Publicar en ECR

```bash
# Autenticarse en ECR
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin <ECR_REGISTRY>

# Tag y push
docker tag alumnos-web:latest <ECR_REGISTRY>/alumnos-web:latest
docker push <ECR_REGISTRY>/alumnos-web:latest
```

---

## Variables de entorno

| Variable       | Descripcion                          | Ejemplo          |
| -------------- | ------------------------------------ | ---------------- |
| `BACKEND_HOST` | Hostname o IP del backend (runtime)  | `alumnos-app`    |
| `VITE_API_URL` | URL del backend en dev server (build)| `http://localhost:8080` |

> `BACKEND_HOST` se usa en produccion (Nginx). `VITE_API_URL` se usa en desarrollo local (Vite proxy).

---

## Contexto en la arquitectura

Esta imagen forma parte de la infraestructura de 3 capas del sistema Alumnos:

```
Internet
   |
EC2-Web   (alumnos-web:latest)    — Capa Web    — Puerto 80   <-- este servicio
   |  (proxy /alumnos → :8080)
EC2-App   (alumnos-app:latest)    — Capa App    — Puerto 8080
   |
EC2-Datos (alumnos-datos:latest)  — Capa Datos  — Puerto 5432
```

- Unico punto de entrada desde internet (SG-Web permite TCP 80 desde 0.0.0.0/0)
- Desplegada en `Subnet-Public` con IP publica asignada automaticamente
- El frontend no llama directamente al backend — todo pasa por el proxy Nginx
- La IP privada del backend se resuelve en tiempo de despliegue desde SSM Parameter Store

---

## Notas de seguridad

- El frontend no almacena credenciales ni tokens en el codigo fuente.
- `BACKEND_HOST` se inyecta en runtime, no en el build, evitando exponer IPs en la imagen.
- En produccion, el puerto 8080 del backend no esta expuesto publicamente — solo accesible desde el proxy Nginx interno.
- Los assets estaticos tienen cache de 1 año con `Cache-Control: public, immutable`.
