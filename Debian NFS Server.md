---
type: card
tags:
  - vm
  - network
  - virtualization
created:
  "{ date }":
---
> [!ABSTRACT] Conceptually Short Summary
> Realizar la configuración de un servidor NFS usando el Sistema Operativo Debian permitiendo al cliente subir archivos y descargarlos desde el servidor

## 🛠️ Requisitos & Precondiciones
- Sistema Operativo Debian Instint
- Máquina cliente (en este caso es una Kali Linux)

## Configuración del Servidor NFS
Para este escenario se procederá a la instalación del paquete del servidor y se implementará escenarios clave dentro del archivo de configuración como `nohide`, `no_root_squash`, `all_squash`. Además, sincronización de UIDs:
### Paso 1: Instalación de paquetes del servidor
1. Primero se procede a la instalación del servidor NFS, para ello se actualiza el gestor de paquetes (`apt`) y luego se importa el paquete necesario
``` bash
sudo apt update && sudo apt install -y nfs-kernel-server
```
--- 
Con esto el servidor ya está preparado y se puede proceder con la creación de los directorios que servirán como shares para el cliente.
2. Ahora se crea la estructura de pruebas en el servidor:
``` bash
# Carpeta base para pruebas generales 
sudo mkdir -p /srv/nfs/general 
sudo mkdir -p /srv/nfs/general/secret_nest

# Carpeta para probar escalada de privilegios (no_root_squash) 
sudo mkdir -p /srv/nfs/dev_share 

# Carpeta para probar aislamiento forzado (all_squash) 
sudo mkdir -p /srv/nfs/public_upload 

# Generar archivos de prueba 
echo "General root access" | sudo tee /srv/nfs/general/readme.txt 
echo "Nested confidential data" | sudo tee /srv/nfs/general/secret_nest/secret.txt
```
En esta sección se han creado varias carpetas, la razón de esto, es:
- **Pruebas generales**: se busca comprender los conceptos básicos, además que hay un parámetro llamado `nohide` que permite ver las carpetas hijas del repositorio compartido, por eso se crea "secret_nest".
- **Escalada de privilegios:** Obtener privilegios de root suplantando su UID/GID desde el cliente
- Luego, se crean los archivos para probar los parámetros establecidos.
---
3. Ahora viene el montaje de un sistema de archivos temporal (tmpfs) para probar el parámetro`nohide`:
``` bash
sudo mount -t tmpfs tmpfs /srv/nfs/general/secret_nest 
echo "This is on a filesystem mounted underneath." | sudo tee /srv/nfs/general/anidado_secreto/disco_separado.txt
```
`nohide` se aplica a las particiones de disco, no a los directorios en sí, entonces por ejemplo si yo tengo dos directorios en `/dev/sda1`, esto cuenta como un volumen y NFS nunca lo oculta incluso con `nohide` desactivado. He ahí la razón de esta partición. 
![[image-8.png|Image in spanish]]
Aquí sucede algo interesante pues antes de montar este sistema de archivos temporal, la carpeta ya tenía un archivo (`secret.txt`), pero como sucedió la participación, ese archivo queda inaccesible temporalmente.
Por eso se crea otro archivo sobre este tmpfs y poder hacer las pruebas 

---
4. Ahora viene lo bueno, pues aquí se configura los shares con sus respectivos parámetros, además de ponerlos en público, esta configuración se realiza en `/etc/exports` 
``` text
# Case 1: Default behavior (implicit hide)
# When mounting /srv/nfs/general, 'secret_nest' will appear empty.
/srv/nfs/general               *(ro,sync,no_subtree_check)

# Case 2: Expose the child filesystem with `nohide`
/srv/nfs/general/secret_nest *(ro,sync,nohide,no_subtree_check)

# Case 3: Classic PrivEsc vector (no_root_squash)
# Allows the client's root user to write as root on the server
/srv/nfs/dev_share              *(rw,sync,no_subtree_check,no_root_squash)

# Case 4: Identity forcing (all_squash)
# Squash any user to nobody/nogroup (anonuid/anongid)
/srv/nfs/public_upload          *(rw,sync,no_subtree_check,all_squash,anonuid=1001,anongid=1001)
```
Varios parámetros ya son un poco explícitos como `ro` (read only) o `rw` (read write), pero también están `sync` y `no_subtree_check` 

<details> 
<summary>📖 Haz clic aquí para ver la explicación de cada parámetro</summary> 

En el caso de `sync` obliga al servidor a guardar los datos en disco duro y luego responder al cliente confirman la operación, lo cual es persistente y seguro pero más lento. Luego `async` guarda los datos en RAM, avisa al usuario que la operación fue hecha, y luego los guarda en disco, es más inseguro porque puede suceder algún fallo al servidor y se borrarían los datos.
 - Con `subtree_check` en cambio es como un verificador de ruta raíz, es decir, mediante el inodo verifica que el archivo siga dentro de la ruta que debería estar y no fue cambiado. El problema sucede cuando otra persona mueve el archivo que estabas usando lo cual al hacer la comprobación hay error. `no_subtree_check` deshabilita esa comprobación evitando ese error, pero claro, si un atacante sabe el inodo de otro archivo, este puede traerlo aunque no esté compartido.
Luego se encuentran los `squash`, cuyo objetivo es desactivar los privilegios del cliente en el servidor:
- `root_squash`: Aquí si el usuario root quiere escribir o leer algún recurso del servidor, este será aplastado a un usuario sin privilegios (nobody/nogroup)
- `no_root_squash`: Con este parámetro el root del cliente es también root en el servidor (UID 0), lo cual permitiría a un atacante elevar privilegios
- `all_squash`: La más general de todas, pues aquí todo usuario es aplastado a un usuario nobody 
</details>





## Key Takeaways 
- Mientras realizaba la instalación de paquetes usé el operador AND (&&), es increíble como a este punto no conocía el orden del funcionamiento, pero creía que se evaluaban las condiciones de forma simultánea, en realidad, es de izquierda a derecha, y si falla el anterior, el siguiente ya no se ejecuta. Por eso me daba error, había ejecutado mal el paquete y sin embargo si actualizó el gestor de paquetes. En cambio, en OR (||) en el primero puede haber error, pero igual ejecuta el siguiente comando.

