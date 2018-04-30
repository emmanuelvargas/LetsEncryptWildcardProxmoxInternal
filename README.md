# LetsEncryptWildcardProxmoxInternal
Generate and renew LetsEncrypt Certificat for Promox through Gandi to an internal server (not reachable from outside)
Generation d'un certifical LetsEncrypt pour Proxmox sur un serveur non joignable depuis internet (dns-01 challenge)
English version below

Réalisé grace à
https://blog.blaisot.org/letsencrypt-wildcard-part2.html

# Voici mon architecture :
Plusieurs serveurs debian avec un proxmox dessus
Non joignables depuis l'exterieur (donc impossible de résoudre le challenge letsencrypt pour que le sereur réponde sur le port 80)
Doivent avoir un certificat letsencrypt capable de se renouveler
Le nom de domaine est géré par Gandi (v5) (fonctionne aussi sur OVH, AWS, et bind local)

# Prérequis
- Vous devrez disposer du client certbot en version 0.22.0 au minimum. à l’heure où j’écris ces lignes, cette version n’est pas encore présente sur les dépots debian ou ubuntu mais nous allons voir comment l’installer, éventuellement en parallèle d’une autre version que vous utilisez déjà. Pour vérifier votre version de certbot, utilisez certbot --version
- La possibilité d’éditer votre zone DNS. De préférence via une API ou à défaut automatisable par scripting.
- installation de proxmox déjà fonctionnel

# Installation de certbot 0.22.0
Ces instructions permettent d’installer certbot sur la plupart des systèmes unix/linux dans /opt y compris si vous avez déjà une autre version de certbot installée par les packages de votre distribution sans y toucher. Cela vous permettra de tester la génération de certificat wildcard pour être prêt dès que cette version sera disponible dans votre repo/ppa préféré.

Créez un répertoire /opt/certbot pour le wrapper de téléchargement/mise à jour de certbot
```
$ mkdir /opt/certbot
$ cd /opt/certbot
```
Téléchargez le wrapper sur le site de certbot
```
$ wget https://dl.eff.org/certbot-auto
$ chmod a+x certbot-auto
```
Lancez l’installation
```
$ ./certbot-auto --install-only
```
Et voilà, certbot 0.23.0 (ou plus récent) est installé dans /opt/eff.org/certbot comme nous le prouve la commande suivante :
```
$ /opt/eff.org/certbot/venv/bin/certbot --version
certbot 0.23.0
```
# Génération de certificat wildcard avec scripts maison d’édition de zone
## Déploiement des scripts

Commencez par cloner les scripts de sblaisot ces scripts dans /opt/le-scripts :
```
 $ git clone https://github.com/sblaisot/certbot-dns-01-authenticators.git /opt/le-scripts
 ```
 4 ensembles de scripts sont disponibles, le premier pour tester le fonctionnement de certbot avec des scripts de validation personnels, le second si vous avez votre serveur DNS bind sur la même machine que celle avec laquelle pour générez les certificats et les deux derniers pour générer des certificats wildcard letsencrypt lorsque vos zones DNS sont gérées sur le service LiveDNS de Gandi ou chez OVH.

Chaque ensemble est composé de 2 scripts : un script auth qui permet de créer l’entrée DNS pour la validation et un script cleanup qui permet de la retirer une fois la validation effectuée.

## Un script pour tester
Les scripts du répertoire test se contentent d’afficher le contenu des variables d’environnements passées par certbot, pour en étudier le fonctionnement. De ce fait, ils ne créent ni ne suppriment aucune entrée DNS et la génération de certificat échoue forcément.

Néanmoins, ils nous permettent de voir comment cela fonctionne. Appelons certbot en lui demandant d’utiliser ces scripts :
```
$ cd /opt/eff.org/certbot/venv/bin
$ ./certbot certonly \
    --server https://acme-staging-v02.api.letsencrypt.org/directory \
    --manual-public-ip-logging-ok \
    --manual \
    --manual-auth-hook /opt/le-scripts/test/auth.sh \
    --manual-cleanup-hook /opt/le-scripts/test/cleanup.sh \
    -d '*.domain.tld'
   ``` 
Nous voyons ici que les scripts sont appelés avant la validation pour créer l’entrée DNS et après la validation pour la supprimer. Ces scripts reçoivent bien les variables d’environnement de certbot.

## Génération de certificat wildcard avec DNS Gandi (LiveDNS)
Maintenant qu’on a bien joué avec nos scripts de test, voici un exemple plus près de la vie réelle pour la génération d’un certificat wildcard lorsque la zone DNS est hébergée chez Gandi.

Il est indispensable que votre zone DNS soit hébergée sur la solution LiveDNS de gandi (v5) qui permet une mise à jour des enregistrement DNS instantanée contrairement à l’ancienne solution (v4). Dans l’ancienne version, il fallait plusieurs minutes pour que vos enregistrements DNS soient créés et ceci n’est pas compatible avec la validation de domaine.

Ensuite, il vous suffit dans le répertoire /opt/le-scripts/gandi-livedns de copier le fichier config.py.example en config.py et d’ajuster les paramètres :

- livedns_api = "https://dns.api.gandi.net/api/v5/" c’est le endpoint de l’API LiveDNS de Gandi. Il n’est habituellement pas nécessaire de modifier ce paramètre
- livedns_apikey = "YOUR-KEY-HERE" Votre clé d’API LiveDNS que vous pourrez générer dans vos paramètres de compte à la rubrique “Modification de mot de passe et configuration des restrictions d’accès“
- livedns_sharing_id = None Si vos domaines ne sont pas rattachés directement à votre compte mais gérés dans une organisation, il faudra indiquer ici votre sharing_id qui permet de gérer l’organisation à partir de votre compte. Vous le retrouverez dans l’URL du manager lorsque vous listez votre domaine sous la forme d'un GUID.

Une fois la configuration effectuée, il ne vous reste plus qu’à générer votre certificat :
```
cd /opt/eff.org/certbot/venv/bin
$ ./certbot certonly \
    --server https://acme-v02.api.letsencrypt.org/directory \
    --manual-public-ip-logging-ok \
    --manual \
    --manual-auth-hook /opt/le-scripts/gandi-livedns/auth.py \
    --manual-cleanup-hook /opt/le-scripts/gandi-livedns/cleanup.py \
    -d '*.domain.tld'
```
## Copie des certificats sur proxmox

```
cp /etc/letsencrypt/live/domain.ltd/fullchain.pem /etc/pve/local/pve-ssl.pem 
cp /etc/letsencrypt/live/domain.ltd/privkey.pem /etc/pve/local/pve-ssl.key 
systemctl restart pveproxy
```

## copie des certificats sur les autres serveurs proxmox:

```
scp /etc/letsencrypt/live/domain.ltd/fullchain.pem root@$IP1:/etc/pve/local/pve-ssl.pem
scp /etc/letsencrypt/live/domain.ltd/privkey.pem root@$IP1:/etc/pve/local/pve-ssl.key
ssh root@$IP1 "systemctl restart pveproxy"
```
on copie pour chaque serveur proxmox de votre réseau.
On fera un script pour le renouvellement automatique ci dessous

## crontab pour le renouvellement automatique (renew)

### création d'un script de copie

ce script sera executé à la fin du processus de renouvellement de certbot

```
# cat renew_cert.sh 
#!/bin/bash

SERVER="192.168.0.151 192.168.0.152 192.168.0.153"
DEBUG=1

## ---- DO NOT CHANGE AFTER HERE

USAGE="Usage: $BASENAME [OPTIONS]
A self script to copy the certificat renewed by the crontab to all impacted servers.
Crontab is
30 6 1,15 * * root /opt/eff.org/certbot/venv/bin/certbot renew --quiet --post-hook /opt/vargas.scripts/renew_cert.sh

where /opt/vargas.scripts/renew_cert.sh is this script 

Help

	--debug			Print more information
	--help			This help

Beta."

for arg in "$@" ; do
  case "$arg" in
    --debug)
      DEBUG=1;;
    --help)
      HELP=1;;
  esac
done

error() {
    echo "$@"
}

function debug() { ((DEBUG)) && echo "### $*"; }

if [ "$HELP" = 1 ]; then
	echo "$USAGE"
    exit 0
fi

debug "Copy cert file to local proxmox pveproxy and restart it"

/bin/cp /etc/letsencrypt/live/vargas.one/fullchain.pem /etc/pve/local/pve-ssl.pem 
/bin/cp /etc/letsencrypt/live/vargas.one/privkey.pem /etc/pve/local/pve-ssl.key 
/bin/systemctl restart pveproxy

debug "Servers to renew cert : $SERVER :"

for i in $SERVER; do
    debug "Server : $i :"
    /usr/bin/scp -i /root/.ssh/id_rsa -q /etc/letsencrypt/live/vargas.one/fullchain.pem root@$i:/etc/pve/local/pve-ssl.pem
    /usr/bin/scp -i /root/.ssh/id_rsa -q /etc/letsencrypt/live/vargas.one/privkey.pem root@$i:/etc/pve/local/pve-ssl.key
    /usr/bin/ssh -i /root/.ssh/id_rsa -q root@$i "systemctl restart pveproxy"
done

debug "done"
```

### entrée la crontab suivante avec un crontab -e

```
37 5 1,15 * * cd /opt/eff.org/certbot/venv/bin/ && ./certbot renew --post-hook /opt/vargas.scripts/renew_cert.sh
```

### modification de votre fichier /etc/hosts

Bien sur vous l'aurez compris il faut que vos serveur existe chez gandi
Pour ma part chez des hosts qui correspondent chez gandi (proxmox01.domain.ltd) et qui pointe vers une adresse IP et j'ai modifié mon host local pour faire pointer proxmox.domain.ltd vers l'IP local de mon reseau.
(il est aussi possible de faire pointer les URL de gandi vers vos ip interne mais ce n'est pas une bonne pratique)
