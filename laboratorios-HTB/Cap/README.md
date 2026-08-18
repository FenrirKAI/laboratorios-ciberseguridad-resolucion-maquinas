# 🟢 Hack The Box — Cap

> Writeup de la máquina **Cap** de Hack The Box.

---

## 📌 Información de la Máquina

| Información | Detalle |
|---|---|
| 🖥️ **Sistema Operativo** | Linux |
| 🎯 **Dificultad** | 🟢 Easy |
| 🌐 **IP** | `10.129.88.34` |
| 🏴 **Plataforma** | Hack The Box |
| 🔎 **Servicios principales** | FTP · SSH · HTTP |

---

## 📖 Resumen

**Cap** es una máquina Linux de dificultad **Easy** que demuestra los riesgos asociados a diferentes vulnerabilidades y malas configuraciones de seguridad.

Durante la resolución se identificaron las siguientes vulnerabilidades:

- **IDOR (Insecure Direct Object Reference)**
- **Transmisión de credenciales mediante FTP en texto plano**
- **Reutilización de credenciales**
- **Linux Capabilities inseguras (`cap_setuid`)**

La cadena de ataque comienza con la explotación de un **IDOR** en una aplicación web, permitiendo descargar una captura de tráfico `.pcap`.

El análisis de dicha captura mediante Wireshark permitió obtener credenciales FTP. Estas credenciales fueron reutilizadas para acceder al sistema mediante SSH.

Finalmente, mediante una configuración insegura de **Linux Capabilities** sobre `python3.8`, fue posible escalar privilegios hasta obtener acceso como `root`.

---

# 1. Reconocimiento

## 1.1 Escaneo Inicial de Puertos

Se comenzó realizando un escaneo sobre el rango completo de puertos TCP para identificar los servicios expuestos en la máquina objetivo.

nmap -p- --open --min-rate 5000 -vvv -sS -n -Pn 10.129.88.34 -oA scans/all_ports

Resultado

Se identificaron 3 puertos TCP abiertos:

Puerto	Servicio
21/tcp	FTP
22/tcp	SSH
80/tcp	HTTP

![Escaneo de puertos](images/01-nmap.png)

1.2 Enumeración de Servicios y Versiones

Posteriormente, se realizó un escaneo dirigido sobre los puertos encontrados para identificar las versiones de los servicios y obtener información adicional mediante scripts por defecto.

nmap -sCV -p21,22,80 10.129.88.34 -oA scans/targeted_services

Servicios Identificados
Puerto	Servicio	Versión
21/tcp	FTP	vsFTPd 3.0.3
22/tcp	SSH	OpenSSH 8.2p1 Ubuntu
80/tcp	HTTP	Gunicorn

![Escaneo de Servicios y versiones](images/02-nmap-servicios.png)

2. Enumeración Web
2.1 Security Dashboard

Al acceder al servicio HTTP mediante el puerto 80, se identificó una aplicación web denominada Security Dashboard.

La aplicación permite realizar capturas de tráfico de red y posteriormente acceder a ellas mediante archivos .pcap.

Durante la enumeración se buscó identificar rutas y recursos accesibles dentro de /data/.

Para ello se utilizó la herramienta ffuf

"ffuf -u [http://10.129.88.34/data/FUZZ](http://10.129.88.34/data/FUZZ) -w /usr/share/seclists/Fuzzing/4-digits-0000-9999.txt -fs 208"

![Escaneo de rutas](images/03-ffuf.png)

Resultado

Durante el fuzzing se identificaron diferentes recursos asociados a las capturas de tráfico almacenadas por la aplicación.

2.2 Identificación de IDOR

Se identificó un comportamiento vulnerable en la ruta:

/data/0

La aplicación permitía acceder directamente a una captura de tráfico sin realizar una validación adecuada de autorización.

Esto corresponde a una vulnerabilidad:

IDOR — Insecure Direct Object Reference

La vulnerabilidad permitió acceder y descargar la captura:

0.pcap

Esta captura sería utilizada posteriormente para analizar el tráfico de red y buscar información sensible.


3. Análisis de Tráfico — Wireshark

Una vez descargado el archivo 0.pcap, se procedió a analizar la captura utilizando Wireshark.

Durante el análisis de los paquetes TCP correspondientes al protocolo FTP, se observó que las credenciales de autenticación eran transmitidas en texto plano.

Esto es posible debido a que FTP no cifra las credenciales durante el proceso de autenticación.

🔑 Credenciales obtenidas

👤 Usuari:nathan
🔑 Contraseña:Buck3tH4TF0RM3!

Estas credenciales permitirían posteriormente autenticarse contra otros servicios de la máquina.

![Analisis de trafico](images/04-wireshark.png)

4. 🔑 Acceso Inicial
4.1 Acceso mediante FTP

Se probaron las credenciales obtenidas contra el servicio FTP:

ftp nathan@10.129.88.34

La autenticación fue exitosa, confirmando que las credenciales obtenidas desde la captura eran válidas.

![Acceso a FTP](images/05-ftp.png)

4.2 Reutilización de Credenciales — SSH

Debido a que las credenciales obtenidas eran válidas para FTP, se comprobó si también podían utilizarse para acceder mediante SSH.

ssh nathan@10.129.88.34

La autenticación fue exitosa y se obtuvo una shell interactiva como el usuario:

nathan

La reutilización de credenciales permitió pasar desde el acceso al servicio FTP a una sesión remota mediante SSH.

![Acceso a SSH](images/06-ssh.png)

5. Escalada de Privilegios
5.1 Enumeración de Linux Capabilities

Una vez obtenida una shell como nathan, se procedió a realizar una enumeración local del sistema.

Una de las comprobaciones realizadas fue la búsqueda de Linux Capabilities asignadas a los binarios:

getcap -r / 2>/dev/null

Durante la enumeración se identificó una configuración insegura en:

/usr/bin/python3.8

El binario presentaba las siguientes capabilities:

cap_setuid,cap_net_bind_service+eip

![getcap](images/07-getcap.png)

6.2 Identificación de cap_setuid

La capability cap_setuid permite a un proceso modificar su identificador de usuario (UID).

En este caso, Python disponía de dicha capability, lo que permitía modificar el UID del proceso a:

UID 0

El UID 0 corresponde al usuario:

root

Por lo tanto, la configuración de esta capability sobre Python permitía realizar una escalada de privilegios.


7. Explotación y Obtención de Root

Para explotar la capability cap_setuid, se utilizó Python para cambiar el UID del proceso a 0 y posteriormente ejecutar una shell.

python3 -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")'

Una vez ejecutado el comando, se obtuvo una shell con privilegios elevados.

Confirmación de privilegios
whoami

Resultado:

root

La escalada de privilegios fue exitosa.


![escalada de privilegios](images/08-root.png)

8. Root 

Una vez obtenido acceso como root, se procedió a localizar las flags correspondientes.

Banderas Obtenidas:
User Flag (user.txt): 72924db756f11370b5c13a940e7e47b6
Root Flag (root.txt): 6bdfeefc9c2ed60b24bcef59824da63f

9. Matriz de Vulnerabilidades e Impacto

## Matriz de Vulnerabilidades e Impacto

| # | Vulnerabilidad | Clasificación | Impacto |
|---|---|---|---|
| **1** | **Insecure Direct Object Reference (IDOR)** | 🔴 **Alta** | Permite a un usuario no autenticado descargar capturas PCAP arbitrarias del servidor. |
| **2** | **Tráfico no cifrado (FTP en texto plano)** | 🟠 **Media** | Exposición de credenciales de usuario en las capturas de red. |
| **3** | **Reutilización de Credenciales** | 🟠 **Media** | La clave del servicio FTP fue reutilizada para la administración remota por SSH. |
| **4** | **Linux Capabilities inseguras (`cap_setuid`)** | 🔴 **Crítica** | Permite a cualquier usuario local escalar privilegios de forma instantánea a `root`. |


10. Herramientas Utilizadas
Herramienta	Uso
🔎 Nmap: Reconocimiento y enumeración de puertos y servicios
🎯 ffuf: Enumeración de rutas y recursos web
📡 Wireshark: Análisis de tráfico de red
📂 FTP: Acceso al servicio FTP
🔐 SSH: Acceso remoto al sistema
🐧 getcap	:Enumeración de Linux Capabilities
🐍 Python	: Explotación de cap_setuid
   GTFOBins: Escalada de privilegios


11. Conclusión

La máquina Cap demuestra cómo diferentes vulnerabilidades y malas configuraciones pueden encadenarse para comprometer completamente un sistema.

El ataque comenzó mediante una vulnerabilidad IDOR, que permitió acceder a una captura de tráfico que no debería haber estado disponible para un usuario no autenticado.

Posteriormente, debido al uso de FTP en texto plano, fue posible recuperar credenciales válidas a partir del análisis de la captura mediante Wireshark.

Estas credenciales fueron reutilizadas para obtener acceso mediante SSH como el usuario nathan.

Finalmente, durante la enumeración local se identificó que el binario python3.8 poseía la capability cap_setuid, lo que permitió modificar el UID del proceso a 0 y obtener una shell con privilegios de roo










