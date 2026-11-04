# NF1.UD4. AA1-NFS: Demostració i Configuració d'un Servidor NFS per a DevOptimize Solutions

---

<br>

## Fase 1: Preparació de l'entorn

Abans de començar la configuració dels serveis, preparem la xarxa i comprovem la connectivitat entre les màquines:
- **Servidor (Ubuntu Server):** Interfície 1 en "Xarxa NAT" i Interfície 2 en "Xarxa només-amb-amfitrió".
- **Client (Zorin OS):** Configuració de xarxa adaptable per a les proves (NAT / Només-amb-amfitrió).

Actualitzem els repositoris i sistemes a totes dues màquines:

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Fase 2: Preparació i configuració del Servidor

En aquesta fase crearem els usuaris, grups i l'estructura de directoris necessària al servidor, i instal·lerem el dimoni NFS.

### 2.1. Creació de grups i usuaris

Creem els grups `devs` i `admins` juntament amb els usuaris corresponents (`dev01` i `admin01`).

```bash
# Creació de grups
sudo groupadd devs
sudo groupadd admins

# Creació d'usuaris associats als seus grups
sudo useradd -m -g devs dev01
sudo useradd -m -g admins admin01
```

> **Nota d'alineació UID/GID:** Per tal que els permisos locals i de xarxa coincideixin sense autenticació centralitzada, cal assegurar-se que els UID i GID d'aquests usuaris siguin idèntics tant al servidor com al client.

### 2.2. Creació de directoris i assignació de permisos

Creem els directoris compartits a `/srv/nfs/` i configurem els permisos locals (UGO):
- L'usuari propietari serà `root`.
- El grup `devs` tindrà accés total a `/srv/nfs/dev_projects`.
- El grup `admins` tindrà accés total a `/srv/nfs/admin_tools`.

```bash
# Creació dels directoris
sudo mkdir -p /srv/nfs/dev_projects
sudo mkdir -p /srv/nfs/admin_tools

# Assignació de grups propietaris
sudo chown root:devs /srv/nfs/dev_projects
sudo chown root:admins /srv/nfs/admin_tools

# Configuració de permisos (770: lect/escr/exec per propietari i grup, cap accés per a altres)
sudo chmod 770 /srv/nfs/dev_projects
sudo chmod 770 /srv/nfs/admin_tools
```

### 2.3. Instal·lació del servei NFS

Instal·lem el servei de servidor NFS:

```bash
sudo apt install nfs-kernel-server -y
```

---

## Fase 3: L'Exportació d'Administració (El Dilema del root_squash)

Analitzarem el comportament de la seguretat per defecte de NFS quan l'usuari `root` del client intenta escriure en un recurs compartit.

### 3.1. Prova 1: Error comú (`root_squash` actiu per defecte)

Editem `/etc/exports` al servidor per exportar `/srv/nfs/admin_tools` amb permisos bàsics:

```bash
sudo nano /etc/exports
```

Afegim la línia:
```text
/srv/nfs/admin_tools *(rw,sync,no_subtree_check)
```

Apliquem els canvis:
```bash
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

**Al Client:** Instal·lem `nfs-common`, creem el punt de muntatge i intentem crear un fitxer com a `root`:

```bash
sudo apt install nfs-common -y
sudo mkdir -p /mnt/admin_tools
sudo mount -t nfs IP_servidor:/srv/nfs/admin_tools /mnt/admin_tools

# Intentem crear un fitxer com a root
sudo touch /mnt/admin_tools/test_root.txt
```

![Error al crear fitxer com a root](images/prova1_root_squash_error.png)

#### Resposta i Justificació Tècnica (Prova 1):
* **Resultat:** L'operació és denegada (*Permission denied*).
* **Explicació:** Per defecte, NFS s'executa amb l'opció `root_squash` activada. Aquesta mesura de seguretat mapeja qualsevol petició feta per l'usuari `root` del client cap a un usuari anònim sense privilegis (`nobody` o `nfsnobody`). Com que el directori local `/srv/nfs/admin_tools` té permisos `770` i pertany a `root:admins`, l'usuari anònim no té permís d'escriptura.

---

### 3.2. Prova 2: La Solució (`no_root_squash`)

Modifiquem `/etc/exports` al servidor per desactivar la degradació del superusuari:

```text
/srv/nfs/admin_tools *(rw,sync,no_subtree_check,no_root_squash)
```

Apliquem els canvis al servidor:
```bash
sudo exportfs -ra
```

**Al Client:** Desmuntem, tornem a muntar i provem de nou:

```bash
sudo umount /mnt/admin_tools
sudo mount -t nfs IP_servidor:/srv/nfs/admin_tools /mnt/admin_tools
sudo touch /mnt/admin_tools/test_root.txt
ls -l /mnt/admin_tools/test_root.txt
```

![Creació correcta amb no_root_squash](images/prova2_no_root_squash_success.png)

#### Resposta i Justificació Tècnica (Prova 2):
* **Resultat:** El fitxer es crea satisfactòriament i el seu propietari apareix com a `root`.
* **Explicació:** Amb `no_root_squash`, el servidor NFS confia plenament en la identitat de l'usuari `root` del client, permetent-li mantenir els privilegis de superusuari directament sobre el sistema de fitxers exportat.

---

## Fase 4: Exportació de Desenvolupament (Permisos rw vs ro i permisos de sistema)

En aquesta fase configurarem la seguretat segons la xarxa d'origen i els permisos d'usuari local.

### 4.1. Configuració d'exportació per xarxa/IP a `/etc/exports`

Editem el fitxer `/etc/exports` al servidor per definir dos nivells d'accés per al mateix directori:

```text
/srv/nfs/dev_projects 192.168.56.0/24(rw,sync,no_subtree_check) 192.168.56.150(ro,sync,no_subtree_check)
```
*(Nota: Si s'utilitza la xarxa NAT, adapteu el rang d'IPs, per exemple `10.0.2.0/24(rw,...)` i `10.0.2.100(ro,...)`)*.

Actualitzem la configuració del servidor:
```bash
sudo exportfs -ra
```

---

### 4.2. Proves d'accés des del Client

#### Prova A: Permís d'escriptura (Dins del rang rw com a `dev01`)
1. Muntem el recurs al client:
   ```bash
   sudo mkdir -p /mnt/dev_projects
   sudo mount -t nfs IP_servidor:/srv/nfs/dev_projects /mnt/dev_projects
   ```
2. Canviem a l'usuari `dev01` i intentem escriure:
   ```bash
   su - dev01
   cd /mnt/dev_projects
   touch projecte_dev.txt
   ```
* **Resultat:** Es pot crear el fitxer correctament ja que la IP està en el rang d'escriptura i l'usuari pertany al grup `devs`.

![Proves d'escriptura dev01 ok](images/prova_dev01_rw_ok.png)

---

#### Prova B: Accés des de la IP de només lectura (`ro`)
1. Desmuntem el recurs i canviem manualment la IP del client a la IP configurada com a `ro` (`192.168.56.150` o `10.0.2.100`).
2. Remuntem i intentem escriure com a `dev01`:
   ```bash
   sudo umount /mnt/dev_projects
   sudo mount -t nfs IP_servidor:/srv/nfs/dev_projects /mnt/dev_projects
   su - dev01
   touch /mnt/dev_projects/test_ro.txt
   ```
* **Resultat:** Denegat (*Read-only file system*). NFS bloqueja qualsevol modificació a nivell de xarxa.

![Error escriptura IP ro](images/prova_ip_ro_error.png)

---

#### Prova C: Validació de permisos locals (Usuari fora del grup)
1. Restablim la IP original del client amb permisos `rw`.
2. Canviem a l'usuari `admin01` i intentem escriure al projecte de desenvolupament:
   ```bash
   su - admin01
   touch /mnt/dev_projects/test_admin.txt
   ```
* **Resultat:** Denegat (*Permission denied*). Tot i que la xarxa permet escriptura (`rw`), el sistema de fitxers local bloqueja l'accés perquè `admin01` no forma part del grup propietari `devs` (permisos UGO `770`).

![Error permisos locals admin01](images/prova_admin01_ugo_error.png)

---

## Fase 5: Muntatge Automàtic amb `/etc/fstab`

Per tal de garantir la persistència dels muntatges en reiniciar els equips del client, editem el fitxer `/etc/fstab` del client:

```bash
sudo nano /etc/fstab
```

Afegim les línies següents al final del fitxer:

```text
# Entrades per als recursos NFS de DevOptimize Solutions
IP_servidor:/srv/nfs/admin_tools /mnt/admin_tools nfs defaults 0 0
IP_servidor:/srv/nfs/dev_projects /mnt/dev_projects nfs defaults 0 0
```

Comprovem la sintaxi i el muntatge sense reiniciar:

```bash
sudo mount -a
df -h | grep nfs
```

![Comprovació mount -a i df -h](images/fstab_mount_check.png)

Reiniciem la màquina client (`sudo reboot`) i comprovem que ambdues unitats romanen muntades automàticament a l'inici.

---

## Comandes de verificació utilitzades

| Comanda | Funció |
| :--- | :--- |
| `sudo exportfs -ra` | Re-exporta tots els directoris de `/etc/exports` sense reiniciar el servei |
| `showmount -e IP_servidor` | Llista els directoris compartits des del client |
| `df -h -t nfs` | Mostra els punts de muntatge NFS actius i l'espai disponible |
| `mount \| grep nfs` | Verifica les opcions de muntatge actuals |

---

## Conclusió i Recomanacions Tècniques per a DevOptimize Solutions

### Avantatges de la solució proposada
1. **Centralització de dades:** S'evita la dispersió del codi font i la documentació en local, reduint riscos de pèrdua d'informació.
2. **Control d'accés granular:** La combinació de les polítiques d'exportació de NFS (per IP/subxarxa) i les regles POSIX locals (UGO/grups) permet ajustar amb precisió qui pot llegir o escriure a cada directori.
3. **Independència del Cloud:** Es manté el control total de la infraestructura sense costos recurrents d'emmagatzematge extern.

### Limitacions identificades
1. **Manca d'autenticació centralitzada:** Al no disposar d'un directori com LDAP o Active Directory, la seguretat depèn íntegrament de la posada en correspondència manual dels UID/GID entre el servidor i els clients.
2. **Seguretat NFSv3:** El protocol utilitzat no xifra el tràfic de dades a la xarxa i confia en la IP del client, facilitant tècniques de *IP spoofing* si no es protegeix la xarxa local.
3. **Ús de `no_root_squash`:** Desactivar el *root squashing* a `admin_tools` suposa un risc de seguretat elevat si una màquina client resulta compromesa.

### Recomanacions de futur
1. **Migració a NFSv4 + Kerberos:** S'aconsella implementar NFSv4 amb xifrat i autenticació basada en tiquets Kerberos per protegir les dades en trànsit.
2. **Implantació d'un servei LDAP/FreeIPA:** Permetrà gestionar els usuaris i els seus UID/GID de forma centralitzada sense haver de crear-los manualment a cada equip.
3. **Ús d'ACLs (Access Control Lists):** Quan l'estructura d'equips creixi, les ACLs permetran assignar permisos a múltiples grups i usuaris sense les restriccions del model UGO estàndard.