#Hack The Box - Cap Writeup

* OS: Linux
* Dificultad:Easy
* IP de la Máquina: 10.129.88.34 
* Plataforma: Hack The Box (HTB)

---

Resumen 
Cap es una máquina Linux de dificultad fácil que demuestra los riesgos asociados con vulnerabilidades de control de acceso en aplicaciones web (IDOR), la transmisión de credenciales en protocolos no cifrados (FTP) y la mala configuración de permisos en ejecutables del sistema (*Linux Capabilities*).

---

1. Reconocimiento (Reconnaissance)

Escaneo Inicial de Puertos (Nmap)
Se comenzó con un escaneo rápido sobre el rango completo de puertos TCP (1-65535) para identificar los servicios abiertos en el objetivo:

bash
"nmap -p- --open --min-rate 5000 -vvv -sS -n -Pn 10.129.88.34 -oA scans/all_ports"

![Escaneo de puertos](images/01-nmap.png)
