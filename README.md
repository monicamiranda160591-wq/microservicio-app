# Microservicio de Catálogo de Productos (`productos-service`)

Microservicio desacoplado para la gestión del catálogo de productos, desarrollado bajo una arquitectura orientada a servicios con **Java 17**, **Spring Boot 3.2** y **Spring Cloud 2023**. Implementa persistencia autónoma en memoria con H2, endpoints de observabilidad con Spring Boot Actuator, capacidad de registro ante Discovery Clients (Eureka/Consul) y un pipeline declarativo completo de CI/CD en **Jenkins**.

---

## 1. Arquitectura y Capacidades de Microservicio

A diferencia de un monolito tradicional, este componente opera como un microservicio autónomo:
* **Responsabilidad Única:** Encapsula exclusivamente el dominio de catálogo e inventario (`/api/productos`).
* **Base de Datos Autónoma (Database per Service):** Almacenamiento local aislado mediante H2 en memoria (`productosdb`), sin acoplamiento a esquemas externos.
* **Service Discovery:** Preparado con `@EnableDiscoveryClient` y cliente Eureka (`spring-cloud-starter-netflix-eureka-client`) para registro automático en la red de microservicios y resolución a través de un API Gateway.
* **Observabilidad y Health Checks:** Expone métricas y estado del servicio a través de Spring Boot Actuator en `/actuator/health` y `/actuator/metrics`.
* **Contenedorización:** Empaquetado ligero y portable vía `Dockerfile` basado en Alpine Linux.

---

## 2. Stack Tecnológico

| Componente | Tecnología / Versión | Propósito |
|---|---|---|
| **Lenguaje** | Java 17 (Eclipse Temurin) | Runtime principal del servicio |
| **Framework Base** | Spring Boot 3.2.5 | Núcleo de la API REST y configuración |
| **Cloud** | Spring Cloud 2023.0.1 | Service Discovery (Eureka Client) y resiliencia |
| **Persistencia** | Spring Data JPA + H2 Database | ORM y base de datos relacional en memoria |
| **Monitoreo** | Spring Boot Actuator | Health check y telemetría del contenedor |
| **Pruebas** | JUnit 5 + Mockito + JaCoCo 0.8.11 | Suite de pruebas unitarias y cobertura de código |
| **Calidad de Código** | SonarQube Scanner 3.11 | Análisis estático de código y Quality Gate |
| **Repositorio de Binarios** | JFrog Artifactory OSS 7.x | Almacenamiento de artefactos versionados (`.jar`) |
| **Automatización** | Jenkins Pipeline Declarativo | Orquestador de Integración y Entrega Continua |
| **Contenedores** | Docker | Imagen base `eclipse-temurin:17-jre-alpine` |

---

## 3. Pipeline CI/CD (Jenkinsfile)

El pipeline automatizado en Jenkins consta de cuatro etapas secuenciales estrictas:

```
  ┌───────────┐     ┌────────────────────────┐     ┌───────────┐     ┌────────────────────────────────────┐
  │ 1. Build  │ ──► │ 2. Testing (UT+JaCoCo) │ ──► │ 3. Sonar  │ ──► │ 4. Generar y Almacenar Artifactory │
  └───────────┘     └────────────────────────┘     └───────────┘     └────────────────────────────────────┘
```

1. **Stage 1: Build**
   * Comando: `mvn clean compile -Dmaven.test.skip=true`
   * Valida la integridad sintáctica y compila todas las clases del microservicio.
2. **Stage 2: Testing (JUnit + JaCoCo)**
   * Comando: `mvn verify`
   * Ejecuta 37 pruebas unitarias de controladores y servicios.
   * Genera reporte de cobertura con JaCoCo y valida el umbral mínimo del **60% de cobertura de líneas**.
3. **Stage 3: Sonar**
   * Comando: `mvn sonar:sonar`
   * Envía el reporte de cobertura y código fuente a SonarQube.
   * La compuerta de calidad (`waitForQualityGate`) detiene el pipeline si no se aprueba el Quality Gate.
4. **Stage 4: Generar artefacto y almacenar en Artifactory**
   * Comando: `mvn package -Dmaven.test.skip=true`
   * Genera el artefacto final `productos-cicd-1.0.0.jar`.
   * Publica el binario versionado en el repositorio Maven `libs-release-local` de JFrog Artifactory mediante el plugin oficial `rtUpload`.

---

## 4. API Endpoints

### Catálogo de Productos
| Método | Endpoint | Descripción |
|---|---|---|
| `GET` | `/api/productos` | Listado completo de productos activos |
| `GET` | `/api/productos/{id}` | Búsqueda de producto por identificador |
| `GET` | `/api/productos/categoria/{categoria}` | Filtrar productos por categoría |
| `GET` | `/api/productos/rango-precio?min=X&max=Y` | Filtrar por rango de precios |
| `GET` | `/api/productos/stock-bajo?limite=N` | Productos con inventario por debajo del umbral |
| `POST` | `/api/productos` | Creación de nuevo producto con validaciones |
| `PUT` | `/api/productos/{id}` | Actualización completa de datos de producto |
| `PATCH` | `/api/productos/{id}/stock?cantidad=N` | Actualización atómica de stock |
| `DELETE` | `/api/productos/{id}` | Eliminación / desactivación lógica del producto |

### Observabilidad y Salud (Actuator)
| Método | Endpoint | Descripción |
|---|---|---|
| `GET` | `/actuator/health` | Estado del microservicio (`UP` / `DOWN`) |
| `GET` | `/actuator/info` | Metadatos de la aplicación y versión |
| `GET` | `/actuator/metrics` | Métricas de JVM, hilos y memoria |

---

## 5. Configuración Requerida en Jenkins

Para ejecutar el pipeline sobre la infraestructura del servidor:

### Herramientas (`Manage Jenkins` > `Tools`)
* **JDK:** `JDK-17` (apuntando a Java 17)
* **Maven:** `Maven-3.9`

### Credenciales (`Manage Jenkins` > `Credentials`)
* `sonar-auth-token`: Token de acceso generado en SonarQube.
* `artifactory-credentials`: Usuario y contraseña con permisos de escritura en Artifactory.

### Servidores Integrados (`Manage Jenkins` > `System`)
* **SonarQube Server:** Nombre `SonarQube` apuntando al host correspondiente.
* **JFrog Artifactory:** Server ID `Artifactory` configurado con URL `http://<IP_SERVIDOR>:8082/artifactory`.

---

## 6. Ejecución Local y Despliegue en Docker

### Compilar y probar localmente:
```bash
# Compilación y ejecución de pruebas con reporte JaCoCo
mvn clean verify
```

### Ejecutar la aplicación:
```bash
mvn spring-boot:run
```

### Construcción y ejecución del contenedor Docker:
```bash
# Generar imagen de contenedor ligera
docker build -t productos-service:1.0.0 .

# Iniciar contenedor exponiendo el puerto 8080
docker run -d -p 8080:8080 --name productos-microservice productos-service:1.0.0
```