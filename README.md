# NF1UD4 - Gestió i Compartició de Fitxers per Xarxa

---

<br>

## Pas 1: Preparació i instal·lació de dependències

Abans de posar-nos a configurar res, comprovem que els sistemes tenen sortida a Internet i actualitzem els llistats de paquets.

### 1.1. Al Servidor (Ubuntu Server)

Procedim a descarregar i instal·lar el paquet principal que dóna vida al servei NFS:

```bash
sudo apt update
sudo apt install nfs-kernel-server -y
```

### 1.2. Al Client (Zorin OS)

Per la banda del client, instal·lem les llibreries i eines necessàries per poder gestionar muntatges NFS:

```bash
sudo apt update
sudo apt install nfs-common -y
```

---

## Pas 2: Posada a punt del Servidor NFS

En aquest punt deixarem a punt el directori que volem exportar, definint correctament les polítiques d'accés i la configuració del servei.

### 2.1. Creació d'usuaris, grups i directori a compartir

Donarem d'alta un grup i un compte d'usuari específics per a la pràctica, i els farem responsables de la carpeta que exposem a la xarxa.

```bash
# Creem el grup i l'usuari que serà propietari del recurs (UID i GID 1100)
sudo groupadd -g 1100 grup_smx
sudo useradd -u 1100 -m -g grup_smx usuari_smx

# Crear una carpeta per compartir
sudo mkdir -p /srv/nfs/compartit

# Canviar propietari i grup de la carpeta
sudo chown usuari_smx:grup_smx /srv/nfs/compartit

# Establir permisos d'accés a la carpeta (rwx per al propietari, rx per al grup)
sudo chmod 750 /srv/nfs/compartit
```

### 2.2. Definició de l'exportació (`/etc/exports`)

L'arxiu `/etc/exports` s'encarrega d'establir quins directoris s'ofereixen a la xarxa, qui hi pot accedir i amb quins privilegis.

Obrim l'arxiu per editar-lo:

```bash
sudo nano /etc/exports
```

Afegim la línia següent al final de tot (permetent connexions des de qualsevol origen `*` amb permisos de lectura/escriptura i sincronització activada):

```text
/srv/nfs/compartit *(rw,sync,no_subtree_check)
```

> **Detall de cada paràmetre:**
> - `rw`: Concedeix permisos tant de lectura com d'escriptura.
> - `sync`: Obliga el servidor a confirmar que les dades s'han gravat al disc abans de respondre a la petició (assegura la integritat).
> - `no_subtree_check`: Evita comprovacions innecessàries de subàrbres, guanyant agilitat en el rendiment.

### 2.3. Aplicació de la configuració i comprovació del servei

Perquè el sistema carregue les noves regles, executem les següents comandes:

```bash
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

Comprovem que el dimoni està actiu i sense incidències:

```bash
sudo systemctl status nfs-kernel-server
```

---

## Pas 3: Connexió i configuració al Client NFS

### 3.1. Sincronització d'identificadors (UID/GID)

```bash
sudo groupadd -g 1100 grup_smx
sudo useradd -u 1100 -m -g grup_smx usuari_smx -s /bin/bash
```

### 3.2. Muntatge temporal del recurs

Generem el directori local on penjarà la unitat remota i fem el muntatge:

```bash
sudo mkdir -p /mnt/compartit
sudo mount -t nfs servidor_nfs:/srv/nfs/compartit /mnt/compartit
```

### 3.3. Verificació del funcionament

Comprovem que l'usuari local té capacitat d'operar dins del directori compartit:

```bash
su - usuari_smx

cd /mnt/compartit

touch fitxer_prova.txt

ls -l fitxer_prova.txt
```

*Si el fitxer es genera correctament i sense avisos de permisos denegats, tot rutlla.*

Un cop fetes les comprovacions, sortim de la sessió de l'usuari (`exit`) i desmuntem el directori:

```bash
sudo umount /mnt/compartit
```

---

## Pas 4: Automatització del muntatge (`/etc/fstab`)

El mètode anterior és volàtil i desapareix en reiniciar. Per aconseguir que es munti de forma automàtica cada vegada que arrenqui l'equip, modifiquem el fitxer `/etc/fstab` del **client**.

```bash
sudo nano /etc/fstab
```

Afegim la línia següent al final del fitxer (recordeu canviar `IP_servidor_nfs` per la IP corresponent):

```text
IP_servidor_nfs:/srv/nfs/compartit /mnt/compartit nfs defaults 0 0
```

**Significat de cada camp:**

| Camp | Descripció |
|------|------------|
| `IP_servidor_nfs:/srv/nfs/compartit` | Localització de la font compartida al servidor |
| `/mnt/compartit` | Punt de destinació al sistema local |
| `nfs` | Tipus de tecnologia de fitxers emprada |
| `defaults` | Paràmetres predeterminats de muntatge |
| `0` | Indica que no es farà bolcat de dades (`dump`) |
| `0` | Indica que no s'aplicarà comprovació d'errors d'arrencada (`fsck`) |

Per validar que la sintaxi és correcta sense haver de reiniciar la màquina, llancem:

```bash
sudo mount -a
```

Si el sistema no retorna cap missatge d'error, el punt automàtic ha quedat ben configurat.

---

## Comandes útils de verificació

| Instrucció | Descripció |
|------------|------------|
| `showmount -e IP_servidor` | Llista els recursos que té oberts el servidor |
| `mount \| grep nfs` | Mostra els sistemes de fitxers NFS actius actualment |
| `df -h` | Ofereix una vista de l'espai d'emmagatzematge de les unitats muntades |
| `cat /etc/exports` | Mostra ràpidament el contingut de les exportacions actuals |