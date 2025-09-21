# Ejemplo de Repositorio GitOps con FluxCD

Este repositorio es una demostración de cómo gestionar un clúster de Kubernetes de forma declarativa utilizando [FluxCD](https://fluxcd.io/), siguiendo los principios de GitOps. El repositorio Git actúa como la única fuente de verdad para el estado del clúster.

## ¿Qué es GitOps?

GitOps es una forma de implementar la Entrega Continua para aplicaciones nativas de la nube. La idea principal es tener un repositorio Git que contenga descripciones declarativas de la infraestructura deseada (en este caso, manifiestos de Kubernetes) y un proceso automatizado que asegure que el entorno de producción coincida con el estado descrito en el repositorio.

## Estructura del Repositorio

La estructura está organizada para separar la configuración del clúster de las definiciones de las aplicaciones, lo cual es una práctica recomendada para escalar.

```
.
├── apps
│   └── webapp
│       ├── deployment.yaml
│       ├── kustomization.yaml
│       ├── namespace.yaml
│       └── service.yaml
└── clusters
    └── demo
        ├── apps.yaml
        ├── flux-system
        │   ├── gotk-components.yaml
        │   ├── gotk-sync.yaml
        │   └── kustomization.yaml
        └── sources.yaml
```

### `clusters/`

Este directorio contiene la configuración específica para cada clúster gestionado por Flux.

*   **`clusters/demo/`**: Contiene la configuración para un clúster llamado `demo`.
    *   **`flux-system/`**: Este es el directorio de "bootstrap" de Flux.
        *   `gotk-components.yaml`: Define todos los componentes de Flux (controladores, CRDs, etc.) que se instalan en el clúster.
        *   `gotk-sync.yaml`: El punto de partida. Le dice a Flux que se sincronice a sí mismo usando este repositorio y que aplique todo lo que se encuentre en la ruta `./clusters/demo`.
    *   **`sources.yaml`**: Define las fuentes de manifiestos. En este caso, define un `GitRepository` llamado `demo-repo` que apunta a este mismo repositorio. Se usa para desplegar las aplicaciones.
    *   **`apps.yaml`**: Define una `Kustomization` que le dice a Flux cómo desplegar las aplicaciones. Apunta a la fuente `demo-repo` y al path `./apps/webapp` para desplegar la aplicación `webapp`.

### `apps/`

Este directorio contiene los manifiestos de Kubernetes para las aplicaciones que se desplegarán en el clúster.

*   **`apps/webapp/`**: Contiene todos los recursos de Kubernetes para una aplicación web de ejemplo (un servidor Nginx).
    *   `namespace.yaml`: Crea el namespace `web`.
    *   `deployment.yaml`: Despliega 3 réplicas de Nginx.
    *   `service.yaml`: Expone el despliegue dentro del clúster.
    *   `kustomization.yaml`: Agrupa todos los manifiestos anteriores para que Kustomize los pueda procesar juntos.

## Flujo de Trabajo

1.  **Bootstrap**: Flux se instala en el clúster y se configura para observar la ruta `./clusters/demo` en la rama `main` de este repositorio.
2.  **Sincronización del Clúster**: La `Kustomization` `flux-system` aplica los manifiestos de `clusters/demo`.
3.  **Creación de Fuentes y Apps**: Al aplicarse `clusters/demo`, se crea el `GitRepository` `demo-repo` y la `Kustomization` `webapp`.
4.  **Sincronización de la App**: El `kustomize-controller` detecta la `Kustomization` `webapp`. Lee los manifiestos del path `./apps/webapp` (desde la fuente `demo-repo`) y los aplica en el clúster dentro del namespace `web`.
5.  **Reconciliación Continua**: Flux monitorea continuamente el repositorio Git. Cualquier cambio (un `git push`) en los manifiestos desencadenará automáticamente una actualización en el clúster para que coincida con el estado definido en Git.

## Cómo Empezar

Para replicar esta configuración en tu propio clúster.

### Requisitos Previos

*   Un clúster de Kubernetes.
*   `kubectl` configurado para acceder a tu clúster.
*   La CLI de `flux` instalada. Instrucciones de instalación.
*   Un token de GitHub con permisos de `repo`.

### Bootstrap

1.  Haz un fork de este repositorio en tu propia cuenta de GitHub.
2.  Exporta tu usuario y el token de GitHub como variables de entorno:
    ```sh
    export GITHUB_USER="<tu-usuario-github>"
    export GITHUB_TOKEN="<tu-token-github>"
    ```
3.  Ejecuta el comando de bootstrap de Flux para conectar tu clúster a tu fork del repositorio:
    ```sh
    flux bootstrap github \
      --owner=$GITHUB_USER \
      --repository=flux-example \
      --branch=main \
      --path=./clusters/demo \
      --personal
    ```

### Verificar la Sincronización

Después de unos minutos, puedes verificar que Flux ha sincronizado todo correctamente.

```sh
# Comprueba que las Kustomizations están listas
flux get kustomizations --watch

# Comprueba que las fuentes Git están listas
flux get sources git

# Comprueba que la aplicación webapp está desplegada
kubectl get pods -n web
```
Deberías ver 3 pods de `webapp` corriendo.

## Realizar un Cambio

Para ver la magia de GitOps en acción:

1.  Clona tu fork del repositorio localmente.
2.  Edita el archivo `apps/webapp/deployment.yaml` y cambia el campo `replicas` de `3` a `5`.
3.  Haz commit y push del cambio a la rama `main`.
    ```sh
    git add .
    git commit -m "Scale webapp to 5 replicas"
    git push origin main
    ```
4.  Observa cómo Flux actualiza el despliegue. Puedes forzar una reconciliación para acelerar el proceso:
    ```sh
    flux reconcile kustomization webapp --with-source
    ```
5.  Verifica que ahora hay 5 pods:
    ```sh
    kubectl get pods -n web
    ```
