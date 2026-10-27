# randomquotes (Tutorial 07)

Despliegue del microservicio Express `randomquotes` con Docker en Amazon Web Services:
**Amazon ECR** (registro de imágenes), **Amazon ECS + AWS Fargate** (4 réplicas),
**Application Load Balancer** (balanceo de carga) y **GitHub Actions** (CI/CD con OIDC).

La app devuelve en `/` una frase aleatoria junto con el hostname del contenedor que
atendió la solicitud, lo que permite ver el balanceo entre réplicas.

## Arquitectura

```
GitHub → GitHub Actions → Amazon ECR → Amazon ECS (Fargate: 4 tareas) ← Application Load Balancer ← Navegador
```

| GCP / tutorial original            | AWS / esta versión          |
|------------------------------------|-----------------------------|
| Compute Engine                     | Amazon EC2 (alternativa)    |
| Instance templates + instance group| Launch Templates + Auto Scaling |
| Docker Swarm / Docker Service      | Amazon ECS / ECS Service    |
| Docker Hub                         | Amazon ECR                  |
| Réplicas                           | ECS Tasks                   |
| Reglas de firewall                 | Security Groups             |
| Balanceo Swarm                     | Application Load Balancer   |
| Cloud SQL                          | Amazon RDS (opcional)       |

## Archivos

- `index.js`: app Express (puerto 80) que responde `frase - Container Id: <hostname>`.
- `Dockerfile`: imagen basada en `node:22-alpine`, instala dependencias con `npm ci --omit=dev`.
- `../.github/workflows/deploy.yml`: workflow que, en cada push a `main` que modifique
  `Tutorial-07/`, construye la imagen, la publica en ECR y fuerza un nuevo despliegue del
  servicio ECS. (GitHub solo ejecuta workflows ubicados en la raíz del repositorio, por eso
  está allí y usa `working-directory: Tutorial-07`.)

## Correr localmente

```
docker build -t randomquotes .
docker run --rm -p 8080:80 randomquotes
```

Abrir http://localhost:8080

## Recursos AWS utilizados

| Recurso               | Nombre / valor                                   |
|-----------------------|--------------------------------------------------|
| Región                | `us-east-1`                                      |
| Repositorio ECR       | `randomquotes`                                   |
| Clúster ECS           | `randomquotes-cluster` (Fargate)                 |
| Task Definition       | `randomquotes-task` — 0.25 vCPU / 0.5 GB, puerto 80/TCP, logs `/ecs/randomquotes` |
| Servicio ECS          | `randomquotes-service` — Desired tasks = 4       |
| Load Balancer         | `randomquotes-alb` (HTTP :80)                    |
| Target Group          | `randomquotes-targets` (tipo IP, health check `/`) |
| Security Groups       | `randomquotes-alb-sg` (80 desde 0.0.0.0/0), `randomquotes-task-sg` (80 solo desde el ALB) |
| Rol IAM (OIDC)        | `GitHubActionsRandomquotesRole`                  |
| Secret de GitHub      | `AWS_ROLE_ARN`                                   |

### Subir la imagen manualmente

```
AWS_REGION=us-east-1
AWS_ACCOUNT_ID=123456789012
REGISTRY="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
IMAGE="$REGISTRY/randomquotes"
aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$REGISTRY"
docker build -t randomquotes .
docker tag randomquotes:latest "$IMAGE:latest"
docker push "$IMAGE:latest"
```

## Evidencia de las pruebas

> Agregar las capturas en `Tutorial-07/evidencias/` y reemplazar los marcadores.

### 1. Imagen publicada en ECR
<!-- ![ECR](evidencias/ecr.png) -->

### 2. Servicio ECS con cuatro tareas RUNNING
<!-- ![ECS tasks](evidencias/ecs-tasks.png) -->

### 3. Balanceo comprobado desde el ALB

```
for i in $(seq 1 12); do
  curl -s http://DNS-DEL-ALB
  echo
done
```

Identificadores de contenedor observados:

| # | Container Id |
|---|--------------|
| 1 |              |
| 2 |              |
| 3 |              |

<!-- ![ALB](evidencias/alb.png) -->

**Papel del Application Load Balancer:** es el único punto de entrada público. Recibe las
solicitudes HTTP en el puerto 80 y las reparte entre las tareas registradas (por IP) en el
Target Group, enviando tráfico solo a los destinos que pasan el health check. Por eso, al
recargar, la respuesta puede venir de contenedores distintos.

### 4. Detención y recuperación automática
<!-- ![Recuperación](evidencias/recuperacion.png) -->

- **¿Por qué se crea una nueva tarea?** Porque el servicio ECS tiene `Desired count = 4`; al
  detener una tarea el número en ejecución baja a 3 y ECS lanza una de reemplazo para volver
  al estado deseado.
- **¿Qué componente mantiene la cantidad deseada?** El ECS Service (su scheduler), que compara
  continuamente las tareas RUNNING con el desired count.
- **¿Qué efecto tiene el health check del balanceador?** El ALB deja de enviar tráfico a la
  tarea detenida/no saludable y solo incluye la nueva tarea cuando pasa el health check en `/`;
  además, ECS puede reemplazar tareas que el ALB marca como unhealthy.

### 5. Despliegue automático con GitHub Actions
<!-- ![Actions](evidencias/actions.png) -->

## Conclusiones

- ECS/Fargate reemplaza a Docker Swarm sin necesidad de administrar máquinas virtuales.
- El Application Load Balancer distribuye la carga entre las réplicas y aísla los contenedores
  del acceso público directo.
- El ECS Service mantiene automáticamente el número de réplicas deseado (autorrecuperación).
- Con GitHub Actions + OIDC cada push a `main` publica una nueva imagen y redespliega el
  servicio sin guardar claves permanentes de AWS en GitHub.
