# Création d'un OS maison centré sur la sécurité, la virtualisation et un peu de fun

Voici un guide faisant partie d'une série, sur l'installation d'un système compartimenté et centralisé sur la virtualisation. C'est une installation pas à pas d'un système plutôt complexe.
Je vous recommanderai d'avoir des connaissances sur Linux (avoir au moins installé Arch à la main), la virtualisation et la sécurité avant de suivre ce guide qui ne sera pas
toujours parfait. Également, vous pouvez aussi vous tourner vers des solutions existantes comme QubesOS.

## C'est quoi un OS compartimenté ?
Aujourd'hui l'informatique à beaucoup évolué, a tel point que nos ordinateurs sont assez puissant pour utiliser plusieurs systèmes en même temps. Avec un ordinateur solide et performant, on a moyen de se créer un système vraiment complet et sécurisé.

Par exemple: 
- Windows 10 et 11 acceuillent la solution WSL (windows subsystem for linux)
- Des solutions logicielles populaire pour la virtualisation de systèmes (virtualbox, vmware...)
- Des OS complet centrés sur la virtualisation (on parle parfois d'hyperviseurs)
- La conteneurisation via Docker

> Toutes ces solutions permettent d'exécuter plusieurs OS en même temps et sur une même machine sans passer par un redémarrage de celle-ci.

En fait, il existe déjà une solution populaire (que Snowden recommande, rien que ça !) qui permet de compartimenté son OS: **QubesOS**

QubesOS repose sur l'hyperviseur de type 1 Xen, ça permet une haute isolation entre l'hyperviseur et les machines virtuelles.
C'est une solution que je conseille si vous voulez pas trop vous casser la tête pour l'installation.

En revanche:
- QubesOS dépend de la VM Dom0 pour afficher un environnement graphique (Dom0 utilise le vieux serveur graphique Xorg :/)
- Utilisation et administration du système parfois + compliqué
- Performances pas toujours au rendez-vous
- Moins flexible et moins "à la carte" que KVM/Qemu

## Mon choix

Je vais donc partir sur une base Linux minimaliste : **Artix Linux**.

- **Virtualisation lourde :**  
  J’ajouterai **QEMU, KVM et libvirt** (KVM est déjà intégré au noyau Linux) pour faire tourner des OS complets, Linux ou Windows, pour différentes tâches.  

- **Environnement graphique :**  
  **Hyprland** sera utilisé pour le bureau.  
  1. C’est joli 😄  
  2. Basé sur **Wayland**, qui a moins de couches d’attaque que Xorg.  

> **À venir : VFIO passthrough**  
> Lorsque j’aurai une carte graphique, j’utiliserai la technique du **VFIO passthrough** pour :  
> - Une VM gaming (accès direct au GPU pour des performances quasi natives)  
> - Une VM d’expérimentation pour l’IA  
> - Déjà dès maintenant, sur une VM **MAO** (Musique Assistée par Ordinateur) pour brancher ma guitare sur une interface audio.  

⚠️ **Explications :** Le VFIO passthrough est une **technique du noyau Linux** permettant de donner un périphérique PCI directement à une VM, pour des performances quasi natives.


 ## Pourquoi Artix Linux ? 
 - Basé sur Arch Linux (le meilleur compromis entre programme compilés avec AUR et les binaires)
 - Rolling Release -> donc màj assez fréquentes
 - Sans systemD (systemd est devenu un système d'init lourd avec le temps bien qu'efficace)
 - version de base avec le strict minimum pour installer un Linux
 - (J'aime bien cette proposition d'arch et j'aime peut-être me compliquer la vie xD)

 Ces points montrent qu'Artix Linux est un OS assez sécurisé et surtout modulaire de par sa conception.
 
> On peut évidemment ajouter plus de programme pour sécuriser tout ça (firewall, AppArmor, SElinux)

## L’intérêt du setup

**But recherché** :  
- L’hôte = une base Linux solide, stable et minimaliste  
- L’hyperviseur **QEMU/KVM** s’occupe du cloisonnement et de la virtualisation  
- S'approcher d'une philosophie type QubesOS mais avec des meilleurs performances.

⚠️ La distribution Linux de l’hôte n’est pas critique en soi, tant que vous partez sur une base **fiable et connue** (évitez les ISO obscures !).  

> Pour de plus amples informations à propos de mon matos physique et du BIOS Libre Coreboot, voir ici.

# Procédure d'installation

## ISO et Live USB
Il faut évidemment récupérer l'ISO officiel d'Artix Linux sur le site (via les miroirs les plus proches) et créer notre clé Bootable, le plus rapide sera avec la commande 'dd'

```Bash
dd if=artix-base-{votre_init_choisi}-{date_de_dernière_release}-x86_64.iso of=/dev/sd{x} status=progress bs=4M; sync
```

Une fois la commande `dd` terminé (ça prend un petit moment) on peut débrancher notre clé puis la préparer sur notre ordinateur "cible". Sur Coreboot au démarrage, on peut accéder au
options de Boot avec la touche Échap.

Et voici un des soucis que je rencontre avec mon moniteur, la console "déborde" de l'écran.

Pas de panique ! il va falloir oeuvrer un peu à l'aveugle, quand ça a fini de défiler (ce n'est pas très long) il faut se connecter, user=artix password=artix par défaut sur le
live usb de cette distribution.

En faisant entré pour l'utilisateur puis le mot de passe, on pourra faire un clear ou un CTRL-L pour afficher notre invite de commande.

Maintenant on peut voir ce qu'on tape, mais ce n'est pas fini on doit régler la taille de police du terminal et le nombre de lignes et colonnes qu'on veut afficher pour ne plus
faire déborder la console de notre écran.

```bash
sudo setfont /usr/share/kbd/consolefonts/{policedevotrechoix}
stty rows 30 cols 100
```
Voila maintenant on peut travailler comme il faut.

### Partitionnement

J'utiliser `sudo cfdisk` pour faciliter notre Partitionnement de disque.
Une fois dessus, on va tout supprimer et repartir sur une base saine.

- Faites une partition EFI de 512 Mo ou plus type EFI system pour démarrer en UEFI
- Et une seconde pour le reste du système

Dans mon cas je n'utilise pas de swap. 32Go de Ram physique sont suffisant.

> ⚠️ N'oubliez pas d'utiliser l'option "Écrire" pour enregistrer votre partitionnement.

Hors cfdisk on peut formater la partition EFI, cette partition nous permettra de booter sur notre distribution Linux:

```Bash
mkfs.fat -F32 /dev/sdX1
```
Ensuite, la partie probablement la plus compliquée **"LUKS+BTRFS"** LUKS nous permettra d'encrypter notre disque, et BTRFS est un système de fichiers qui est idéal pour les SSD, et
la possibilité de créer des snapshots ou sauvegarde.

```Bash
cryptsetup luksFormat /dev/sdX2 -> suivez les instructions puis choisissez votre mdp
cryptsetup open /dev/sdX2 cryptroot -> on ouvre et mappe notre partition
```
Une fois la partition luks chiffrée, a partir de maintenant, on utilisera le "mappage de partition" `/dev/mapper/cryptroot`, on formate en Btrfs puis on monte la partition sur /mnt 

```Bash
mkfs.btrfs /dev/mapper/cryptroot
mount /dev/mapper/cryptroot /mnt
```
On crée ensuite les sous-volumes Btrfs qui nous aideront a créer des snapshots des différentes parties de notre système (Home, var, etc...)

```Bash 
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@var
umount /mnt
```

Les sous-volumes créer, on peut démonter /mnt puis monter les sous-volumes nouvellement créés.

```Bash
mount -o noatime,compress=zstd,space_cache=v2,subvol=@ /dev/mapper/cryptroot /mnt
mkdir -p /mnt/{boot,home,var,.snapshots}

mount -o noatime,compress=zstd,space_cache=v2,subvol=@home /dev/mapper/cryptroot /mnt/home
mount -o noatime,compress=zstd,space_cache=v2,subvol=@var /dev/mapper/cryptroot /mnt/var/
mount -o noatime,compress=zstd,space_cache=v2,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots

mount /dev/sdX1 /mnt/boot

```
Il faut inclure toutes les options pour les sous-volumes btrfs.

Tout est monté ! On peut procéder a l'installation du système.

### Installation du Système
Je vais poursuivre cette installation en me référant a la méthode d'artix linux, Arch Linux a déjà plein de tutos en plus d'un wiki très complet sur le site d'Arch.

Artix Linux n'est pas encore bien connu et la documentation n'est pas toujours nette (je me souviens devoir naviguer entre 3 docs pour installer proprement mon système.)


À ce stade on ne pourra peut-être pas installer no paquets tout de suite, il faut rafraichir le trousseau GPG sinon, basetrap et pacman n'installeront rien. Pour ce faire:

```Bash
rm -rf /etc/pacman.d/gnupg
pacman-key --init

pacman-key --populate artix
pacman -Sy artix-keyring archlinux-keyring
pacman-key --populate artix
pacman-key --refresh-keys
```
Voila les signatures PGP sont rafraichies, on peut installer les paquet nécessaires.
On utilise `basetrap` pour installer les paquets sur un point de montage

```Bash
basetrap /mnt base base-devel dinit elogind-dinit neovim grub efibootmgr btrfs-progs vim linux(-kerneldésiré) linux-firmware cryptsetup mkinitcpio artix-archlinux-support
```
Une fois tout installé, on peut générer le fstab puis "chrooter" sur notre nouveau Système.
> Le chroot est une commande permettant d'entrer dans un Système linux


```Bash
genfstab -U /mnt >> /mnt/etc/fstab
artix-chroot /mnt
```

## Configuration de base

Maintenant que nous sommes dans notre nouveau système grâce au `artix-chroot`, on va effectuer la configuration de base. C'est ce qui permettra à notre installation de démarrer correctement et d'avoir les bons paramètres dès le premier boot.

### Fuseau horaire

On commence par définir notre fuseau horaire. Dans mon cas je suis en France.

```bash
ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
hwclock --systohc
```

La première commande crée un lien vers le bon fuseau horaire, la seconde synchronise l'horloge matérielle avec l'heure système.

---

### Configuration de la langue

On va maintenant configurer les locales.

Éditez le fichier :

```bash
nvim /etc/locale.gen
```

Décommentez la ligne correspondant à votre langue, par exemple :

```
en_US.UTF-8 UTF-8
fr_FR.UTF-8 UTF-8
```

Puis générez les locales :

```bash
locale-gen
```

Définissez ensuite la locale utilisée par défaut :

```bash
echo "LANG=fr_FR.UTF-8" > /etc/locale.conf
```

Libre à vous d'utiliser une locale anglaise si vous préférez avoir les messages système en anglais.

---

### Nom de la machine

On choisit maintenant un nom pour notre ordinateur.

```bash
echo "artix-host" > /etc/hostname
```

Ensuite, on complète le fichier `/etc/hosts` :

```bash
nvim /etc/hosts
```

En ajoutant :

```
127.0.0.1   localhost
::1         localhost
127.0.1.1   artix-host.localdomain artix-host
```

Remplacez évidemment **artix-host** par le nom que vous avez choisi.

---

### Clavier de la console

Même si nous utiliserons plus tard un environnement graphique, autant avoir un clavier correctement configuré dès maintenant.

```bash
echo "KEYMAP=fr" > /etc/vconsole.conf
```

Si vous utilisez un clavier différent, adaptez simplement la valeur.

## Création des utilisateurs

Il est fortement déconseillé d'utiliser le compte **root** au quotidien.

On commence par définir son mot de passe :

```bash
passwd
```

Puis on crée un utilisateur classique.

```bash
useradd -m -G wheel -s /bin/bash votre_utilisateur
```

On lui attribue ensuite un mot de passe :

```bash
passwd votre_utilisateur
```

Le groupe **wheel** permettra d'utiliser les privilèges administrateur, tandis que **libvirt** et **kvm** seront utiles lorsque nous installerons notre environnement de virtualisation.

---

### Installation de sudo

On installe ensuite sudo.

```bash
pacman -S sudo
```

Puis on édite sa configuration.

```bash
EDITOR=nvim visudo
```

Décommentez la ligne suivante :

```
%wheel ALL=(ALL:ALL) ALL
```

Les membres du groupe **wheel** pourront ainsi exécuter des commandes administrateur avec `sudo`.

Si vous préférez `doas`, libre à vous de l'utiliser, mais la majorité des documentations utilisent `sudo`.

## Configuration de mkinitcpio

Comme notre système utilise un disque chiffré avec **LUKS** ainsi que **Btrfs**, il faut maintenant vérifier que l'image de démarrage contient les bons modules.

Éditez le fichier :

```bash
nvim /etc/mkinitcpio.conf
```

Les hooks doivent contenir au minimum les éléments nécessaires au chiffrement et au système de fichiers.

Par exemple :

```
HOOKS=(base udev autodetect microcode modconf kms block keyboard keymap consolefont encrypt filesystems btrfs fsck)
```

⚠️ Selon votre version d'Artix, de mkinitcpio ou votre système d'init, les hooks peuvent légèrement varier. N'hésitez pas à consulter la documentation officielle si nécessaire.

Une fois les modifications effectuées, on régénère l'image de démarrage.

```bash
mkinitcpio -P
```

Si aucune erreur n'apparaît, tout est prêt pour démarrer un système chiffré.

## Installation du chargeur de démarrage

Il ne reste plus qu'à installer GRUB afin que notre ordinateur puisse démarrer sur notre nouvelle installation.

Installation sur la partition EFI :

```bash
grub-install \
    --target=x86_64-efi \
    --efi-directory=/boot \
    --bootloader-id=GRUB
```

Comme notre système est chiffré, GRUB doit connaître l'identifiant de la partition LUKS.

On récupère son UUID :

```bash
blkid /dev/sdX2
```

Copiez la valeur de l'UUID puis éditez :

```bash
nvim /etc/default/grub
```

Repérez la ligne :

```
GRUB_CMDLINE_LINUX_DEFAULT=
```

et ajoutez les paramètres nécessaires au déverrouillage du disque, par exemple :

```
cryptdevice=UUID=<UUID>:cryptroot root=/dev/mapper/cryptroot
```

Remplacez bien `<UUID>` par celui obtenu précédemment.

Enfin, on génère la configuration finale de GRUB.

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

Si aucune erreur n'est affichée, notre système est maintenant capable de démarrer, de déverrouiller automatiquement la partition LUKS et de lancer Artix Linux.

Il ne reste plus qu'a installer les premiers programmes ainsi que l'interface graphique souhaité. Pour ma part j'ai choisi hyprland avec DankMaterialShell, mais vous pouvez prendre
ce que vous souhaiter.

```bash
pacman -S git curl wget networkmanager snapper qemu-full libvirt virt-manager kitty apparmor apparmor-utils nftables polkit hyprland xdg-desktop-portal-hyprland pipewire
pipewire-alsa pipewire-pulse pipewire-jack wireplumber mesa vulkan-radeon ...
```

On peut redémarrer et si tout se passe bien vous avez un système opérationnel et prêt à créer des machines virtuelles.

Ceci conclut cette 1ere longue partie, prochainement nous verrons les options de libvirt et le VFIO passthrough.
