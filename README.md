# Laboratorio 1 – Issabel PBX / Asterisk

Simulación de una central telefónica empresarial (Claro) usando **Issabel PBX** sobre **Asterisk 18**, virtualizada en **Hyper-V**. El laboratorio cubre extensiones, movilidad de llamadas (Follow Me), operadora automática (IVR), videollamada, buzón de voz personalizado, correo electrónico con dominio propio y monitoría del sistema.

## Tabla de contenido

- [Arquitectura](#arquitectura)
- [Requisitos cumplidos](#requisitos-cumplidos)
- [Extensiones](#extensiones)
- [1. Preparación de la VM en Hyper-V](#1-preparación-de-la-vm-en-hyper-v)
- [2. Extensión móvil (Follow Me)](#2-extensión-móvil-follow-me)
- [3. Operadora automática (IVR)](#3-operadora-automática-ivr)
- [4. Videollamada](#4-videollamada)
- [5. Buzón de voz personalizado](#5-buzón-de-voz-personalizado)
- [6. Correo electrónico con dominio propio](#6-correo-electrónico-con-dominio-propio)
- [7. Monitoría del PBX](#7-monitoría-del-pbx)
- [Problemas resueltos durante el laboratorio](#problemas-resueltos-durante-el-laboratorio)
- [Créditos](#créditos)

---

## Arquitectura

```
┌─────────────────────┐        ┌───────────────────────────────┐
│   PC (Windows)       │        │     VM Issabel PBX (Hyper-V)   │
│   Hyper-V Manager     │  NAT/  │  Rocky Linux 8 + Asterisk 18   │
│   Conmutador Externo ├───────►│  IP: 10.86.201.119              │
│   (tethering USB)     │  red   │  Apache (PHP 7.4) + Roundcube   │
└─────────┬─────────────┘        │  (PHP 8.1, en paralelo)         │
          │                      └───────────────┬─────────────────┘
          │ WiFi (hotspot)                        │
┌─────────▼─────────┐                    ┌────────▼─────────┐
│  Celular A          │                    │  Celular B         │
│  Softphone (Linphone)│                   │  Softphone (Linphone)│
│  Ext. móvil (203)     │                   │  Ext. Cliente (204)  │
└─────────────────────┘                    └───────────────────┘
```

- **Hipervisor:** Hyper-V (Windows)
- **Sistema operativo del PBX:** Rocky Linux 8.8 (incluido en el instalador de Issabel)
- **PBX:** Issabel PBX 2.12 / Asterisk 18.19.0
- **Red:** Conmutador Externo de Hyper-V enlazado a **tethering USB** de un celular (el adaptador WiFi nativo no soportó el modo puente necesario)
- **Softphones usados:** Zoiper 5 (audio) y Linphone (audio + video, gratuito)
- **Cliente de correo:** Roundcube (instalado manualmente, corriendo sobre PHP 8.1 en paralelo al PHP 7.4 que usa el panel de Issabel)

---

## Requisitos cumplidos

| # | Requisito del enunciado | Estado |
|---|---|---|
| 1 | Llamadas entre extensiones | ✅ |
| 2 | Extensión móvil (recibe la llamada al pasar la extensión → celular) | ✅ Follow Me |
| 3 | Operadora propia programada (no la default) | ✅ IVR con 5 opciones |
| 4 | Permitir videollamada | ✅ |
| 5 | Configuración externa (trunk) | ⏭️ Omitido (indicación del docente) |
| 6 | Buzón de voz | ✅ Con saludo personalizado en español |
| 7 | Correo electrónico con dominio propio | ✅ Envío y recepción, con adjunto de audio |
| 8 | Monitoría del PBX | ✅ CDR Report |
| 9 | Servicios transparentes para el usuario | ✅ (el flujo IVR → Follow Me → buzón es invisible para quien llama) |

---

## Extensiones

| Extensión | Nombre | Rol | Follow Me |
|---|---|---|---|
| 201 | Soporte Técnico | Resuelve fallas técnicas; el encargado a veces sale a campo | Sí → 203 |
| 202 | Atención al Cliente | Consultas administrativas, facturación | No |
| 203 | Soporte Técnico Móvil | Celular del encargado de Soporte | — |
| 204 | Cliente | Simula una llamada externa entrante | No |
| 205 | Ventas | Contratación de nuevos servicios | No |

> Todas las extensiones son de tipo **PJSIP**, transporte **UDP** (el transporte TCP detectado automáticamente por algunos softphones no fue compatible con el `qualify`/registro de Asterisk en este entorno).

<details>
<summary>📷 Capturas: creación de las extensiones 201, 202 y 203</summary>

| | |
|---|---|
| ![Menú PBX Configuration → Extensions](capturas/02-extensiones/27-menu-pbx-configuration-extensions.png) | ![Add an Extension](capturas/02-extensiones/28-add-extension-generic-pjsip.png) |
| ![Extensión 201 - Jefe Taller / Soporte](capturas/02-extensiones/29-extension-201-jefe-taller.png) | ![Voicemail de la 201](capturas/02-extensiones/30-extension-201-voicemail.png) |
| ![Secret de la 201](capturas/02-extensiones/31-extension-201-secret.png) | ![Extensión 202 - Recepción](capturas/02-extensiones/32-extension-202-recepcion.png) |
| ![Secret de la 202](capturas/02-extensiones/33-extension-202-secret.png) | ![Extensión 203 - Móvil](capturas/02-extensiones/34-extension-203-jefe-taller-movil.png) |
| ![Secret de la 203](capturas/02-extensiones/35-extension-203-secret.png) | ![Listado final 201/202/203](capturas/02-extensiones/36-lista-extensiones-201-202-203.png) |

</details>

---

## 1. Preparación de la VM en Hyper-V

1. VM en **Generación 1**, 2+ vCPU, 4 GB RAM, 40 GB disco.
2. ISO oficial de Issabel: https://www.issabel.org/get-issabel/
3. **Adaptador de red:** se probó primero un **Conmutador Externo enlazado a la tarjeta WiFi** del host → la interfaz de la VM quedaba en estado `NO-CARRIER` permanentemente (los adaptadores WiFi de laptop rara vez soportan el modo puente que exige un switch externo de Hyper-V).
4. **Solución:** se cambió el switch externo para enlazarlo a la interfaz de **tethering USB** de un celular (en vez de WiFi) → la interfaz de la VM pasó a `state UP`.
5. Dentro de la VM, la interfaz no traía una conexión gestionada por NetworkManager por defecto, así que se creó manualmente:

```bash
nmcli connection add type ethernet ifname eth0 con-name eth0 ipv4.method auto
nmcli connection up eth0
```

6. Verificación:

```bash
ip a
# eth0 queda con inet 10.86.201.119/24 (rango del hotspot)
```

<details>
<summary>📷 Capturas: creación de la VM e instalación de Issabel</summary>

| | |
|---|---|
| ![Descarga de Issabel](capturas/01-instalacion-vm/01-descarga-issabel-web.png) | ![Nombre de la VM](capturas/01-instalacion-vm/02-hyperv-nombre-vm.png) |
| ![Generación 1](capturas/01-instalacion-vm/03-hyperv-generacion.png) | ![Memoria asignada](capturas/01-instalacion-vm/04-hyperv-memoria.png) |
| ![Configuración de red](capturas/01-instalacion-vm/05-hyperv-red.png) | ![Disco duro virtual](capturas/01-instalacion-vm/06-hyperv-disco.png) |
| ![Opciones de instalación (ISO)](capturas/01-instalacion-vm/07-hyperv-opciones-instalacion.png) | ![Menú Configuración de la VM](capturas/01-instalacion-vm/08-hyperv-menu-configuracion.png) |
| ![Adaptador de red / Conmutador Externo](capturas/01-instalacion-vm/09-hyperv-adaptador-red.png) | ![Iniciar la VM](capturas/01-instalacion-vm/10-hyperv-iniciar-vm.png) |
| ![Arranque del instalador](capturas/01-instalacion-vm/11-consola-boot-instalador.png) | ![Selección de idioma](capturas/01-instalacion-vm/12-instalador-idioma.png) |
| ![Resumen de instalación](capturas/01-instalacion-vm/13-instalador-resumen.png) | ![Distribución de teclado](capturas/01-instalacion-vm/14-instalador-teclado.png) |
| ![Contraseña de root](capturas/01-instalacion-vm/15-instalador-password-root.png) | ![Comenzar instalación](capturas/01-instalacion-vm/16-instalador-comenzar-instalacion.png) |
| ![Progreso de instalación](capturas/01-instalacion-vm/17-instalador-progreso.png) | ![Contraseña de MariaDB](capturas/01-instalacion-vm/18-consola-password-mariadb.png) |
| ![Contraseña admin de IssabelPBX](capturas/01-instalacion-vm/19-consola-password-admin-issabel.png) | ![Idioma del PBX](capturas/01-instalacion-vm/20-consola-idioma-pbx.png) |
| ![Driver SIP por defecto (chan_pjsip)](capturas/01-instalacion-vm/21-consola-driver-sip-pjsip.png) | ![Login de Rocky Linux](capturas/01-instalacion-vm/22-consola-login-rocky.png) |
| ![`ip a` sin IP asignada](capturas/01-instalacion-vm/23-consola-ip-a-sin-ip.png) | ![`nmcli` — IP asignada correctamente](capturas/01-instalacion-vm/24-consola-nmcli-ip-asignada.png) |

</details>

---

## 2. Extensión móvil (Follow Me)

Objetivo del enunciado: *"el jefe debe estar en la oficina, pero debe responder donde esté o dejar un mensaje"*.

Configuración en **PBX → PBX Configuration → Follow Me → 201**:

| Campo | Valor |
|---|---|
| Enable | Yes |
| Initial Ring Time | 15 |
| Ring Strategy | ringallv2 |
| Ring Time | 20 |
| Follow-Me List | `203#` |
| Destination if no answer | Voicemail → 201 |

**Flujo resultante:**

```
Llaman a 201 (oficina) → timbra 15s sin respuesta
  → timbra también en 203 (celular) → timbra 20s sin respuesta
    → cae al buzón de voz de 201
```

Confirmado en el **CDR Report** (`Reports → CDR Report`), donde el campo *Dst. Channel* muestra el intento de Follow Me: `Local/FMGL-203#@from-...`.

<details>
<summary>📷 Capturas: configuración de Follow Me</summary>

| | |
|---|---|
| ![Menú Applications → Follow Me](capturas/03-follow-me-y-video/37-menu-applications-followme.png) | ![Follow-Me List — selector de extensiones](capturas/03-follow-me-y-video/38-followme-201-lista-dropdown.png) |
| ![Follow Me 201 — General Settings](capturas/03-follow-me-y-video/39-followme-201-general-settings.png) | ![Destination if no answer → Voicemail 201](capturas/03-follow-me-y-video/40-followme-destination-if-no-answer.png) |

</details>

---

## 3. Operadora automática (IVR)

### Guion de bienvenida (generado con ElevenLabs, WAV 8000 Hz mono)

```
Bienvenido a Claro. Gracias por comunicarte con nosotros.

Para continuar, por favor selecciona una de las siguientes opciones:

Para soporte técnico, marque 1.
Para atención al cliente, marque 2.
Para ventas, marque 3.
Para dejar un mensaje, marque 4.
Para repetir estas opciones, marque 5.
```

### Configuración del IVR (PBX → PBX Configuration → IVR → "Operadora Claro")

| Ext | Destino |
|---|---|
| 1 | Extensión 201 (Soporte Técnico) |
| 2 | Extensión 202 (Atención al Cliente) |
| 3 | Extensión 205 (Ventas) |
| 4 | Voicemail → 201 |
| 5 | El mismo IVR (repite el menú) |

- **Direct Dial:** Extensions (permite marcar una extensión conocida en cualquier momento)
- **Timeout Destination / Invalid Destination:** Voicemail → 201 (evita que la llamada termine en error)

### Acceso al IVR desde extensiones internas

Como no se configuró un trunk externo, el IVR se expone internamente mediante un **Misc Application**:

| Feature Code | Destino |
|---|---|
| 700 | IVR → Operadora Claro |

<details>
<summary>📷 Capturas: configuración del IVR</summary>

| | |
|---|---|
| ![Edit IVR — General Options / Announcement / Direct Dial](capturas/04-ivr/01-ivr-general-options.png) | ![IVR Entries completas (1-5)](capturas/04-ivr/02-ivr-entries-completas.png) |

</details>

---

## 4. Videollamada

- Se probó primero con **Zoiper 5 Free**, que **no incluye video** en su versión gratuita.
- Se migró a **Linphone** (gratuito, con video incluido).
- Se agregaron los codecs de video en cada extensión (**pjsip → allow**):

```
ulaw,alaw,gsm,opus,g729,h264,vp8
```

> Importante: el campo `allow` **reemplaza** la lista de codecs, no la complementa — hay que incluir también los de audio o se pierde la llamada de voz.

Videollamada confirmada exitosamente entre las extensiones 201 y 202.

<details>
<summary>📷 Capturas: Linphone y codecs de video</summary>

| | |
|---|---|
| ![Descarga de Linphone](capturas/03-follow-me-y-video/41-linphone-org-descarga.png) | ![Pantalla de conexión de Linphone](capturas/03-follow-me-y-video/42-linphone-connection-screen.png) |
| ![Cuenta SIP de terceros en Linphone](capturas/03-follow-me-y-video/43-linphone-third-party-sip-account.png) | ![Codecs (allow) en la extensión 203](capturas/03-follow-me-y-video/44-extension-203-pjsip-avanzado-codecs.png) |

</details>

<details>
<summary>📷 Evidencia de funcionamiento: extensiones registradas y llamadas reales</summary>

| | |
|---|---|
| ![Extensión 201 registrada en el PC (Zoiper, check verde)](capturas/07-evidencia-funcionamiento/01-extension-201-registrada-pc.png) | ![Extensión 203 registrada en el celular ("Cuenta activada")](capturas/07-evidencia-funcionamiento/02-extension-203-registrada-celular.png) |
| ![Echo Test (*43) — llamada activa confirmando audio](capturas/07-evidencia-funcionamiento/03-echo-test-llamada-activa.png) | ![Registro de llamadas reales entre 201, 202 y 203 (contestadas)](capturas/07-evidencia-funcionamiento/04-registro-llamadas-201-202-203.png) |
| ![Sesión de videollamada activa en Linphone entre 201 y 202](capturas/07-evidencia-funcionamiento/05-videollamada-linphone-sesion-activa.png) | ![Videollamada real — cámaras del celular y del PC transmitiendo en vivo](capturas/08-llamadas-en-vivo/01-videollamada-camaras-reales.png) |
| ![Llamada entrante real: Cliente (204) marcando al IVR (700), recibida en Linphone del PC](capturas/08-llamadas-en-vivo/02-llamada-entrante-ivr-700-cliente.png) | |

</details>

---

## 5. Buzón de voz personalizado

Issabel no expone en el panel un campo directo para subir un saludo personalizado de voicemail ("Unavailable/Busy Recording"), así que se aplicó copiando el archivo manualmente a la carpeta de cada buzón.

### Guion del saludo (genérico, ElevenLabs)

```
En este momento no podemos atender su llamada.

Por favor, deje su mensaje después del tono, indicando su nombre y motivo
de la llamada, y nos pondremos en contacto con usted a la brevedad posible.

Gracias por su paciencia.
```

### Pasos

1. Subir el audio en **Applications → System Recordings** (nombre: `buzon`).
2. Ubicar el archivo real en el servidor:

```bash
find / -iname "buzon.wav" 2>/dev/null
# /var/lib/asterisk/sounds/custom/buzon.wav
```

3. Copiarlo como saludo de "no disponible" y de "ocupado" en cada extensión (Asterisk usa archivos distintos según el motivo del rechazo):

```bash
for ext in 201 202 203 205; do
  mkdir -p /var/spool/asterisk/voicemail/default/$ext/INBOX
  chown -R asterisk:asterisk /var/spool/asterisk/voicemail/default/$ext
  cp /var/lib/asterisk/sounds/custom/buzon.wav /var/spool/asterisk/voicemail/default/$ext/unavail.wav
  cp /var/lib/asterisk/sounds/custom/buzon.wav /var/spool/asterisk/voicemail/default/$ext/busy.wav
  chown asterisk:asterisk /var/spool/asterisk/voicemail/default/$ext/unavail.wav
  chown asterisk:asterisk /var/spool/asterisk/voicemail/default/$ext/busy.wav
done
```

4. **Idioma:** para evitar que Asterisk reproduzca el mensaje genérico en inglés (`vm-intro.gsm`) después del saludo propio, se configuró `Language Code = es` en **todas** las extensiones que participan en la llamada — incluida la que **marca** (no solo la que recibe), ya que el idioma efectivo lo determina el canal que origina la llamada.

Verificación:

```bash
asterisk -rx "pjsip show endpoint 201" | grep -i language
# language: es
```

---

## 6. Correo electrónico con dominio propio

### 6.1 Dominio y cuentas (panel de Issabel)

- **Email → Domains:** dominio `claro.com`
- **Email → Accounts:** `soporte@claro.com`, `atencion@claro.com`, `ventas@claro.com`

<details>
<summary>📷 Capturas: dominio y cuentas de correo</summary>

| | |
|---|---|
| ![Crear cuenta soporte@claro.com](capturas/05-correo/01-email-accounts-crear.png) | ![Listado de cuentas (soporte/atencion/ventas)](capturas/05-correo/02-email-accounts-lista.png) |
| ![Dominio claro.com registrado (3 cuentas)](capturas/05-correo/03-email-domains.png) | |

</details>

### 6.2 Voicemail-to-email

En cada extensión, sección **Voicemail**:

- **Email Address:** cuenta correspondiente (ej. `soporte@claro.com`)
- **Email Attachment:** `yes`

Con esto, cada mensaje de voz llega por correo con el archivo `.wav` adjunto. Verificado leyendo directamente el buzón de Cyrus IMAP:

```bash
find / -iname "*claro.com*" 2>/dev/null | grep -v httpd
# /var/spool/imap/domain/c/claro.com
cat "/var/spool/imap/domain/c/claro.com/a/user/atencion/1."
```

### 6.3 Cliente web de correo (Roundcube)

Issabel no trae un webmail instalado. Se instaló **Roundcube** manualmente, en paralelo al PHP que usa el panel de Issabel (7.4), usando **PHP 8.1** (repositorio Remi) sin afectar el panel.

**Puntos clave de la instalación:**

1. Instalar PHP 8.1 como paquete aparte (no reemplaza la 7.4 del sistema):

```bash
dnf install -y https://rpms.remirepo.net/enterprise/remi-release-8.rpm
dnf install -y php81 php81-php-fpm php81-php-imap php81-php-mbstring \
                php81-php-xml php81-php-mysqlnd php81-php-gd php81-php-intl
```

2. **Deshabilitar el conf global** que trae el paquete `php81-php.conf` — aplica PHP 8.1 a *todo* el sitio y rompe el panel de Issabel:

```bash
mv /etc/httpd/conf.d/php81-php.conf /etc/httpd/conf.d/php81-php.conf.disabled
```

3. Crear un conf propio que aplique PHP 8.1 **solo** a la carpeta de Roundcube:

```apache
# /etc/httpd/conf.d/roundcube-php81.conf
<Directory "/var/www/html/roundcube">
    <FilesMatch \.php$>
        SetHandler "proxy:unix:/var/opt/remi/php81/run/php-fpm/www.sock|fcgi://localhost"
    </FilesMatch>
    DirectoryIndex index.php
    AllowOverride All
    Require all granted
</Directory>
```

4. Ajustar la ACL del socket de PHP-FPM 8.1: Apache en esta instalación de Issabel corre como usuario **`asterisk`**, no `apache`:

```ini
; /etc/opt/remi/php81/php-fpm.d/www.conf
listen.acl_users = asterisk
listen.owner = asterisk
listen.group = asterisk
```

5. Base de datos y esquema de Roundcube:

```bash
mysql -u root -p<PASSWORD_ROOT_MYSQL> -e "
CREATE DATABASE roundcubedb CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'roundcube'@'localhost' IDENTIFIED BY '<PASSWORD_ROUNDCUBE>';
GRANT ALL PRIVILEGES ON roundcubedb.* TO 'roundcube'@'localhost';
FLUSH PRIVILEGES;"

mysql -u root -p<PASSWORD_ROOT_MYSQL> roundcubedb < /var/www/html/roundcube/SQL/mysql.initial.sql
```

6. Envío de correo (SMTP): Postfix en esta instalación **no tiene SASL habilitado** (`smtpd_sasl_auth_enable = no`), así que Roundcube debe conectarse **sin autenticación**, usando el puerto 25 (no 587):

```php
// config/config.inc.php
$config['smtp_host'] = 'localhost:25';
$config['smtp_user'] = '';
$config['smtp_pass'] = '';
```

Resultado: acceso en `https://<IP-del-servidor>/roundcube`, envío y recepción de correo funcionando, con el adjunto de voicemail visible desde la interfaz web.

<details>
<summary>📷 Capturas: Roundcube funcionando (login, envío y recepción)</summary>

| | |
|---|---|
| ![Login de Roundcube](capturas/05-correo/04-roundcube-login.png) | ![Bandeja de entrada vacía, recién configurada](capturas/05-correo/05-roundcube-inbox.png) |
| ![Redactar correo: soporte@claro.com → ventas@claro.com](capturas/05-correo/06-roundcube-redactar.png) | ![Correo recibido y abierto: "prueba" / "Prueba de correo"](capturas/05-correo/07-roundcube-correo-recibido.png) |

</details>

---

## 7. Monitoría del PBX

- **Reports → CDR Report:** historial completo de llamadas (origen, destino, estado, duración), incluye el rastro del Follow Me.
- **Reports → Graphic Report / Summary:** estadísticas de uso.
- **PBX → Voicemails:** listado de mensajes de voz por extensión con reproducción directa.
- **Dashboard principal:** CPU, memoria, llamadas activas en tiempo real.

<details>
<summary>📷 Capturas: monitoría</summary>

| | |
|---|---|
| ![PBX → Voicemails — mensajes por extensión](capturas/06-monitoria/01-pbx-voicemails.png) | ![Reports → CDR Report — historial de llamadas](capturas/06-monitoria/02-cdr-report.png) |

</details>

> En el CDR Report, la fila `Dst. Channel: Local/FMGL-203#@from-...` es la evidencia registrada del intento de Follow Me hacia la extensión 203 antes de caer a buzón.

---

## Problemas resueltos durante el laboratorio

| Problema | Causa | Solución |
|---|---|---|
| Interfaz de red en `NO-CARRIER` | Switch externo de Hyper-V enlazado a WiFi (no soporta modo puente) | Cambiar el switch a tethering USB o a una tarjeta Ethernet |
| Sin IP aunque la interfaz esté `UP` | No existía conexión gestionada por NetworkManager | `nmcli connection add ... ipv4.method auto` + `nmcli connection up` |
| Panel/SSH inaccesibles tras varias pruebas de llamada fallidas | Fail2ban baneó la IP del PC/celular (jail `asterisk`) | `fail2ban-client set asterisk unbanip <IP>` + `addignoreip` para pruebas |
| Nuevo registro SIP rechazado ("exceed max contacts of 1") | Contacto SIP viejo en estado `Unavail` ocupando el único slot | Aumentar `Max Contacts` a 2 en la extensión, o `module reload res_pjsip.so` |
| Registro SIP marcado `Unavailable` pese a credenciales correctas | Softphone registrado por TCP, extensión configurada solo para UDP | Forzar transporte **UDP** en el softphone |
| Error 503 al llamar entre extensiones | Softphone (celular anfitrión del hotspot) no podía alcanzar su propia red compartida | Usar un segundo dispositivo o tethering USB en vez de hotspot WiFi |
| Video no disponible | Zoiper 5 Free no incluye videollamada | Migrar a Linphone (gratuito, con video) |
| Video no conecta tras habilitar codecs | El campo `allow` sobrescribió también los codecs de audio | Incluir audio + video juntos: `ulaw,alaw,gsm,opus,g729,h264,vp8` |
| Audio MP3 del IVR no se reproduce (`decodeMP3` warnings) | Formato MP3 no compatible con el decodificador de Asterisk | Convertir a WAV, 8000 Hz, mono, PCM |
| IVR: "Please select a Destination" | El campo *Destination if no answer* de Follow Me es obligatorio | Crear un grupo de VMBlast o usar `Voicemail` directo (requiere buzón habilitado antes) |
| Voicemail en inglés a pesar del audio propio | Idioma `en` por defecto en la extensión que **origina** la llamada | `Language Code = es` en todas las extensiones (no solo en la que recibe) |
| Mensaje en inglés solo al usar la opción 4 del IVR | Esa ruta usa el saludo de **busy**, no *unavailable*; solo se había copiado uno | Copiar el audio también como `busy.wav` |
| Roundcube: error 500 en todo el panel de Issabel | El paquete `php81` instala un conf de Apache que aplica PHP 8.1 globalmente | Deshabilitar `php81-php.conf`, dejar solo el conf específico de la carpeta `/roundcube` |
| Roundcube: `Permission denied` al socket de PHP-FPM | Apache corre como usuario `asterisk`, no `apache`, en esta instalación | ACL del socket (`listen.acl_users`) apuntando a `asterisk` |
| Roundcube: `Access denied for user roundcube` | Contraseña del usuario MySQL no coincidía con la del config.inc.php | `ALTER USER 'roundcube'@'localhost' IDENTIFIED BY '...'` |
| Roundcube: `Table 'roundcubedb.session' doesn't exist` | Base de datos creada vacía, sin el esquema importado | Importar `SQL/mysql.initial.sql` |
| Roundcube: no envía correo, "conexión rehusada" | `config.inc.php` apuntaba a `smtp_host = localhost:587` (puerto sin servicio) | Cambiar a `localhost:25` |
| Roundcube: "Ha fallado la autenticación" al enviar | Postfix no tiene SASL habilitado, pero Roundcube exigía usuario/contraseña | Dejar `smtp_user`/`smtp_pass` vacíos |
| VM con IP estática inalcanzable desde el celular | El adaptador de Hyper-V estaba en una **Red Interna** (aislada del host), no en un Conmutador Externo real | Cambiar a Conmutador Externo enlazado a una tarjeta física (Ethernet o tethering) |

---

## Créditos

Laboratorio desarrollado como práctica de la asignatura de Redes/Telefonía IP, usando Issabel PBX (Asterisk) sobre Hyper-V.
