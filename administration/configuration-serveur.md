# Configuration d’un serveur virtuel sécurisé en local

Prérequis : une machine avec une distribution de Linux avec SSH et Apache installés. Nous allons partir du principe que nous allons configurer le serveur non pas directement mais en y accédant via SSH depuis notre machine. Nous travaillerons avec un serveur situé sur une VirtualBox.

Avant de quitter le serveur après l’installation, nous récupérons son adresse IP :

```
ip a

inet 10.0.2.15/24
```

Depuis la machine distante :

```
ssh cnam@10.0.2.15
```

On rentre le mot de passe et on est connecté.

Note : ici sur une machine VirtualBox ça ne fonctionnera pas tel quel, mais c’est le principe dans la vraie vie.

En local, le plus simple est de mettre en place la redirection de ports dans les options avancées de l’onglet réseau de VirtualBox.

Le principe est de rediriger la machine locale vers la machine virtuelle par le biais d’une association d’IP et de ports.

| IP hôte        | Port hôte     | IP invité  | Port invité   |
|:--------------:|:-------------:|:----------:|:-------------:|
| 127.0.0.1      | 8080          | 10.0.2.15  | 80            |
| 127.0.0.1      | 2222          | 10.0.2.15  | 22            |

Ainsi, faire appel à localhost:8080 dans la barre d’adresse revient à faire un localhost sur la machine virtuelle, et on peut se connecter en SSH de la manière suivante :

```
ssh cnam@localhost -p 2222
```

On interroge le port 2222 de notre localhost et grâce à la redirection on arrive sur le port 22 de notre machine virtuelle. C'est assez pratique.

Note : une bonne pratique serait de changer le port 22 sur le serveur et de le remplacer par n’importe quel port disponible (par exemple 5678). Cela ne l’empêcherait pas d’être rapidement trouvé par quelqu’un qui le cherche, mais le port 22 étant de fait particulièrement ciblé, c’est plutôt une bonne chose de le changer. Cela se fait au niveau du fichier /etc/ssh_config. 

Note : il est tout à fait possible de déclarer plusieurs ports.

Note : il est possible de limiter l’accès à certaines adresses IP (attention quand même, en cas de changement ou de perte d’adresse IP on peut s’enfermer dehors.)

Plutôt qu’un accès par mot de passe, nous pouvons mettre en place une identification par clés. Le principe est de générer des clés d’identification sur notre poste de travail, et d’envoyer la clé publique sur le serveur. 

### Génération des clés sur notre poste

```
ssh-keygen -t dsa -b 4096 -c "website"
```
- -t permet de définir le type de clé (rsa/dsa)
- -b permet de définir la force (nombre de bits) de l’encodage (minimum 1024, 2048 ou 4096 pour être tranquille)
- -c permet d’ajouter un commentaire, ce qui peut aider à s’y retrouver si on doit gérer plusieurs clés. Ici nous mettons le nom du projet.

Si aucune option n’est renseignée, ssh-keygen va poser des questions et on pourra renseigner ses choix à la volée.

### Transfert de la clé sur le serveur


Pour copier la clé sur le serveur, il faut utiliser la commande ssh-copy-id dont la syntaxe est la suivante :

```
ssh-copy-id -i ~/.ssh/mykey user@host
```
Le dossier .ssh est automatiquement créé s’il n’existe pas sur la machine cible. Le fichier authorized_keys est édité ou automatiquement créé s’il n’existe pas.


Quelle que soit la méthode retenue, nous pouvons à présent nous connecter à notre serveur de manière sécurisée.

## Création du répertoire et d’une page de test

```
cd /var/www/html
mkdir website
cd website
echo 'hello' > index.html
```

Nous créons par commodité un dossier au sein du répertoire /var/www, auquel Apache a accès. Si l’on souhaite placer les fichiers ailleurs dans l’arborescence, il faudra s’assurer que l’utilisateur Apache (User Apache / Group Apache) puisse y accéder.

Plusieurs choix sont possibles :

- On peut ajouter l’utilisateur qui héberge les fichiers au groupe Apache
- On peut intervenir sur les permissions pour que les fichiers soient accessibles en lecture et en exécution. Attention, le home directory de l’utilisateur, par défaut, a un accès restreint.
- Enfin on peut simplement utiliser un lien symbolique. L’utilisateur héberge le dossier mais on crée un lien symbolique et on autorise Apache à le suivre.

```
ln -s /home/cnam/www /var/www/html/website
``` 

Dans ce cas le dossier website n’existe pas, c’est un lien symbolique vers /home/cnam/www où se trouvent les fichiers. 

## Ajout d’un alias dans le fichier hosts pour résolution DNS locale
```
vim /etc/hosts

127.0.0.1   localhost   website
```

`ping website` doit être concluant

## Sécurisation https avec openssl
Nous allons utiliser openssl pour sécuriser notre site et pouvoir y accéder via le protocole https.

### On génère la clé

```
cd /etc/httpd/conf
mkdir sslkeys
cd sslkeys
openssl genrsa -out website.key 1024
```

### On génère le certificat
```
openssl req -new -x509 -days 365 -key website.key -out website.crt
```
Répondre aux questions.
Attention: pour le champ « Common Name (eg, YOUR name) : », il faut mettre le même nom que le serveur, dans notre exemple website.

On peut vérifier le certificat de la manière suivante :

```
openssl x509 -in website.crt -text -noout
```

## Création d’un fichier de configuration dédié
Nous pouvons à présent configurer un serveur virtuel pour notre projet.

```
vim /etc/httpd/conf/vhosts.d/website.conf

<VirtualHost *:443>
    ServerAdmin admin@website.fr
    ServerName website
    DocumentRoot /var/www/html/website

    SSLEngine on
    SSLCertificateFile /etc/httpd/conf/sslkeys/website.crt
    SSLCertificateKeyFile /etc/httpd/conf/sslkeys/website.key
</VirtualHost>
```

## Rédemarrage d’Apache
```
systemctl restart httpd
```

Notre contenu est désormais accessible sur : https://website.
Le navigateur affiche un avertissement car le serveur s’autocertifie.
On peut ajouter une exception car nous savons que nous ne sommes pas méchant.

Pour un usage autre que personnel, il faudrait solliciter un certficat auprès d’un organisme de confiance.

Pour information, OVH inclut un certificat Let’s encrypt avec ses hébergements, adapté à un site personnel. Pour un site professionnel, compter 50 euros HT/an. (Et dans ces cas-là rien à configurer, juste activer l’option – et payer, éventuellement.)

## Amélioration de la sécurité

À présent que tout fonctionne, nous pouvons ajuster la sécurité selon nos besoins dans la configuration de notre virtualhost.

```
Options -Indexes +FollowSymlinks
```
voire
```
Options none
```
sont des options à considérer attentivement.

Rappelons en effet que par défaut nous héritons de la configuration globale d’Apache, et que `Options Indexes` affiche le contenu du répertoire si aucun fichier index.html (par défaut) n’est présent, ce que l’on ne souhaite pas forcément. L’option `FollowSymLinks quant à elle permet à Apache de suivre les liens symboliques pour aller chercher dossiers ou fichiers ailleurs dans le système. Ce qui est toujours à considérer avec prudence.

De manière générale, on préfèrera commencer par fermer les vannes, et ouvrir seulement les robinets dont on a besoin.

Nous pouvons également, pour des besoins spécifiques, n’accorder l’accès que sur identification par login et mot de passe (ce que nous pourrions également faire directement depuis le code de notre application, avec plus de marge de manœuvre sur la partie graphique de l’interface de connexion, c’est un choix à faire, peut-être un arbitrage entre la simplicité côté serveur, et la convivialité côté application).

Voici un exemple de mise en place de sécurisation par mot de passe :

```
<VirtualHost *:443>
    ServerAdmin admin@website
    ServerName website
    DocumentRoot /var/www/html/website

    <Directory /var/www/html/website>
        Options none
        AllowOverride none
        AuthType Basic
        AuthName private
        AuthUserFile /home/cnam/users
        Require valid-user
    </Directory>
    
    SSLEngine on
    SSLCertificateFile /etc/httpd/conf/sslkeys/website.crt
    SSLCertificateKeyFile /etc/httpd/conf/sslkeys/website.key
</VirtualHost>
```
L’utilisateur peut lui-même gérer les accès. Création d’un nouvel accès :

Dans le répertoire /home/cnam :
````
htpasswd -c users robert
````

Attention aux droits. Apache doit pouvoir accéder à ce fichier. Les droits par défaut de /home ne le permettent pas.

## Transfert des fichiers de notre site

À présent que notre serveur est configuré selon nos souhaits, nous allons transférer les fichiers de notre site via le protocole ftp. On peut vérifier la présence d’un serveur ftp sur la machine avec la commande nmap. Le port 21 doit apparaître. Si aucun serveur ftp n’est installé, installer proftpd.

Par défaut la connexion en root n’est pas autorisée, ce qui est une bonne chose.

Mais par défaut, un utilisateur connecté via ftp peut remonter dans l’arborescence, ce qui n’est pas souhaitable.

Il faut modifier le fichier de configuration.

```
vim /etc/proftpd.conf

# On ajoute la directive suivante qui cantonne 
# chaque utilisateur à son dossier personnel
DefaultRoot ~
```

Et tant que l’on est dans le fichier on peut également ajouter le commentaire suivant :

```
# Include /etc/proftpd-anonymous.conf
```

Il nous suffira de décommenter pour activer les sessions anonymes si nous le souhaitons (pour un accès à des fichiers se trouvant par défaut dans /var/ftp/pub).

À l’instar l’Apache, proftpd permet la configuration de serveurs ftp via des virtualhosts. Il est recommandé de placer dans ce cas-là les fichiers de configuration dans le dossier /etc/proftpd.d. Ils seront automatiquement inclus via la directive

```
Include /etc/proftpd.d/*.conf
```
située dans le fichier de configuration général.


Voici un exemple de configuration :

```
<VirtualHost ftpcnam>
    ServerAdmin admin-ftpcnam@gmail.com
    ServerName ftpcnam
    Group cnam
    Port 22
    <Limit LOGIN>
        Order Allow,Deny 
        AllowGroup cnam 
        Deny From All
    </Limit>
    DefaultRoot /home/cnam/ftp/pub
    MaxLoginAttempts 3
    TimeoutIdle 60
    AllowOverwrite on 
</VirtualHost>
```

Une fois que le ftp est configuré, la commande pour s’y connecter est la suivante :

```
ftp [IP]
```

Il est également possible de se connecter avec un logiciel tel que Filezilla en renseignant hôte, utilisateur, mot de passe et port, pour un transfert de fichiers plus aisé.

Nous pouvons mettre notre site en ligne. Mais pas trop vite. En bon administrateur nous devons mettre en place un système de gestion des [fichiers journaux](fichiers-journaux.md), ce qui vaut bien une fiche à part.
