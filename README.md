# ¿Qué es un Golden Ticket?
Un Golden Ticket es un Ticket Granting Ticket (TGT) de Kerberos falsificado, utilizado en entornos Windows Active Directory para obtener autenticación como cualquier usuario del dominio, incluso como una cuenta privilegiada.

La técnica se basa en comprometer la clave secreta de la cuenta krbtgt, utilizada por el controlador de dominio para firmar y validar los TGT de Kerberos. Con acceso a esta clave, un atacante puede generar tickets que el dominio considere legítimos.

A diferencia de comprometer una cuenta individual, la obtención de la clave de krbtgt representa un compromiso mucho más profundo del dominio, ya que permite la creación de identidades Kerberos arbitrarias y puede utilizarse como mecanismo de persistencia prolongada.
