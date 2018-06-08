# Configuration multi-utilisateurs

Le plus simple pour permettre aux différents utilisateurs d’un système UNIX de publier leurs pages personnelles est d’utiliser la directive UserDir.

On peut voir une liste des modules chargés par Apache avec la commande :

```
sudo apachectl -M
```
Pour installer le module mod_userdir s’il n’apparaît pas dans la liste :
```
dnf install apache-mod_userdir
```
```
sudo apachectl -M
...
userdir_module (shared)
...
```

Cette installation est susceptible de créer un fichier /etc/httpd/conf/conf.d/userdir.conf.

```  
﻿UserDir public_html
 
<Directory "/home/*/public_html">
    AllowOverride FileInfo AuthConfig Limit Indexes
    Options MultiViews Indexes SymLinksIfOwnerMatch IncludesNoExec
    Require method GET POST OPTIONS
</Directory>

<Directory "/home/*/public_html/cgi-bin">
    Options ExecCGI
    SetHandler cgi-script
</Directory>
```
Après redémarrage d’Apache, chaque utilisateur du système disposant d’un dossier public_html dans son répertoire /home peut publier du contenu accessible via http://localhost/~user.


