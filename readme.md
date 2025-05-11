# Repositorio GitOps de WellTrack

Este repositorio almacena las configuraciones de Kubernetes específicas de la aplicación (charts de Helm) para el proyecto WellTrack, sirviendo como el **estado deseado** para los despliegues gestionados por Argo CD.

## Descripción General de la Arquitectura

Este repositorio GitOps es parte de una estrategia de despliegue orquestada por un repositorio **DevOps** separado ([PTI-WellTrackDevops](https://github.com/hongda-zhu/PTI-WellTrackDevops)) que implementa el patrón "App of Apps" de Argo CD.

El flujo general incluye:

- **Repositorios de Aplicación** (ej., [PTI-WellTrackBack](https://github.com/hongda-zhu/PTI-WellTrackBack), [PTI-WellTrackFront](https://github.com/hongda-zhu/PTI-WellTrackFront)): Contienen el código fuente de la aplicación.
- **GitHub Actions**: Se ejecutan en los repositorios de aplicación para construir/publicar imágenes Docker (ej., a GHCR o Harbor) y actualizar el `values.yaml` del chart de Helm correspondiente en *este* repositorio GitOps.
- **Este Repositorio (`PTI-WellTrackGitOps`)**: Contiene los charts de Helm que definen el estado deseado para cada aplicación (Deployment, Service, ExternalSecret, etc.). Es actualizado por los pipelines de CI.
- **Repositorio DevOps (`PTI-WellTrackDevops`)**: Define *qué* aplicaciones de Argo CD deben existir y las apunta a los charts dentro de *este* repositorio. Gestiona el despliegue de componentes de infraestructura (Harbor, Vault, External Secrets Operator, Argo CD mismo, etc.) y las definiciones de las aplicaciones.
- **ArgoCD**: Se ejecuta en el clúster, monitoriza los charts en *este* repositorio (según las instrucciones de `PTI-WellTrackDevops`), y sincroniza el estado del clúster para que coincida con el estado deseado definido aquí.
- **Infraestructura de Soporte**: Incluye un registro de contenedores (ej., Harbor/GHCR), Vault para secretos, y el ExternalSecrets Operator.

*(Considera actualizar el diagrama de arquitectura si existe)*

## Estructura del Repositorio

```
PTI-WellTrackGitOps/
├── charts/
│   ├── welltrack-frontend/
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       └── ingress.yaml
│   ├── welltrack-backend/
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       └── ingress.yaml
│   │       └── externalsecret.yaml # Ejemplo, podría no estar si se usan secrets nativos temporalmente
│   └── welltrack-ml/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           └── ... (deployment.yaml, service.yaml, ingress.yaml, etc.)
└── readme.md
# (El directorio argo-apps/ ya no se usa ya que las Aplicaciones se definen en PTI-WellTrackDevops)
```

## Proceso de Configuración y Despliegue

El despliegue se gestiona centralmente a través del repositorio [PTI-WellTrackDevops](https://github.com/hongda-zhu/PTI-WellTrackDevops).

1.  **Bootstrap de Argo CD e Infraestructura:** Sigue las instrucciones de configuración en el repositorio `PTI-WellTrackDevops`. Esto típicamente implica aplicar una aplicación raíz de Argo CD (`bootstrap.yaml`) que luego despliega Argo CD mismo y todos los demás componentes de infraestructura (Harbor, Vault, External Secrets Operator, etc.) junto con las definiciones de `Application` de Argo CD para los servicios de frontend, backend y ML.
2.  **Argo CD Toma el Control:** Una vez hecho el bootstrap, Argo CD (configurado por `PTI-WellTrackDevops`) automáticamente:
    *   Monitorizará los charts de Helm dentro del directorio `charts/` de *este* repositorio.
    *   Sincronizará los recursos definidos (Deployments, Services, etc.) con el clúster de Kubernetes.

**No aplicas manifiestos directamente desde este repositorio.**

## Flujo de Trabajo CI/CD

1. Los desarrolladores envían código a los repositorios de aplicación (ej., `PTI-WellTrackBack`).
2. Las GitHub Actions en esos repositorios se disparan:
   - Construyen imágenes Docker.
   - Publican imágenes en el registro de contenedores (ej., GHCR/Harbor).
   - **Hacen checkout de *este* repositorio GitOps (`PTI-WellTrackGitOps`).**
   - **Actualizan el `image.tag` en el `values.yaml` del chart correspondiente (ej., `charts/welltrack-backend/values.yaml`).**
   - **Confirman y envían el cambio de vuelta a *este* repositorio.**
3. Argo CD detecta el commit enviado a *este* repositorio.
4. Argo CD sincroniza automáticamente el estado del clúster aplicando el chart de Helm actualizado, desplegando la nueva versión de la aplicación.

## Configuración

La configuración para cada aplicación se gestiona dentro del archivo `values.yaml` de su chart de Helm ubicado en el directorio `charts/`. HTTPS para Ingress se gestiona mediante `cert-manager` usando anotaciones y una sección `tls` en la configuración de Ingress dentro del `values.yaml` de cada chart.

### Configuración del Frontend (`charts/welltrack-frontend/values.yaml`):
```yaml
replicaCount: 1
image:
  repository: ghcr.io/hongda-zhu/welltrack-frontend
  tag: latest # Este valor es actualizado por CI
  pullPolicy: Always
service:
  port: 3000
ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: selfsigned-cluster-issuer # O tu emisor real
  hosts:
    - host: app.welltrack.local
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts:
        - app.welltrack.local
      secretName: app-welltrack-local-tls
# ...
```

### Configuración del Backend (`charts/welltrack-backend/values.yaml`):
```yaml
replicaCount: 1
image:
  repository: ghcr.io/hongda-zhu/welltrack-backend
  tag: latest # Este valor es actualizado por CI
  pullPolicy: Always
service:
  port: 3001
ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: selfsigned-cluster-issuer # O tu emisor real
  hosts:
    - host: api.welltrack.local
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts:
        - api.welltrack.local
      secretName: api-welltrack-local-tls
# ... (sección de databaseCredentials omitida por brevedad, pero presente)
```

### Configuración de ML (`charts/welltrack-ml/values.yaml`):
```yaml
replicaCount: 1
image:
  repository: ghcr.io/hongda-zhu/welltrack-ml # O tu repositorio de imagen ML específico
  tag: latest # Este valor es actualizado por CI (si aplica)
  pullPolicy: Always # o IfNotPresent según tu estrategia
service:
  port: 5001 # Puerto de ejemplo para el servicio ML
ingress:
  enabled: true # Establecer en false si no se expone vía Ingress
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: selfsigned-cluster-issuer # O tu emisor real
  hosts:
    - host: ml.welltrack.local
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts:
        - ml.welltrack.local
      secretName: ml-welltrack-local-tls
# ...
```

*(Actualiza las URLs de los repositorios, nombres de host, secretNames y otros valores por defecto según sea necesario)*

## Información Sensible

La información sensible (como contraseñas de bases de datos, claves API para servicios externos) se gestiona usando HashiCorp Vault y el Kubernetes External Secrets Operator.
- Los recursos `ExternalSecret` definidos dentro de las plantillas Helm (ej., `charts/welltrack-backend/templates/externalsecret.yaml`) especifican qué secretos obtener de Vault.
- El External Secrets Operator lee estos recursos y crea objetos `Secret` estándar de Kubernetes en el namespace apropiado, que luego son montados por los pods de la aplicación.
- Los certificados TLS para Ingress, si son gestionados por `cert-manager` como está configurado, resultarán en que `cert-manager` cree y gestione los objetos `Secret` de Kubernetes que contienen el certificado y la clave privada.
- **Nunca confirmes manifiestos `Secret` de Kubernetes que contengan datos sensibles o claves privadas directamente a este repositorio.**

## Contribuciones

1.  **Cambios Primarios vía CI:** Las actualizaciones a las etiquetas de imagen dentro de los archivos `values.yaml` **solo** deben ocurrir a través de los pipelines CI/CD automatizados que se ejecutan en los repositorios de aplicación.
2.  **Cambios Manuales:** Los commits directos a este repositorio generalmente deben limitarse a:
    - Modificar plantillas de Helm (`templates/`).
    - Ajustar la configuración por defecto no sensible en `values.yaml`.
    - Actualizar archivos `Chart.yaml`.
3.  **Estructura:** Mantén la estructura de chart establecida dentro del directorio `charts/`.

## Repositorios Relacionados

- **Orquestación:** [PTI-WellTrackDevops](https://github.com/hongda-zhu/PTI-WellTrackDevops)
- **Aplicaciones:**
    - [WellTrack Frontend](https://github.com/hongda-zhu/PTI-WellTrackFront)
    - [WellTrack Backend](https://github.com/hongda-zhu/PTI-WellTrackBack)
    - [WellTrack ML](https://github.com/hongda-zhu/PTI-WellTrackML) # TODO: Reemplazar con la URL real del Repo de la App ML
