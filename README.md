# Custom Execution Environment UBI9 (Python 3.12 / AAP 2.6)

Repositorio para la construcción y distribución de un **Execution Environment (EE)** desacoplado de las imágenes oficiales de AAP, construido directamente sobre **Red Hat Universal Base Image 9 (`ubi9/python-312`)**.

Este EE está diseñado para escenarios donde se requiere control absoluto del stack base, compatibilidad nativa con **Python 3.12**, soporte para `ansible-core 2.16.x` y automatización multicloud (AWS, Azure, VMware, Windows, NetApp e Infoblox).

---

## 📋 Especificaciones Técnicas

| Componente | Versión / Detalle |
| :--- | :--- |
| **Plataforma Objetivo** | Red Hat Ansible Automation Platform (AAP) 2.6 / AWX / CLI |
| **Imagen Base** | `registry.access.redhat.io/ubi9/python-312:latest` |
| **Ansible Core** | `ansible-core 2.16.14` |
| **Python Runtime** | `Python 3.12.15` |
| **Herramienta de Build** | `ansible-builder` v3 (`version: 3`) |
| **Motor de Contenedores** | Podman |

---

## 📦 Contenido y Dependencias

### 1. Colecciones de Ansible (`requirements.yml`)
- **Infraestructura AAP**: `infra.aap_configuration`, `infra.ee_utilities`, `infra.aap_utilities`
- **Cloud & Virtualización**: `amazon.aws`, `azure.azcollection`, `community.vmware`
- **Sistemas Operativos**: `community.windows`, `community.general`, `community.crypto`
- **Networking & Almacenamiento**: `netapp.ontap`, `infoblox.nios_modules`

### 2. Dependencias Python (`requirements.txt`)
- **Windows / Autenticación**: `pywinrm[kerberos,credssp]`, `requests-credssp`, `requests-ntlm`, `requests-kerberos`
- **VMware**: `pyVmomi`, `pyVim`
- **AWS**: `boto3`, `botocore`
- **Azure**: `azure-mgmt-compute`, `azure-mgmt-network`, `azure-mgmt-resource`, `azure-mgmt-storage`, `azure-identity`
- **NetApp & Infoblox**: `netapp-lib`, `infoblox-client`
- **Core & Criptografía**: `cryptography`, `jmespath`

### 3. Paquetes del Sistema (`bindep.txt`)
- **Runtime**: `git`, `rsync`, `krb5-libs`, `krb5-workstation`, `libffi`, `openssl-libs`
- **Compilación (Build stage)**: `gcc`, `make`, `python3.12-devel`, `krb5-devel`, `libffi-devel`, `openssl-devel`, `openldap-devel`

---

## ⚙️ Arquitectura de Construcción y Resoluciones Técnicas

Para construir este EE sobre `ubi9/python-312` se resolvieron varios retos de bajo nivel en `execution-environment.yml`:

1. **Alineación de `sys.path` y Virtualenv**:
   La imagen `ubi9/python-312` inicializa un virtualenv en `/opt/app-root/`. Las librerías instaladas por `ansible-builder` caen en `/usr/local/lib/` y `/usr/local/lib64/`. Se inyecta un fichero de ruta `.pth` en el virtualenv y se exporta `PYTHONPATH` para que el runtime las descubra automáticamente.

2. **Saneamiento de `enum34`**:
   Se elimina el paquete legacy `enum34` arrastrado por dependencias transitivas para evitar el enmascaramiento del módulo nativo `@enum.global_enum` de Python 3.12.

3. **Optimización de Memoria (OOM Prevention)**:
   Se configura `ENV PIP_OPTS="--no-cache-dir"` para permitir la resolución y compilación de más de 180 dependencias sin saturar la memoria RAM.

---

## 🚀 Paso 1: Requisitos Previos

Instalación de herramientas en el host:

    subscription-manager repos --enable=ansible-automation-platform-2.6-for-rhel-9-x86_64-rpms
    dnf install podman ansible-builder ansible-navigator -y
    
---

## 🛠️ Paso 2: Construcción del Execution Environment

Ejecutar la compilación desde la raíz del repositorio:

    ansible-builder build -t localhost/custom-ee:2.0.0 --container-runtime podman -v 3

---

## 🧪 Paso 3: Validación y Smoke Testing

### 3.1. Prueba interactiva de carga de librerías

    podman run --rm -it localhost/custom-ee:2.0.0 python3 -c 'import winrm, boto3, pyVmomi, netapp_lib, azure.identity, cryptography; print("VALIDACION EXITOSA - TODAS LAS LIBRERIAS CARGAN DE FORMA NATIVA")'

### 3.2. Ejecución del Playbook de Auditoría

    ansible-navigator run test-ee.yml -m stdout --eei localhost/custom-ee:2.0.0

---

## 📤 Paso 4: Publicación en el Registro de Contenedores

Etiquetar y subir la imagen generada:

    podman login <hub-o-registro.tu-empresa.com>
    podman tag localhost/custom-ee:2.0.0 <hub-o-registro.tu-empresa.com>/ansible/custom-ee:2.0.0
    podman push <hub-o-registro.tu-empresa.com>/ansible/custom-ee:2.0.0

---

## ⚙️ Paso 5: Integración en Ansible Automation Platform (AAP)

1. **Registrar el EE en Automation Controller**:
   - Navegar a **Administration > Execution Environments > Add**.
   - **Name**: `Custom EE UBI9 Python 3.12`.
   - **Image**: `<hub-o-registro.tu-empresa.com>/ansible/custom-ee:2.0.0`.
   - **Pull**: `Always` o `Missing`.

2. **Asignación operativa**:
   - Asignar a nivel de **Organización**, **Proyecto** o directamente en el **Job Template**.

---

## 📄 Estructura del Repositorio

    ├── .gitignore               # Exclusiones de Git
    ├── README.md                # Documentación técnica
    ├── bindep.txt               # Paquetes RPM del sistema y desarrollo
    ├── execution-environment.yml # Definición del EE con ansible-builder v3
    ├── requirements.txt         # Dependencias Python
    ├── requirements.yml         # Colecciones de Ansible
    └── test-ee.yml              # Playbook de verificación funcional
