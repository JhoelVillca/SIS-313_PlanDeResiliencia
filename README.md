# Sistema de Resiliencia Operativa y DRP Automatizado
## Tecnologias Usadas:

| Tecnología | Rol | ¿Por qué esta y no otra? |
| :--- | :--- | :--- |
| **🐧 Linux (Ubuntu Server)** | Sistema Base | Estabilidad, soporte nativo de contenedores y gestión eficiente de recursos en entornos virtualizados. |
| **📦 Restic** | Motor de Backup | **Deduplicación y Cifrado**: A diferencia de `tar` o `rsync`, Restic rompe los archivos en bloques pequeños, cifra todo con AES-256 y solo guarda los cambios exactos. *Resultado:* Backups 90% más ligeros y rápidos. |
| **🗄️ MinIO** | Bóveda (Storage) | **Compatibilidad S3**: Simula la nube de AWS (S3) pero localmente. Nos permite desacoplar el almacenamiento de la aplicación. Si la app muere, los datos viven en un "búnker" aislado. |
| **🧠 Ansible** | Orquestador (DRP) | **Automatización Idempotente**: Un DRP en papel no sirve en pánico. Ansible ejecuta la recuperación automáticamente. Si se rompe la DB, un comando reconstruye el entorno, instala dependencias e inyecta los datos. |
| **📸 LVM (Logical Volume Manager)** | Integridad de Datos | **Snapshots en Caliente**: Copiar una base de datos encendida corrompe los datos. LVM congela el sistema de archivos en milisegundos, permitiendo backups consistentes sin apagar el servicio. |
| **⏱️ Systemd Timers** | Cronómetro | **Fiabilidad**: Reemplazamos `cron` porque Systemd maneja dependencias (espera a que haya red), reintentos automáticos y logs binarios detallados para auditoría. |
| **🌐 mDNS / Hostnames** | Red Dinámica | **Portabilidad**: Configuramos la red basada en nombres (`app-node`, `db-node`) en lugar de IPs estáticas fijas, permitiendo desplegar la infraestructura en cualquier red  sin reconfigurar todo el código. |

