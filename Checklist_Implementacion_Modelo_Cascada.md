# Checklist Detallado y Avanzado: Implementación de Proyecto en GCP con Cloud Run, GitHub Actions y Cloudflare

Este checklist es una guía secuencial para la implementación de servicios contenedorizados en GCP Cloud Run, automatizada con GitHub Actions y expuesta vía Cloudflare. Se basa en principios de arquitectura cloud robusta, seguridad (PoLP, WIF, Secret Management), automatización y observabilidad.

## Principios y Consideraciones Clave

- **Principio de Mínimo Privilegio (PoLP):** Otorgar únicamente los permisos necesarios a identidades y servicios.  
- **Workload Identity Federation (WIF):** Método de autenticación preferido para cargas de trabajo externas (CI/CD) sin intercambio de claves estáticas.  
- **Cloud Run Service Account (Runtime SA):** Identidad bajo la cual se ejecuta el contenedor, crucial para el acceso seguro a otros servicios GCP (Secret Manager, Databases, etc.).  
- **12-Factor App:** Configuración gestionada externamente (variables de entorno/secretos), Logs como streams de eventos (stdout/stderr), Procesos stateless.  
- **Inmutabilidad del Contenedor:** La imagen Docker es la unidad de despliegue; la configuración se inyecta externamente.  
- **Observabilidad:** Integración con Cloud Logging y Cloud Monitoring para visibilidad operativa.

> Fase Modelo Cascada: PLANIFICACION/REQUISITOS
# Fase 0: Planificación y Diseño Arquitectural

## 0.1. Definición del Alcance y la Estructura de la Aplicación:

1. Identificar la(s) carga(s) de trabajo a desplegar (Frontend, Backend(s), Microservicio(s)).  
2. Determinar si se desplegará como un único servicio Cloud Run o múltiples servicios interconectados.  
3. Definir la(s) tecnología(s) de la aplicación (lenguaje, framework).

## 0.2. Diseño de la Arquitectura en GCP y Relación con Cloudflare:

1. Esbozar la topología de los servicios (Cloud Run, bases de datos, Pub/Sub, etc.) y flujos de interacción.  
2. Determinar la estrategia de exposición: Acceso público vía Cloudflare \-\> Cloud Run o acceso privado (Internal) con Load Balancer/VPC. (El checklist se centra en acceso público vía Cloudflare).  
3. Definir la estructura de repositorios de código en GitHub.

## 0.3. Establecimiento de Estándares de Nomenclatura:

1. Definir convenciones de nombres para: ID del Proyecto GCP, IDs de Servicios Cloud Run, Nombres de Repositorios en GitHub, Nombres de Repositorios en Artifact Registry, Nombres de Secretos en Secret Manager, Tags de Imágenes Docker, Nombres de Service Accounts.

## 0.4. Definición de Entornos de Despliegue y Configuración por Entorno:

1. Especificar los entornos (ej. development, staging, production).  
2. Identificar la configuración específica (variables de entorno, secretos, endpoints de servicios externos) para cada entorno.

## 0.5. Diseño Detallado de IAM y Estrategia de Seguridad:

1. Definir el Service Account dedicado para la pipeline CI/CD (con permisos de despliegue).  
2. Definir el Service Account dedicado para el runtime de cada servicio Cloud Run (con permisos para acceder a secretos, bases de datos, otros APIs de GCP). Evitar usar el Default Compute Engine Service Account en producción.  
3. Planificar la implementación de Workload Identity Federation (WIF) para la autenticación de GitHub Actions en GCP.  
4. Identificar los secretos necesarios para cada servicio y entorno, y planificar su gestión en Secret Manager.

## 0.6. Estimación de Requisitos de Recursos y Escalabilidad:

1. Estimar la carga de tráfico esperada.  
2. Determinar la configuración de CPU, Memoria, Mín/Máx instancias y Concurrencia para Cloud Run por entorno.

## 0.7. Selección de Región(es) GCP:

1. Elegir la región o regiones para desplegar los recursos GCP.

> Fase Modelo Cascada: DISEÑO DEL SISTEMA Y SOFTWARE
# Fase 1: Preparación del Entorno GCP

## 1.1. Creación y Configuración del Proyecto GCP:

1. Crear el Proyecto GCP con un ID siguiendo la convención definida (gcloud projects create \[PROJECT\_ID\]).  
2. Vincular el proyecto a la Cuenta de Facturación (gcloud billing projects link \[PROJECT\_ID\] \--billing-account=\[BILLING\_ACCOUNT\_ID\]).

## 1.2. Configuración Inicial de IAM y Service Accounts:

1. Aplicar políticas de IAM a nivel Organización/Carpeta si aplica.  
2. Crear el Service Account para CI/CD (gcloud iam service-accounts create \[CI\_SA\_ID\] \--display-name="\[CI\_SA\_NAME\]").  
3. Otorgar roles con mínimos privilegios al SA de CI/CD:  
4. roles/artifactregistry.writer (Push de imágenes a Artifact Registry).  
5. roles/run.admin (Desplegar y gestionar servicios Cloud Run).  
6. roles/secretmanager.secretAccessor (Si el CI/CD necesita leer secretos durante el despliegue para inyección \- \--set-secrets).  
7. roles/iam.serviceAccountUser (Permitir que el SA de CI/CD actúe como el SA de Runtime al desplegar).  
8. Configurar Workload Identity Federation (WIF) Provider en GCP:  
9. Crear un Pool de Workload Identity (gcloud iam workload-identity-pools create \[POOL\_ID\] \--location=global).  
10. Crear un Provider OIDC asociado al Pool para GitHub (gcloud iam workload-identity-pools providers create-oidc \[PROVIDER\_ID\] \--location=global \--workload-identity-pool=\[POOL\_ID\] \--issuer\_uri="https://token.actions.githubusercontent.com").  
11. Obtener el nombre completo del WIF Provider (formato projects/PROJECT\_NUMBER/locations/global/pools/POOL\_ID/providers/PROVIDER\_ID).  
12. Enlazar el SA de CI/CD al WIF Provider: Conceder el rol roles/iam.workloadIdentityUser al SA de CI/CD, con una condición que restrinja la federación a la identidad específica de GitHub (ej. principal://iam.googleapis.com/projects/.../locations/global/pools/.../subject/repo:owner/repo\_name:ref:refs/heads/main).  
13. Crear el Service Account dedicado para el Runtime del Servicio Cloud Run (gcloud iam service-accounts create \[RUNTIME\_SA\_ID\] \--display-name="\[RUNTIME\_SA\_NAME\]").

## 1.3. Habilitar APIs Requeridas:

1. Habilitar run.googleapis.com.  
2. Habilitar artifactregistry.googleapis.com.  
3. Habilitar secretmanager.googleapis.com.  
4. Habilitar logging.googleapis.com.  
5. Habilitar monitoring.googleapis.com.  
6. (Opcional, si se usa VPC privada) Habilitar vpcaccess.googleapis.com (Serverless VPC Access).

## 1.4. Configuración del Entorno de Desarrollo/CI Local:

1. Instalar Google Cloud SDK (gcloud) y configurar la autenticación local.  
2. Configurar el proyecto y la región por defecto para gcloud CLI.  
3. Instalar Docker.

## 1.5. Creación de Recursos GCP Base (Network y Storage):

1. Crear Repositorio en Artifact Registry para imágenes Docker (gcloud artifacts repositories create \[GAR\_REPO\_NAME\] \--repository-format=docker \--location=\[REGION\]).  
2. (Opcional) Crear Bucket(s) en Cloud Storage para logs centralizados o artefactos de build.  
3. (Opcional, si se usa VPC privada) Configurar Serverless VPC Access Connector en la VPC deseada (gcloud compute networks vpc-access connectors create \[CONNECTOR\_NAME\] \--region=\[REGION\] \--range=\[IP\_RANGE\]/28 \--network=\[VPC\_NETWORK\]).

> Fase Modelo Cascada: DISEÑO DEL SISTEMA Y SOFTWARE
# Fase 2: Estructura Inicial del Código y Dockerización

## 2.1. Creación de Repositorios en GitHub:

1. Crear el/los repositorio(s) en GitHub (owner/repo\_name) según la Fase 0\.

## 2.2. Clonación Local y Estructura de Directorios:

1. Clonar repositorios localmente.  
2. Establecer la estructura de carpetas estándar (ej. src/, tests/, config/).  
3. Crear archivos de control de versiones y build: .gitignore, .dockerignore.  
4. Crear el directorio .github/workflows/.

2.3. Desarrollo de Estructura Base de la Aplicación:

1. Implementar un esqueleto funcional mínimo (ej. Backend con un endpoint /health).  
2. Asegurar que la aplicación lee la configuración desde variables de entorno (principio 12-Factor).  
3. Asegurar que la aplicación escribe logs a stdout o stderr.

2.4. Creación y Optimización de Dockerfile(s):

1. Escribir Dockerfiles eficientes y seguros (builds multi-stage, imágenes base mínimas, usuario no-root).  
2. Asegurar que el contenedor escucha en el puerto especificado por la variable de entorno PORT (Cloud Run inyecta PORT).  
3. Exponer el puerto correcto en el Dockerfile (EXPOSE $PORT o EXPOSE 8080 si se usa 8080 por defecto).  
4. Definir un ENTRYPOINT o CMD claro.

## 2.5. Implementación de Health & Readiness Probes:

1. Implementar endpoints de estado en la aplicación (ej. /health, /ready).  
2. El endpoint /health debe indicar si la aplicación está viva.  
3. El endpoint /ready debe indicar si la aplicación está lista para recibir tráfico (dependencias inicializadas, etc.).

> Fase Modelo Cascada: IMPLEMENTACIÓN
# Fase 3: Gestión de Secretos (Valores y Acceso en Runtime)

## 3.1. Definición Lógica de Secretos en Secret Manager (revisión Fase 1.5):

1. Confirmar que los nombres de los secretos para cada configuración sensible y entorno han sido creados en Secret Manager.

## 3.2. Agregar Versiones/Valores Específicos por Entorno:

1. Para cada secreto, añadir la(s) versión(es) correspondiente(s) con los valores específicos para cada entorno (gcloud secret-manager versions add \[SECRET\_NAME\] \--data-file=/path/to/value.txt). Utilizar nombres de secreto que distingan entornos o gestionar versiones/labels.

## 3.3. Otorgar Permisos de Acceso a Secretos para el Service Account de Runtime:

1. Conceder el rol roles/secretmanager.secretAccessor al Service Account de Runtime de Cloud Run (creado en Fase 1.2) sobre cada secreto al que necesita acceder (gcloud secrets add-iam-policy-binding \[SECRET\_NAME\] \--role="roles/secretmanager.secretAccessor" \--member="serviceAccount:\[RUNTIME\_SA\_EMAIL\]").

> Fase Modelo Cascada: IMPLEMENTACIÓN
# Fase 4: Implementación de la Pipeline CI/CD (GitHub Actions)

## 4.1. Configuración de Secretos en GitHub Repository:

1. En GitHub (Settings \> Secrets \> Actions), añadir secretos del repositorio:  
   1. GCP\_PROJECT\_ID  
   2. GCP\_REGION  
   3. CLOUD\_RUN\_SERVICE\_NAME  
   4. GAR\_REPO\_NAME  
   5. IMAGE\_NAME (nombre base de la imagen)  
   6. GCP\_WORKLOAD\_IDENTITY\_PROVIDER (nombre completo del WIF Provider en GCP)  
   7. GCP\_SERVICE\_ACCOUNT (email del Service Account de CI/CD)  
   8. RUNTIME\_SERVICE\_ACCOUNT\_EMAIL (email del Service Account de Runtime de Cloud Run)

4.2. Creación del Archivo de Workflow (deploy.yml):

1. Crear el archivo YAML en .github/workflows/deploy.yml.  
2. Definir los eventos que disparan el workflow (ej. push a main, release).

## 4.3. Implementación de los Pasos del Workflow:

1. **Checkout:** Usar actions/checkout@v4.  
2. **Autenticación en GCP (WIF):** Usar google-github-actions/auth@v2 con workload\_identity\_provider y service\_account.  
3. **Configurar Docker para Artifact Registry:** Usar google-github-actions/setup-gcloud@v2 y configurar Docker credential helper (docker-credential-gcr configure-docker \[REGION\]-docker.pkg.dev).  
4. **Build de Imagen Docker:** Ejecutar docker build \-t \[REGION\]-docker.pkg.dev/${{ secrets.GCP\_PROJECT\_ID }}/${{ secrets.GAR\_REPO\_NAME }}/${{ secrets.IMAGE\_NAME }}:${{ github.sha }} ..  
5. **Push de Imagen Docker:** Ejecutar docker push \[REGION\]-docker.pkg.dev/${{ secrets.GCP\_PROJECT\_ID }}/${{ secrets.GAR\_REPO\_NAME }}/${{ secrets.IMAGE\_NAME }}:${{ github.sha }}.  
6. **Despliegue en Cloud Run:**  
   1. Ejecutar  
      1. `gcloud run deploy ${{ secrets.CLOUD_RUN_SERVICE_NAME }} --image=[REGION]-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/${{ secrets.GAR_REPO_NAME }}/${{ secrets.IMAGE_NAME }}:${{ github.sha }} --region=${{ secrets.GCP_REGION }} --platform=managed.`  
   2. Especificar el Service Account de Runtime (\--service-account=${{ secrets.RUNTIME\_SERVICE\_ACCOUNT\_EMAIL }}).  
   3. Configurar la inyección de secretos desde Secret Manager (\--set-secrets=ENV\_VAR\_NAME=SECRET\_NAME:latest,...).  
   4. Configurar otras variables de entorno no secretas (\--set-env-vars=VAR\_NAME=value,...).  
   5. Configurar recursos y escalabilidad (-\-cpu, \--memory, \--min-instances, \--max-instances, \--concurrency).  
   6. Configurar políticas de acceso (\--allow-unauthenticated para acceso público; \--ingress=internal-and-cloud-load-balancing si se usa Load Balancer/VPC).  
   7. (Opcional, si se usa VPC privada) Configurar conexión VPC (\--vpc-connector=\[CONNECTOR\_NAME\] \--vpc- egress=all-traffic).  
   8. Configurar Liveness Probe (-\-liveness-probe-path=\[PATH\] \--liveness-probe-timeout=\[TIMEOUT\]s).  
   9. Configurar Readiness Probe (\--readiness-probe-path=\[PATH\] \--readiness-probe-timeout=\[TIMEOUT\]s).  
7. (Opcional) Añadir pasos para Tests Automatizados (Unitarios, Integración).  
8. (Opcional) Configurar notificaciones de estado del workflow.

> Fase Modelo Cascada: PRUEBAS
# Fase 5: Primer Despliegue y Validación Inicial en GCP

# 5.1. Disparo de la Pipeline CI/CD:

1. Ejecutar el evento que dispara el workflow (ej. git push).

## 5.2. Monitoreo de la Ejecución del Workflow:

1. Seguir el progreso en GitHub Actions UI.

## 5.3. Verificación del Despliegue y Configuración en GCP:

1. Ir a la consola de Cloud Run.  
2. Confirmar que el servicio ha sido desplegado y una nueva revisión está activa.  
3. Usar `gcloud run` services describe \[SERVICE\_NAME\] \--region=\[REGION\] para validar:  
   1. Imagen del contenedor desplegada.  
   2. Service Account de Runtime correcto.  
   3. Variables de entorno y secretos inyectados correctamente.  
   4. Configuración de recursos, escalabilidad, probes.  
   5. Conector VPC si aplica.

## 5.4. Obtención y Prueba de la URL por Defecto de Cloud Run:

1. Anotar la URL generada por Cloud Run (ej. https://\[SERVICE\_HASH\]-\[PROJECT\_ID\].\[REGION\].run.app).  
2. Probar el acceso directo a esta URL (usando curl o navegador) para validar que el contenedor inicia, responde y pasa los probes.  
3. Probar los endpoints /health y /ready directamente si son accesibles.

> Fase Modelo Cascada: DESPLIEGUE
# Fase 6: Configuración de Dominio y Seguridad Perimetral (Cloudflare)

## 6.1. Delegación de Dominio/Subdominio a Cloudflare (si aplica):

Actualizar los servidores DNS del dominio en el registrador para usar los de Cloudflare.

## 6.2. Creación de Registros DNS en Cloudflare:

Crear un registro CNAME (Proxy habilitado \- "nube naranja") en Cloudflare (ej. www o api) apuntando a la URL generada por Cloud Run (ej. \[SERVICE\_HASH\]-\[PROJECT\_ID\].\[REGION\].run.app).

## 6.3. Configuración de SSL/TLS en Cloudflare:

En SSL/TLS \> Overview, configurar el modo "Full (Strict)". (Cloud Run provee certificado managed).  
Verificar que el certificado en el dominio de Cloudflare es válido.

## 6.4. Configuración de Reglas de Seguridad (WAF, Firewall):

Habilitar y configurar reglas gestionadas de WAF.  
Configurar reglas de firewall personalizadas (ej. rate limiting, bloqueo por IP/país).

## 6.5. Configuración de Reglas de Página (Page Rules):

Configurar redirecciones (ej. HTTP a HTTPS), caching, optimizaciones de rendimiento por path, si es necesario.

## 6.6. Verificación Final del Acceso vía Dominio:

Esperar la propagación de DNS.  
Acceder al servicio a través del dominio configurado (ej. https://\[your-domain.com\]).  
Validar que el acceso funciona correctamente y que Cloudflare está actuando como proxy (verificando headers como cf-ray).

> Fase Modelo Cascada: MANTENIMIENTO
# Fase 7: Validación Operativa, Monitoreo y Mantenimiento

## 7.1. Validación Funcional Completa a Través del Dominio:

1. Ejecutar pruebas de aceptación/integración completas sobre el servicio desplegado a través del dominio de Cloudflare.

## 7.2. Configuración y Verificación de Logging:

1. Confirmar que los logs de la aplicación aparecen en Cloud Logging.  
2. Utilizar el Explorador de Logs para filtrar y analizar los logs.  
3. Configurar Log Sinks a destinos de almacenamiento/análisis a largo plazo (ej. BigQuery, GCS) o a sistemas de gestión de logs centralizados.

## 7.3. Configuración de Monitoreo y Alertas (Cloud Monitoring):

1. Crear Paneles (Dashboards) personalizados con métricas clave de Cloud Run (request count, latency p50/p95/p99, error count/rate 4xx/5xx, instance count, CPU/memory utilization).  
2. Crear Políticas de Alerta sobre umbrales críticos de estas métricas (ej. Latencia \> X, Tasa de errores \> Y%, Instancias \= 0 inesperadamente).  
3. Configurar periodos de evaluación bajos (ej. 1-5 minutos) para la detección temprana de problemas.  
4. Configurar canales de notificación (email, Slack, Pub/Sub, Webhooks).

## 7.4. Configuración de Autoscaling Basada en Carga:

1. Ajustar los parámetros de escalabilidad (--min-instances, \--max-instances, \--concurrency) basándose en pruebas de carga y la Fase 0\. Considerar min-instances=0 para ahorro de costos vs. min-instances\>0 para menor latencia en el primer request.

## 7.5. Documentación Técnica y Operativa:

1. Documentar la arquitectura final desplegada, incluyendo Service Accounts, permisos, configuración de Cloud Run, Flujos de CI/CD y configuración de Cloudflare.  
2. Crear procedimientos operativos estándar (SOPs) para monitoreo, resolución de problemas comunes, y procesos de despliegue/rollback.

## 7.6. Planificación y Prueba de Procedimientos de Rollback:

1. Definir cómo revertir a una revisión anterior estable en caso de un despliegue fallido (usando la capacidad de Cloud Run de mantener revisiones).  
2. Probar este procedimiento.

## 7.7. Considerar la Migración a IaC (Opcional, para Producción):

1. Evaluar la refactorización de la configuración de infraestructura de GCP (Proyecto, IAM, Artifact Registry, Cloud Run Service, VPC Connector) utilizando herramientas IaC como Terraform o Pulumi para una gestión más versionada, reproducible y auditable.

