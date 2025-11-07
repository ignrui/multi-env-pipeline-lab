# Informe - Laboratorio 2: Canalizaciones CI/CD multiambiente con GitHub Actions

## Parte 1: Preparación del entorno

### Paso 1.1: Verificar el clúster de Kubernetes 

**Comandos ejecutados:**
```bash
minikube start --driver=docker --cpus=4 --memory=4096
kubectl cluster-info
kubectl get nodes
kubectl get namespaces
```

**Acciones realizadas:**
Iniciamos el clúster de Kubernetes utilizando Minikube con Docker como driver. Configuramos 4 CPUs y 4096MB de memoria para asegurar que el clúster tenga recursos suficientes para ejecutar múltiples aplicaciones en diferentes entornos.

**Resultados obtenidos:**
- Minikube se inició correctamente en la versión v1.37.0
- El clúster de Kubernetes (v1.34.0) está funcionando en https://127.0.0.1:65132
- El nodo minikube está en estado "Ready" con el rol control-plane
- Los namespaces por defecto (default, kube-system, kube-public, kube-node-lease) están activos

**Problemas encontrados:**
Inicialmente Docker no estaba ejecutándose, lo que causó un error al intentar iniciar Minikube. Se resolvió iniciando Docker Desktop antes de ejecutar el comando de Minikube.

### Paso 1.2: Crear espacios de nombres por entorno 

**Comandos ejecutados:**
```bash
kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace staging --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace prod --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace dev environment=dev
kubectl label namespace staging environment=staging
kubectl label namespace prod environment=prod
kubectl get namespaces --show-labels | grep -E "dev|staging|prod"
```

**Acciones realizadas:**
Creamos tres namespaces separados en nuestro clúster de Kubernetes para aislar los diferentes entornos de despliegue. Cada namespace fue etiquetado apropiadamente para facilitar su identificación y gestión.

**Resultados obtenidos:**
- Namespace `dev` creado con etiqueta `environment=dev`
- Namespace `staging` creado con etiqueta `environment=staging` 
- Namespace `prod` creado con etiqueta `environment=prod`
- Todos los namespaces están activos y correctamente etiquetados

**Explicación:**
La separación por namespaces nos permite aislar completamente los recursos de cada entorno, evitando conflictos entre deployments, servicios y configuraciones. Las etiquetas facilitan la identificación y selección de recursos mediante selectores en nuestras canalizaciones CI/CD.

### Paso 1.3: Crear repositorio en GitHub

**Acciones realizadas:**
Creamos un nuevo repositorio en GitHub con el nombre `multi-env-pipeline-lab` siguiendo las especificaciones del laboratorio:

- **Nombre del repositorio**: multi-env-pipeline-lab
- **Descripción**: Lab 2: Multi-environment CI/CD pipelines
- **Configuración**: Repositorio público inicializado con README
- **URL**: https://github.com/ignrui/multi-env-pipeline-lab

**Resultado:**
El repositorio se creó exitosamente y está listo para ser utilizado como base para nuestras canalizaciones CI/CD multiambiente.

### Paso 1.4: Clonar y configurar el repositorio local

**Comandos ejecutados:**
```bash
git clone https://github.com/ignrui/multi-env-pipeline-lab.git
cd multi-env-pipeline-lab
ls -la
cat README.md
git config --global user.name && git config --global user.email
```

**Acciones realizadas:**
Clonamos el repositorio recién creado en nuestro entorno local dentro del directorio de trabajo del laboratorio. Verificamos que la configuración de Git esté correctamente establecida con nuestras credenciales.

**Resultados obtenidos:**
- Repositorio clonado exitosamente en `/home/ctag/formacion/DevOps/Lab1.2/multi-env-pipeline-lab`
- Directorio `.git` presente, confirmando que es un repositorio válido
- README.md inicial con el título y descripción del proyecto
- Configuración de Git verificada:
  - Usuario: ignrui
  - Email: ignacio.ruiz@ctag.com

**Explicación:**
Establecimos nuestro entorno de desarrollo local conectado al repositorio remoto de GitHub. Este será nuestro punto de partida para desarrollar las canalizaciones CI/CD, manifiestos de Kubernetes y configuraciones específicas por entorno.

---

## Resumen de la Parte 1 

Completamos exitosamente la preparación del entorno para el laboratorio:

1.  **Clúster de Kubernetes**: Minikube funcionando con Docker (4 CPUs, 4GB RAM)
2.  **Namespaces por entorno**: dev, staging y prod correctamente etiquetados
3.  **Repositorio GitHub**: multi-env-pipeline-lab creado y configurado
4.  **Entorno local**: Repositorio clonado y Git configurado

Nuestro entorno está completamente preparado para comenzar con el desarrollo de las canalizaciones CI/CD multiambiente.

---

## Parte 2: Crear la aplicación y configuraciones de entorno

### Paso 2.1: Crear estructura del proyecto 

**Comandos ejecutados:**
```bash
mkdir -p apps/webapp
mkdir -p environments/{dev,staging,prod}
mkdir -p .github/{workflows,actions/deploy}
mkdir -p scripts
find . -type d
```

**Acciones realizadas:**
Creamos la estructura de directorios completa para organizar nuestra aplicación multiambiente. La estructura sigue las mejores prácticas de GitOps separando manifiestos base de configuraciones específicas por entorno.

**Estructura creada:**
```
multi-env-pipeline-lab/
├── README.md
├── apps/webapp/               # Manifiestos base de la aplicación
├── environments/              # Configuraciones específicas por entorno
│   ├── dev/
│   ├── staging/
│   └── prod/
├── .github/                   # GitHub Actions workflows
│   ├── workflows/
│   └── actions/deploy/
└── scripts/                   # Scripts auxiliares
```

**Explicación:**
Esta estructura permite separar claramente los manifiestos base reutilizables de las configuraciones específicas de cada entorno usando Kustomize, facilitando el mantenimiento y la escalabilidad.

### Paso 2.2: Crear manifiestos base de la aplicación

**Archivos creados:**
- `apps/webapp/deployment.yaml`: Deployment de Kubernetes con 2 réplicas base
- `apps/webapp/service.yaml`: Service tipo ClusterIP para exponer la aplicación
- `apps/webapp/configmap.yaml`: ConfigMap con configuración base
- `apps/webapp/kustomization.yaml`: Configuración base de Kustomize

**Características implementadas:**
- **Aplicación**: nginxdemos/hello (aplicación web de demostración)
- **Recursos**: Limits y requests de CPU/memoria definidos
- **Health checks**: Liveness y readiness probes configurados
- **Variables de entorno**: Configuradas mediante ConfigMap
- **Etiquetas**: Sistema de etiquetado consistente para identificación

**Configuración base:**
- Réplicas: 2
- Imagen: nginxdemos/hello:latest
- Puerto: 80
- Variables: ENVIRONMENT, VERSION, LOG_LEVEL, FEATURE_FLAG_NEW_UI

### Paso 2.3: Crear superposiciones por entorno

**Comandos de verificación ejecutados:**
```bash
kubectl kustomize environments/dev
kubectl kustomize environments/staging
kubectl kustomize environments/prod
```

**Configuraciones por entorno:**

**Development (dev):**
- Namespace: dev
- Prefijo: dev-
- Réplicas: 1
- Variables: ENVIRONMENT=development, VERSION=dev-latest, LOG_LEVEL=debug
- Feature flags: NEW_UI=true

**Staging (staging):**
- Namespace: staging  
- Prefijo: staging-
- Réplicas: 2
- Variables: ENVIRONMENT=staging, VERSION=1.0.0-rc, LOG_LEVEL=info
- Feature flags: NEW_UI=true

**Production (prod):**
- Namespace: prod
- Prefijo: prod-
- Réplicas: 3
- Variables: ENVIRONMENT=production, VERSION=1.0.0, LOG_LEVEL=warn
- Feature flags: NEW_UI=true

**Resultados obtenidos:**
Todas las configuraciones se compilaron correctamente con Kustomize. Cada entorno genera manifiestos únicos con:
- Nombres de recursos prefijados por entorno
- Configuraciones específicas en ConfigMaps
- Número de réplicas apropiado para cada entorno
- Etiquetas de entorno para identificación

**Problema encontrado:**
Recibimos advertencias sobre `bases` siendo deprecated. En versiones futuras deberemos usar `resources` en su lugar, pero funciona correctamente.

### Paso 2.4: Crear scripts auxiliares

**Archivo creado:**
- `scripts/health-check.sh`: Script de verificación de salud para deployments

**Comandos ejecutados:**
```bash
chmod +x scripts/health-check.sh
find . -name "*.yaml" -o -name "*.sh" | sort
```

**Funcionalidades del script:**
- **Parámetros**: namespace y service-name
- **Verificaciones**: Estado de pods y conectividad del servicio
- **Reintentos**: 30 intentos con intervalo de 5 segundos
- **Salida**: Mensajes informativos con emojis y códigos de exit apropiados

**Uso:**
```bash
./scripts/health-check.sh <namespace> <service-name>
```

**Explicación:**
Este script será esencial en nuestras canalizaciones CI/CD para verificar que los despliegues se completaron exitosamente antes de proceder a siguientes etapas.

---

## Resumen de la Parte 2

Completamos exitosamente la creación de la aplicación y configuraciones multiambiente:

1. ✅ **Estructura del proyecto**: Directorios organizados siguiendo mejores prácticas de GitOps
2. ✅ **Manifiestos base**: Deployment, Service y ConfigMap con configuraciones robustas
3. ✅ **Configuraciones por entorno**: Kustomize configurado para dev, staging y prod
4. ✅ **Scripts auxiliares**: Health check para verificación de despliegues

Nuestro proyecto tiene una base sólida para implementar canalizaciones CI/CD multiambiente con diferentes configuraciones por entorno usando Kustomize.

---

## Parte 3: Configurar un runner autohospedado de GitHub

### Paso 3.2: Crear el directorio del runner

**Comandos ejecutados:**
```bash
mkdir -p ~/actions-runner && cd ~/actions-runner
pwd
```

**Acciones realizadas:**
Creamos un directorio dedicado para el runner de GitHub Actions en el directorio home del usuario. Este directorio contendrá todos los archivos necesarios para el funcionamiento del runner autohospedado.

**Resultado:**
Directorio creado en `/home/ctag/actions-runner` y navegación exitosa al mismo.

### Paso 3.3: Descargar el runner

**Comandos ejecutados:**
```bash
curl -o actions-runner-linux-x64-2.329.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.329.0/actions-runner-linux-x64-2.329.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.329.0.tar.gz
ls -la
```

**Acciones realizadas:**
Descargamos la versión 2.329.0 del runner de GitHub Actions para Linux x64 desde los releases oficiales de GitHub. Extraímos el archivo comprimido obteniendo todos los archivos ejecutables necesarios.

**Archivos obtenidos:**
- `config.sh`: Script de configuración del runner
- `run.sh`: Script para ejecutar el runner
- `svc.sh`: Script para gestión como servicio
- Directorios `bin/` y `externals/` con binarios necesarios

### Paso 3.4: Configurar el runner

**Comando ejecutado:**
```bash
./config.sh --url https://github.com/ignrui/multi-env-pipeline-lab --token BTMDJJGUXPDNK6JK24HEVP3JBW4GS
```

**Configuración aplicada:**
- **Runner group**: Default (por defecto)
- **Runner name**: LAB0037 (hostname del sistema)
- **Labels**: self-hosted, Linux, X64 (automáticas)
- **Work folder**: _work (por defecto)

**Resultados obtenidos:**
- Conexión exitosa a GitHub
- Runner registrado correctamente en el repositorio
- Configuración guardada en archivos `.credentials`, `.runner`, etc.

**Explicación:**
El runner queda registrado en nuestro repositorio específico y podrá ejecutar workflows que especifiquen `runs-on: self-hosted`.

### Paso 3.5: Iniciar el runner

**Comando ejecutado:**
```bash
cd ~/actions-runner && ./run.sh
```

**Acciones realizadas:**
Iniciamos el runner en modo interactivo en un terminal separado para mantenerlo funcionando durante todo el laboratorio. El runner se conecta a GitHub y queda en estado de escucha.

**Estado del runner:**
- Conectado a GitHub exitosamente
- Versión 2.329.0 funcionando
- Estado: "Listening for Jobs"
- ✅Ejecutándose en terminal dedicado (no interrumpible)

### Paso 3.8: Probar el runner

**Archivo creado:**
- `.github/workflows/test-runner.yaml`: Workflow de prueba para verificar conectividad

**Comandos ejecutados:**
```bash
git add .github/workflows/test-runner.yaml
git commit -m "Add runner test workflow"
git push origin main
```

**Funcionalidades del workflow de prueba:**
- **Trigger**: `workflow_dispatch` (ejecución manual)
- **Runner**: `runs-on: self-hosted`
- **Verificaciones**:
  - Información del entorno del runner
  - Acceso a kubectl y configuración
  - Listado de namespaces de Kubernetes
  - Verificación de entornos dev, staging, prod
  - Mensaje de éxito

**Resultado:**
Workflow creado y subido al repositorio, listo para ejecutar manualmente desde la interfaz de GitHub Actions.

**Ejecución del workflow de prueba:**
```
2025-11-07 08:20:15Z: Listening for Jobs
2025-11-07 08:21:32Z: Running job: Test Runner & Cluster Access
2025-11-07 08:21:41Z: Job Test Runner & Cluster Access completed with result: Succeeded
```

**Verificación exitosa:**
- El runner detectó y procesó el job automáticamente
- Duración: ~9 segundos (08:21:32 - 08:21:41)
- Estado final: **Succeeded**
- Conectividad GitHub ↔ Runner ↔ Kubernetes confirmada

---

## Resumen de la Parte 3

Completamos exitosamente la configuración del runner autohospedado:

1. **Runner instalado**: Versión 2.329.0 en ~/actions-runner
2. **Conectividad establecida**: Runner registrado y conectado a GitHub
3. **Runner activo**: Ejecutándose en terminal dedicado, escuchando trabajos
4. **Workflow de prueba**: Creado para verificar acceso al clúster Minikube
5. **Acceso local al clúster**: El runner tiene acceso directo a Minikube vía kubectl

**Arquitectura lograda:**
```
GitHub Repository → GitHub Actions Workflow → Self-hosted Runner → Minikube Cluster
```

Nuestro runner autohospedado resuelve el problema de conectividad entre GitHub Actions (nube) y nuestro clúster Minikube (local), permitiendo que los workflows ejecuten comandos kubectl directamente contra nuestro entorno de desarrollo.

**Validación completa realizada:**
La ejecución exitosa del workflow "Test Runner & Cluster Access" confirma que:
- Runner operativo y registrado en GitHub
- Conectividad bidireccional GitHub ↔ Runner establecida  
- Acceso directo del runner al clúster Minikube via kubectl
- Namespaces dev, staging y prod detectados correctamente
- Entorno listo para canalizaciones CI/CD multiambiente

---

## Próximos pasos

**Prueba del runner ejecutada exitosamente**

El workflow de prueba se ejecutó correctamente:
- **Inicio**: 2025-11-07 08:21:32Z  
- **Finalización**: 2025-11-07 08:21:41Z
- **Duración**: 9 segundos
- **Estado**: Succeeded
- **Verificaciones completadas**: Información del runner, acceso kubectl, namespaces, entornos del laboratorio

---

## Estado General del Laboratorio

Hemos completado exitosamente las primeras 3 partes del laboratorio:

### **Parte 1 - Preparación del entorno**
- Clúster Minikube funcionando (v1.34.0)
- Namespaces dev, staging, prod creados y etiquetados
- Repositorio GitHub configurado
- Entorno local preparado

### **Parte 2 - Aplicación y configuraciones**  
- Estructura GitOps implementada
- Manifiestos base de Kubernetes creados
- Configuraciones multiambiente con Kustomize
- Scripts de health-check implementados

### **Parte 3 - Runner autohospedado**
- Runner v2.329.0 instalado y configurado
- Conectividad GitHub ↔ Minikube establecida
- Workflow de prueba ejecutado exitosamente
- Entorno listo para CI/CD automatizado

---

## Parte 4: Crear flujos de trabajo de GitHub Actions

### Paso 4.1: Crear pipeline CI/CD principal

**Archivo creado:**
- `.github/workflows/pipeline.yaml`: Pipeline multiambiente completo

**Comandos ejecutados:**
```bash
cat > .github/workflows/pipeline.yaml << 'EOF'
# Contenido del pipeline con jobs para dev, staging y production
EOF
```

**Características del pipeline:**
- **Trigger**: Push a main en rutas específicas (`apps/**`, `environments/**`, `.github/workflows/**`)
- **Runner**: Todos los jobs usan `runs-on: self-hosted`
- **Jobs progresivos**: deploy-dev → deploy-staging → deploy-prod
- **Entornos GitHub**: Configuración para dev, staging y production
- **Aprobación manual**: Requerida para el entorno production

**Estructura del workflow:**
- **Job 1**: Deploy a Dev (automático)
- **Job 2**: Deploy a Staging (requiere dev)  
- **Job 3**: Deploy a Producción (requiere staging + aprobación)

### Paso 4.2: Verificar el archivo del workflow

**Comandos ejecutados:**
```bash
ls -la .github/workflows/
cat .github/workflows/pipeline.yaml
git add .github/workflows/pipeline.yaml
git commit -m "Add multi-environment CI/CD pipeline"
git push origin main
```

**Verificación exitosa:**
- Archivo pipeline.yaml creado correctamente
- Sintaxis YAML válida
- Estructura de jobs y steps apropiada
- Referencia correcta a entornos GitHub

### Resolución de problemas encontrados

**Problema 1: Advertencias deprecated**
- **Issue**: Kustomization usaba `bases` (deprecated)
- **Solución**: Reemplazado con `resources` en todos los environments
- **Resultado**: Eliminadas las advertencias de Kustomize

**Problema 2: Archivos faltantes en repositorio**
- **Issue**: Error "no such file or directory" en directorio `apps/`
- **Causa**: No se hizo `git add .` para incluir todos los archivos creados en la Parte 2
- **Debug**: Añadidos comandos de verificación en pipeline para diagnosticar
- **Solución**: `git add .` para incluir apps/ y scripts/ completos
- **Archivos añadidos**:
  - `apps/webapp/deployment.yaml`
  - `apps/webapp/service.yaml` 
  - `apps/webapp/configmap.yaml`
  - `apps/webapp/kustomization.yaml`
  - `scripts/health-check.sh`

**Resultado:**
Pipeline creado y listo para ejecutarse una vez que se configuren los entornos de GitHub.

---

## Resumen de la Parte 4

Completamos exitosamente la creación de flujos de trabajo de GitHub Actions:

### **Logros conseguidos:**
1. **Pipeline CI/CD**: Workflow multiambiente definido correctamente
2. **Estructura progresiva**: Jobs secuenciales dev → staging → production
3. **Runner autohospedado**: Todos los jobs configurados para self-hosted
4. **Debugging implementado**: Comandos de verificación para troubleshooting
5. **Repositorio completo**: Todos los manifiestos y scripts incluidos

### **Preparación completada para:**
- Configuración de entornos GitHub (Parte 5)
- Ejecución del pipeline completo
- Gestión de aprobaciones manuales
- Verificación de despliegues multiambiente

---

## Parte 5: Configurar entornos de GitHub

### ¿Por qué reglas de protección de entornos?

Aunque el runner tiene acceso directo al clúster, implementamos aprobaciones humanas para producción. Los entornos de GitHub ofrecen:

**Aprobación manual** para deployments críticos
**Temporizadores de espera** para ventanas de despliegue controladas
**Restricciones por rama** para asegurar que solo main despliega a prod
**Historial y auditoría** completa de todos los despliegues

### Paso 5.1: Crear entorno Dev (sin protección)

**Acciones realizadas en GitHub:**
1. Navegación: Settings → Environments
2. Clic en "New environment"
3. Nombre: `dev`
4. Configuración: Sin reglas de protección
5. Resultado: Entorno dev creado para despliegue automático

### Paso 5.2: Crear entorno Staging (protección opcional)

**Acciones realizadas en GitHub:**
1. Clic en "New environment"
2. Nombre: `staging` 
3. Configuración: Sin reglas de protección (opcional en laboratorio)
4. Resultado: Entorno staging configurado para despliegue automático

**Nota:** En entornos productivos, staging podría requerir aprobaciones adicionales.

### Paso 5.3: Crear entorno Producción (requiere aprobación)

**Configuración aplicada:**
- ☑️ **Required reviewers**: Usuario GitHub añadido como revisor requerido
- ☑️ **Wait timer**: 0 minutos (despliegue inmediato tras aprobación)
- ☑️ **Deployment branches**: Solo rama `main` autorizada para despliegue
- **Resultado**: Entorno production protegido con aprobación manual obligatoria

### Paso 5.4: Verificar la configuración de entornos

**Estado final de entornos:**

| Entorno | Reglas de protección | Estado |
|---------|---------------------|--------|
| dev | Ninguna | Listo para despliegue automático |
| staging | Ninguna | Listo para despliegue automático |  
| production | Revisores requeridos, Rama: main | Protegido con aprobación manual |

### Paso 5.5: Flujo de despliegue implementado

**Arquitectura del pipeline:**
```
Push a la rama main
  ↓
Deploy a Dev (automático, runner autohospedado)
  ↓ 
Deploy a Staging (automático, runner autohospedado)
  ↓
Deploy a Producción (PAUSADO - esperando aprobación)
  ↓
Aprobación manual en GitHub UI
  ↓
Deploy a Producción (ejecutado por runner autohospedado)
  ↓
¡Completado!
```

### Ejecución exitosa del pipeline completo

**Timeline de la ejecución real:**
```
2025-11-07 08:44:29Z: Job Deploy a Dev completed with result: Succeeded
2025-11-07 08:44:31Z: Running job: Deploy a Staging  
2025-11-07 08:44:50Z: Job Deploy a Staging completed with result: Succeeded
2025-11-07 08:46:28Z: Running job: Deploy a Producción
2025-11-07 08:46:49Z: Job Deploy a Producción completed with result: Succeeded
```

**Proceso completado:**
1. **Deploy a Dev**: Automático - 29 segundos
2. **Deploy a Staging**: Automático tras dev - 19 segundos
3. **Pausa para aprobación**: Pipeline esperando aprobación manual 
4. **Aprobación concedida**: Aprobación manual realizada en GitHub UI
5. **Deploy a Producción**: Completado - 21 segundos

**Verificación final de despliegues:**
- **Dev namespace**: 1 réplica funcionando
- **Staging namespace**: 2 réplicas funcionando  
- **Production namespace**: 3 réplicas funcionando
- **Health checks**: Todos los rollouts exitosos
- **Configuraciones**: Variables específicas por entorno aplicadas correctamente

---

## Resumen de la Parte 5

Configuramos exitosamente los entornos de GitHub y ejecutamos el pipeline completo:

### **Entornos configurados:**
1. **Dev**: Sin protección - Despliegue automático
2. **Staging**: Sin protección - Despliegue automático  
3. **Production**: Protegido - Requiere aprobación manual + rama main

### **Pipeline ejecutado exitosamente:**
- **Flujo progresivo**: dev → staging → production funcionando
- **Gates de aprobación**: Pausa manual en production implementada
- **Runner autohospedado**: Todos los jobs ejecutados correctamente
- **Verificaciones**: Health checks y rollouts completados
- **Escalabilidad**: Diferentes réplicas por entorno (1, 2, 3)

---

## Estado Final del Laboratorio

### **Infraestructura completada:**

**Parte 1 - Preparación**
- Minikube v1.34.0 funcionando
- Namespaces dev, staging, prod configurados
- Repositorio GitHub establecido

**Parte 2 - Aplicación**  
- Manifiestos Kubernetes base creados
- Configuraciones Kustomize por entorno
- Scripts de verificación implementados

**Parte 3 - Runner autohospedado**
- Runner v2.329.0 instalado y conectado
- Acceso directo GitHub Actions ↔ Minikube
- Workflows de prueba validados

**Parte 4 - CI/CD Multiambiente**
- Pipeline completo funcionando
- Despliegues automáticos progresivos
- Aprobaciones manuales implementadas
- **3 entornos desplegados y verificados**


Hemos implementado una solución GitOps completa con canalizaciones CI/CD multiambiente, que incluye:
- Separación por entornos con configuraciones específicas
- Despliegues automáticos con gates de aprobación
- Verificaciones de salud automatizadas  
- Runner autohospedado para conectividad local
- Monitorización en tiempo real de deployments

El sistema está listo para desarrollo continuo y puede escalarse para casos de uso reales.

---

## Punto de control final

A este punto, hemos completado exitosamente:

**Runner autohospedado operativo** - Conectado y funcionando
**Pipeline de GitHub Actions** disparado y completado múltiples veces  
**Despliegue en dev** automático funcionando
**Despliegue en staging** automático funcionando
**Despliegue en producción** tras aprobación manual completado
**Todos los pods en ejecución** en los tres namespaces
**Configuraciones específicas por entorno** aplicadas correctamente

### Verificación final de deployments:

```bash
kubectl get pods -A | grep webapp
```

**Resultado:**
```
dev           dev-webapp-5b7cf884bb-b2fqq        1/1     Running   0          23m
prod          prod-webapp-6c8f78db8b-2brtm       1/1     Running   0          12m  
prod          prod-webapp-6c8f78db8b-f5d25       1/1     Running   0          12m
prod          prod-webapp-6c8f78db8b-p8lxw       1/1     Running   0          12m
staging       staging-webapp-7478bd4fcb-gwzgs    1/1     Running   0          14m
staging       staging-webapp-7478bd4fcb-mnqhd    1/1     Running   0          14m
```

**Total: 6 pods funcionando correctamente** - 1 en dev, 2 en staging, 3 en production

---

## Entregables del laboratorio

### Capturas de pantalla requeridas:

**Workflow de GitHub Actions** mostrando todas las etapas completadas  
![Workflow Success](./Images/workflow%20succes.png)
*Figura 1: Pipeline multiambiente ejecutado exitosamente con todos los jobs completados*

**Paso de aprobación** del despliegue a producción  
![Workflow Approval](./Images/Wait%20workflow%20aproval.png)
*Figura 2: Gate de aprobación manual para despliegue a producción*

**kubectl get pods -A | grep webapp** mostrando los tres entornos  
**Comando ejecutado:** 6 pods corriendo distribuidos en dev (1), staging (2), prod (3)
```bash
dev           dev-webapp-5b7cf884bb-b2fqq        1/1     Running   0          23m
prod          prod-webapp-6c8f78db8b-2brtm       1/1     Running   0          12m  
prod          prod-webapp-6c8f78db8b-f5d25       1/1     Running   0          12m
prod          prod-webapp-6c8f78db8b-p8lxw       1/1     Running   0          12m
staging       staging-webapp-7478bd4fcb-gwzgs    1/1     Running   0          14m
staging       staging-webapp-7478bd4fcb-mnqhd    1/1     Running   0          14m
```

**Página de resumen de GitHub Actions** mostrando despliegue exitoso  
![Actions Summary](./Images/actions%20summary.png)
*Figura 3: Resumen de GitHub Actions con historial de ejecuciones exitosas*

**Test del runner autohospedado** verificando conectividad  
![Test Self Hosted](./Images/Test%20self%20hosted.png)
*Figura 4: Verificación exitosa del runner autohospedado con acceso a Kubernetes*

### Análisis de las capturas:

**Figura 1 - Workflow Success:** Muestra la ejecución completa del pipeline "Multi-Environment CI/CD Pipeline" con todos los jobs (Deploy a Dev, Deploy a Staging, Deploy a Producción) completados exitosamente. Se aprecia el flujo progresivo y la duración de cada etapa.

**Figura 2 - Wait Workflow Approval:** Captura del momento crítico donde el pipeline se pausa antes del despliegue a producción, esperando la aprobación manual del revisor. Esto demuestra el gate de seguridad implementado para el entorno productivo.

**Figura 3 - Actions Summary:** Vista general del historial de ejecuciones de GitHub Actions, mostrando múltiples runs exitosos del pipeline, incluyendo tanto el workflow de prueba del runner como las ejecuciones completas del pipeline multiambiente.

**Figura 4 - Test Self Hosted:** Validación inicial del runner autohospedado mostrando la verificación exitosa de:
- Información del runner (hostname, OS, directorio)
- Acceso a kubectl y configuración de Kubernetes
- Listado de namespaces disponibles
- Confirmación de entornos dev, staging y prod

### Arquitectura final implementada:

```
GitHub Repository (multi-env-pipeline-lab)
    ↓ (push a main)
GitHub Actions Pipeline (Multi-Environment CI/CD)
    ↓ (ejecuta en)
Self-hosted Runner v2.329.0 (WSL Ubuntu)
    ↓ (despliega via kubectl + kustomize)
Minikube Cluster v1.34.0
    ├── dev namespace: 1 pod webapp (development config)
    ├── staging namespace: 2 pods webapp (staging config)  
    └── prod namespace: 3 pods webapp (production config)
```

### Configuraciones específicas verificadas:

| Entorno | Réplicas | Variables | Feature Flags | Log Level |
|---------|----------|-----------|---------------|-----------|
| dev | 1 | VERSION=dev-latest | NEW_UI=true | debug |
| staging | 2 | VERSION=1.0.0-rc | NEW_UI=true | info |
| prod | 3 | VERSION=1.0.0 | NEW_UI=true | warn |

