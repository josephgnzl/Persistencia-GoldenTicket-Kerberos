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
Pedir TGS (getST.py -spn cifs/<FQDN_DC>)
        ↓
TGS válido
        ↓
Pass-The-Ticket (pseexec / wmiexec)
        ↓
Acceso al dominio
        ↓
Persistencia (válido hasta que se rote la credencial de krbtgt)

```
