# Ansible Empresa — Guía de referencia

Este repositorio contiene toda la automatización de infraestructura de la empresa usando Ansible, AWX y ArgoCD sobre Kubernetes (K3s).

---

## Índice

1. [Conceptos clave](#conceptos-clave)
2. [Estructura del proyecto](#estructura-del-proyecto)
3. [Flujo de despliegue](#flujo-de-despliegue)
4. [GitOps con ArgoCD — la fuente de verdad es Git](#gitops-con-argocd--la-fuente-de-verdad-es-git)
5. [ArgoCD vs Terraform — no son lo mismo](#argocd-vs-terraform--no-son-lo-mismo)
6. [Cómo empezar](#cómo-empezar)

---

## Conceptos clave

### Playbook

Un playbook es el archivo principal donde describís **qué querés hacer** y **en qué servidores**. Está escrito en YAML y es el punto de entrada de cualquier automatización.

Un playbook puede hacer una sola cosa simple (reiniciar un servicio) o orquestar tareas complejas en cientos de servidores en un orden determinado.

```yaml
# playbooks/restart_service.yml
---
- name: Reiniciar servicio en servidores
  hosts: app_servers          # A quién le aplica (viene del inventario)
  become: yes                 # Ejecutar como sudo

  tasks:
    - name: Reiniciar el servicio
      systemd:
        name: "{{ service_name }}"
        state: restarted

    - name: Verificar que quedó corriendo
      systemd:
        name: "{{ service_name }}"
      register: resultado

    - name: Fallar si no levantó
      fail:
        msg: "El servicio no levantó correctamente"
      when: resultado.status.ActiveState != "active"
```

**Principio clave — idempotencia:** Un playbook bien escrito puede ejecutarse 10 veces seguidas y siempre producir el mismo resultado. Si el servicio ya está corriendo, Ansible lo detecta y no hace nada. Esto lo hace seguro para ejecutar en producción.

---

### Inventory (Inventario)

El inventario define **quiénes son tus servidores** y cómo se agrupan. Ansible necesita saber a qué máquinas conectarse antes de poder hacer cualquier cosa.

Los servidores se organizan en **grupos** para poder aplicarles configuraciones distintas.

```yaml
# inventories/prod/hosts.yml
all:
  children:
    app_servers:
      hosts:
        app-01.empresa.com:
          ansible_host: 192.168.0.10
        app-02.empresa.com:
          ansible_host: 192.168.0.11

    db_servers:
      hosts:
        db-01.empresa.com:
          ansible_host: 192.168.0.20

    web_servers:
      hosts:
        web-01.empresa.com:
          ansible_host: 192.168.0.30
```

Los inventarios se separan por ambiente. El playbook es siempre el mismo — lo que cambia es contra qué inventario se ejecuta:

```bash
# Ejecutar contra desarrollo
ansible-playbook playbooks/deploy_app.yml -i inventories/dev/hosts.yml

# Ejecutar contra producción
ansible-playbook playbooks/deploy_app.yml -i inventories/prod/hosts.yml
```

---

### Roles

Un rol es una forma de **organizar y reutilizar** conjuntos de tareas. En lugar de repetir las mismas tareas en varios playbooks, las encapsulás en un rol y lo llamás desde donde lo necesites.

Podés pensar en un rol como una función en programación: lo escribís una vez y lo usás en múltiples lugares.

```
roles/
└── nginx/
    ├── tasks/
    │   └── main.yml      # Las tareas que ejecuta el rol
    ├── handlers/
    │   └── main.yml      # Acciones que se disparan ante eventos (ej: reiniciar nginx)
    ├── templates/
    │   └── nginx.conf.j2 # Plantillas de configuración con variables
    ├── files/
    │   └── index.html    # Archivos estáticos que el rol copia
    └── defaults/
        └── main.yml      # Valores por defecto de variables del rol
```

Un rol se usa desde un playbook así:

```yaml
# playbooks/site.yml
---
- name: Configurar servidores web
  hosts: web_servers
  roles:
    - common   # Se aplica a todos primero
    - nginx    # Luego instala y configura nginx
```

**Ventaja real:** Si mañana necesitás instalar nginx en 20 servidores nuevos, solo agregás los hosts al grupo `web_servers` en el inventario. El rol ya está escrito.

---

### Group Vars (Variables de grupo)

Las variables de grupo permiten definir **valores distintos para distintos ambientes o grupos de servidores**, sin cambiar el playbook ni el rol.

```
group_vars/
└── all/
    ├── vars.yml    # Variables en texto plano
    └── vault.yml   # Variables sensibles encriptadas con Ansible Vault
```

```yaml
# inventories/prod/group_vars/app_servers.yml
java_version: "17"
app_port: 8080
max_memory: "4g"
log_level: "WARN"
```

```yaml
# inventories/dev/group_vars/app_servers.yml
java_version: "17"
app_port: 8080
max_memory: "1g"       # Menos memoria en dev
log_level: "DEBUG"     # Más logging en dev
```

El playbook usa `{{ java_version }}` y Ansible automáticamente toma el valor correcto según el inventario que estés usando.

---

### Collections (Colecciones)

Las colecciones son **paquetes de roles, módulos y plugins** creados por la comunidad o por fabricantes (Red Hat, AWS, VMware, etc.) que extenden las capacidades de Ansible.

En lugar de escribir todo desde cero, usás colecciones ya probadas:

```yaml
# collections/requirements.yml
collections:
  - name: community.general        # Módulos de uso general
  - name: ansible.posix            # Módulos para sistemas POSIX/Linux
  - name: community.postgresql     # Módulos para gestionar PostgreSQL
  - name: amazon.aws               # Módulos para gestionar recursos de AWS
```

Se instalan con un solo comando:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

**Analogía:** Si Ansible es Python, las colecciones son las librerías que instalás con pip.

---

### AWX (Ansible Tower gratuito)

AWX es la interfaz web y API de Ansible. Permite ejecutar playbooks sin acceso SSH, gestionar credenciales de forma segura, programar ejecuciones automáticas y ver el historial de todo lo que se ejecutó.

Lo que AWX agrega sobre Ansible puro:
- Interfaz web para lanzar playbooks
- Control de acceso por roles (quién puede ejecutar qué)
- Gestión de credenciales encriptadas
- Scheduling (ejecuciones programadas)
- Notificaciones (Slack, email, etc.)
- API REST para integraciones

---

### ArgoCD

ArgoCD implementa **GitOps** para Kubernetes. Monitorea el repositorio Git y cuando detecta un cambio en los manifiestos YAML de K8s, los aplica automáticamente al cluster.

Lo que esto significa en la práctica: el estado deseado de tu cluster está definido en Git. Si algo se modifica manualmente en el cluster, ArgoCD lo detecta y lo corrige automáticamente para que coincida con lo que está en Git.

---

## Estructura del proyecto

```
ansible-empresa/
│
├── inventories/
│   ├── dev/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       ├── all.yml
│   │       └── app_servers.yml
│   └── prod/
│       ├── hosts.yml
│       └── group_vars/
│           ├── all.yml
│           └── app_servers.yml
│
├── playbooks/
│   ├── site.yml                # Playbook maestro
│   ├── deploy_app.yml
│   ├── patch_servers.yml
│   ├── restart_service.yml
│   ├── add_user.yml
│   ├── backup_db.yml
│   └── health_check.yml
│
├── roles/
│   ├── common/                 # Se aplica a todos los servidores
│   ├── nginx/
│   ├── java/
│   └── postgresql/
│
├── group_vars/
│   └── all/
│       ├── vars.yml            # Variables globales
│       └── vault.yml           # Secretos encriptados
│
├── k8s/
│   ├── awx/
│   │   ├── kustomization.yaml
│   │   └── awx-instance.yml
│   └── argocd/
│       └── apps/
│           └── awx-app.yml
│
├── collections/
│   └── requirements.yml
│
├── docs/
│   └── runbooks/
│       ├── deploy.md
│       └── rollback.md
│
├── ansible.cfg
├── .gitignore
└── README.md
```

---

## Flujo de despliegue

El siguiente diagrama muestra cómo fluye cualquier automatización desde que el ingeniero escribe un playbook hasta que se aplica en los servidores:

```
┌─────────────────────────────────────────────────┐
│                    Tu PC                        │
│       VS Code · Git · Playbooks · Roles         │
└─────────────────────┬───────────────────────────┘
                      │ git push
                      ▼
┌─────────────────────────────────────────────────┐
│                   GitLab                        │
│    Repositorio central · Ramas dev/main/prod    │
└──────────────┬──────────────────┬───────────────┘
               │                  │
               ▼                  ▼
┌──────────────────────┐  ┌──────────────────────┐
│       ArgoCD         │  │        AWX            │
│  Detecta cambios     │  │  Lee playbooks        │
│  Aplica YAMLs en K3s │  │  Gestiona ejecuciones │
└──────────┬───────────┘  └──────────┬────────────┘
           │                         │
           ▼                         ▼
┌──────────────────────┐  ┌──────────────────────┐
│   K3s (Kubernetes)   │  │   Ansible Engine      │
│  Pods: AWX · ArgoCD  │  │  Ejecuta via SSH      │
└──────────────────────┘  └──────────┬────────────┘
                                     │ SSH
                                     ▼
┌─────────────────────────────────────────────────┐
│          Servidores gestionados                 │
│        Linux · Windows · VMs · Cloud            │
└─────────────────────────────────────────────────┘
```

### El flujo paso a paso

1. El ingeniero escribe o modifica un playbook en VS Code
2. Hace `git commit` y `git push` al repositorio en GitLab
3. ArgoCD detecta el cambio en los manifiestos K8s y los aplica al cluster automáticamente
4. AWX detecta el cambio en el repositorio y el playbook actualizado queda disponible
5. AWX ejecuta el playbook usando Ansible Engine
6. Ansible Engine se conecta por SSH a los servidores sin necesidad de agentes
7. Los servidores quedan configurados según lo definido en el playbook

---

## GitOps con ArgoCD — la fuente de verdad es Git

ArgoCD monitorea el repositorio Git constantemente. Cuando detecta un cambio en los manifiestos YAML de Kubernetes, los aplica automáticamente al cluster sin que nadie tenga que entrar a ninguna consola.

Esto aplica a cualquier cluster Kubernetes: K3s propio, EKS en AWS, GKE en Google Cloud o AKS en Azure. El comportamiento es exactamente el mismo.

### El flujo sin tocar ninguna consola

```
Vos modificás un manifiesto YAML en VS Code
                │
                │  git push
                ▼
             GitLab
                │
                │  ArgoCD monitorea el repo cada X segundos
                ▼
      ArgoCD detecta el cambio
                │
                │  Compara Git vs estado actual del cluster
                │  Encuentra diferencia
                ▼
      EKS / K3s / GKE / AKS
                │
                │  Aplica el cambio automáticamente
                ▼
         Cluster actualizado ✅
         Sin tocar la consola de AWS ni kubectl
```

### Auto-corrección de desvíos

Si alguien entra a la consola de AWS y modifica algo manualmente en el cluster, ArgoCD lo detecta y lo revierte automáticamente para que coincida con lo que está en Git.

```
Git dice:          replicas = 4
Alguien cambia:    replicas = 2  (directo en AWS Console)
ArgoCD detecta:    diferencia entre Git y cluster
ArgoCD corrige:    replicas = 4  (vuelve a lo que dice Git)
```

Git se convierte en la única fuente de verdad. Nadie del equipo necesita acceso directo al cluster para hacer cambios — todo pasa por Git, con historial y revisión de código.

### Rollback instantáneo

Si un cambio rompe algo en producción, el rollback es revertir el commit en Git:

```bash
git revert <commit>
git push
# ArgoCD detecta el cambio y vuelve al estado anterior ✅
```

No hace falta recordar qué comandos se ejecutaron ni qué había antes. Git tiene todo el historial.

---

## ArgoCD vs Terraform — no son lo mismo

Son similares en filosofía (el estado deseado vive en Git) pero gestionan cosas distintas y se usan juntos, no en lugar uno del otro.

| | ArgoCD | Terraform |
|---|---|---|
| **Qué gestiona** | Recursos dentro de Kubernetes | Infraestructura de nube (VMs, redes, DBs, clusters) |
| **Formato** | YAML (manifiestos K8s) | HCL (lenguaje propio de Terraform) |
| **Estado en Git** | ✅ | ✅ |
| **Detecta desvíos** | ✅ y los corrige solo | ✅ pero requiere intervención manual |
| **Aplica cambios solo** | ✅ Automático | ❌ Requiere `terraform apply` |
| **Específico de K8s** | ✅ Solo K8s | ❌ Cualquier proveedor cloud |

### Terraform crea todo, incluido el cluster EKS y sus nodos

Terraform no solo crea lo "externo" — crea el cluster EKS completo, los nodos workers y todo lo necesario para que K8s funcione en AWS:

```
Terraform crea:
├── VPC, subnets, security groups     ← red donde vive el cluster
├── Cluster EKS                       ← el cluster Kubernetes en sí
├── Node groups (las VMs/nodos EC2)   ← las máquinas donde corren los pods
├── Roles IAM                         ← permisos para que K8s opere en AWS
├── Load Balancer Controller          ← le da a K8s el poder de crear LBs
├── RDS, S3, ElastiCache              ← otros servicios AWS
└── Kubeconfig                        ← acceso al cluster para kubectl y ArgoCD
```

Es importante distinguir entre **nodo** y **pod**:

```
Nodo  →  la VM (EC2) donde corre Kubernetes    →  Terraform lo crea
Pod   →  el contenedor corriendo en el nodo    →  ArgoCD lo gestiona
```

### Load Balancer en EKS — quién crea qué y quién asigna las IPs

El Load Balancer es resultado de una colaboración entre Terraform, K8s y AWS:

```
Terraform
└── Crea el Load Balancer Controller y los roles IAM
    (le da a K8s el permiso y la capacidad de crear LBs en AWS)

ArgoCD aplica este manifiesto:
    apiVersion: v1
    kind: Service
    spec:
      type: LoadBalancer   ← K8s le pide a AWS que cree un ALB

AWS
└── Crea el Load Balancer real y asigna la IP pública automáticamente
```

Las IPs las asigna siempre AWS, no Terraform ni ArgoCD:

```
IP pública del Load Balancer  →  AWS la asigna automáticamente
IPs de los pods               →  K8s las asigna del rango de la VPC
IPs de los nodos (EC2)        →  AWS las asigna
```

### Cómo se usan juntos en capas

```
Terraform
└── Crea la infraestructura base en AWS
    ├── VPC y subnets
    ├── Cluster EKS + nodos workers
    ├── Load Balancer Controller
    ├── Roles IAM
    ├── Base de datos RDS
    └── Buckets S3

ArgoCD (dentro del cluster EKS)
└── Gestiona lo que corre dentro del cluster
    ├── Pods y Deployments
    ├── Services (que pueden crear LBs en AWS)
    ├── AWX
    └── Configuraciones K8s

Ansible (desde AWX)
└── Configura los servidores que no son K8s
    ├── Servidores Linux on-premise
    ├── Servidores Windows
    └── Cualquier máquina con SSH o WinRM
```

### ¿Ansible puede gestionar contenedores?

Sí, Ansible tiene módulos para Docker y Kubernetes. Cuándo conviene usarlo depende del contexto:

```
Contexto 1 — Servidor con Docker simple (sin K8s)
└── Ansible tiene sentido
    Ya estás usándolo para configurar el servidor
    No justifica agregar otra herramienta solo para los contenedores

Contexto 2 — Tenés Kubernetes
└── Usás ArgoCD directamente
    K8s tiene sus propias herramientas nativas
    Ansible sería un paso atrás
```

### ¿Ansible puede reemplazar a Cobian para backups a S3?

Sí, completamente. Ansible puede copiar directorios de servidores Windows directamente a un bucket S3, con más control y centralización que cualquier herramienta de backup tradicional.

```yaml
# playbooks/backup_a_s3.yml
---
- name: Copiar archivos de Windows a S3
  hosts: windows_servers
  vars:
    directorio_origen: 'C:\Datos\Reportes'
    bucket_s3: "mi-empresa-backups"
    prefijo_s3: "backups/reportes"

  tasks:
    - name: Sincronizar directorio con S3
      community.aws.s3_sync:
        bucket: "{{ bucket_s3 }}"
        file_root: "{{ directorio_origen }}"
        key_prefix: "{{ prefijo_s3 }}/{{ inventory_hostname }}"
        delete: no

    - name: Registrar en log
      ansible.windows.win_lineinfile:
        path: 'C:\Logs\backup.log'
        line: "Backup ejecutado: {{ ansible_date_time.iso8601 }}"
        create: yes
```

Lo que ganás sobre Cobian:

| | Cobian | Ansible |
|---|---|---|
| Múltiples servidores | ❌ Uno por vez | ✅ Todos a la vez |
| Programación | ✅ | ✅ Desde AWX |
| Logs centralizados | ❌ | ✅ En AWX |
| Condiciones complejas | ❌ | ✅ |
| Notificaciones | Básico | ✅ Slack, email, etc. |
| Auditoría | Limitada | ✅ Completa en AWX |
| Costo | Licencia | Gratis |

Para conectarse a servidores Windows Ansible usa **WinRM** en lugar de SSH. Para S3 usa la colección `amazon.aws` que tiene módulos específicos para todos los servicios de AWS.

### Control de costos — por qué importa quién crea el Load Balancer

Cuando K8s/ArgoCD crea un recurso de AWS como un Load Balancer, Terraform no sabe que existe. No está en su state y no puede gestionarlo.

```
Terraform state (lo que Terraform conoce):
├── VPC                ✅ lo ve y lo gestiona
├── Cluster EKS        ✅ lo ve y lo gestiona
├── Nodos EC2          ✅ lo ve y lo gestiona
└── RDS                ✅ lo ve y lo gestiona

Load Balancer creado por K8s:
└── ❌ Terraform no lo ve
    No está en su state
    No lo puede eliminar con terraform destroy
    No aparece en reportes de costo de Terraform
```

El LB igual aparece en la consola de AWS y en el billing — el problema es que no podés gestionarlo desde Terraform ni auditarlo de forma centralizada.

### El problema real: recursos huérfanos

```
Desarrollador despliega una app en K8s
K8s crea un Load Balancer  →  u$d 20/mes

Desarrollador borra la app del repositorio
ArgoCD borra el Deployment y el Service
Pero el LB queda huérfano en AWS  →  seguís pagando u$d 20/mes sin saberlo
```

Esto se llama resource leak y es uno de los problemas más comunes y costosos en AWS. Empresas que acumulan decenas de Load Balancers, IPs elásticas y volúmenes EBS que nadie sabe de dónde salieron y siguen pagando por ellos mes a mes.

Con Terraform el recurso está en el state, `terraform destroy` lo elimina junto con todo, y sabés exactamente qué existe y cuánto cuesta.

### Herramientas para detectar recursos huérfanos en AWS

Si ya tienen recursos creados por K8s que no están en Terraform, estas herramientas ayudan a encontrarlos:

| Herramienta | Para qué |
|---|---|
| **AWS Cost Explorer** | Ver costos por servicio y detectar picos inesperados |
| **AWS Tag Editor** | Encontrar recursos sin tags de identificación |
| **cloud-nuke** | Eliminar recursos huérfanos (usar con cuidado en prod) |
| **Infracost** | Estimar costos antes de aplicar cambios con Terraform |

### Para qué sirve cada herramienta del ecosistema

| Herramienta | Propósito |
|---|---|
| **Terraform** | Crear y gestionar infraestructura cloud (EKS, VMs, RDS, S3, redes, LBs) |
| **ArgoCD** | GitOps para Kubernetes — mantener el cluster sincronizado con Git |
| **Flux** | Alternativa a ArgoCD para GitOps en Kubernetes |
| **Ansible** | Configurar servidores Linux y Windows, backups, tareas operativas |
| **AWX** | Interfaz web y API para gestionar y ejecutar Ansible |

---

## Cómo empezar

### Prerequisitos

- Git instalado en tu PC
- VS Code con las extensiones: Ansible (Red Hat), YAML (Red Hat), GitLens
- Acceso SSH al servidor de control (donde corre K3s)
- Cuenta en GitLab con acceso al repositorio

### Clonar el repositorio

```bash
git clone git@gitlab.com:empresa/ansible-empresa.git
cd ansible-empresa
```

### Instalar dependencias de Ansible

```bash
pip install ansible ansible-lint
ansible-galaxy collection install -r collections/requirements.yml
```

### Configurar credenciales

Las credenciales sensibles se gestionan con Ansible Vault. Nunca editar `vault.yml` sin encriptar:

```bash
# Crear o editar el archivo de secretos
ansible-vault edit group_vars/all/vault.yml
```

### Ejecutar un playbook

```bash
# Verificar qué haría el playbook sin ejecutarlo (dry run)
ansible-playbook playbooks/health_check.yml -i inventories/dev/hosts.yml --check

# Ejecutar contra desarrollo
ansible-playbook playbooks/health_check.yml -i inventories/dev/hosts.yml

# Ejecutar contra producción (requiere confirmación del equipo)
ansible-playbook playbooks/health_check.yml -i inventories/prod/hosts.yml
```

---

## Reglas del equipo

- Los playbooks **nunca** se ejecutan directamente en producción sin pasar por dev primero
- Las credenciales **nunca** se suben en texto plano a Git — siempre usar Ansible Vault
- Todo cambio en producción debe tener un Pull Request aprobado por al menos otro miembro del equipo
- Los playbooks nuevos se prueban primero con `--check` (modo dry run)
- Staging se agrega como ambiente intermedio cuando el equipo lo requiera
