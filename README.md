## Présentation
Ce projet développé lors de mon année d'étude à l'Université du Québec à Chicoutimi (UQAC) vise à configurer et déployer un SIEM (Security Information and Event Management) opérationnel à la volée. SimpleSIEM est livré avec cinq signatures préenregistrées afin de détecter les scénarios d'attaque suivant:
- Reconnaissance de l'infrastructure
- Enumération et fuzzing de services web
- Tentatives de bruteforce d'identifiant web online
- Téléchargement de fichiers malveillant

Pour consulter et reproduire le déroulement des scénarios d'attaque, vous pouvez consulter le rapport de projet. La pile applicative sous-jacente est composée de:
## Installation
Veuillez vous référer au guide d'installation de Docker pour pouvoir mettre en place l'environnement d'exécution
et déployer le projet: https://docs.docker.com/engine/install/

Il vous faudra ensuite télécharger le repos GitHub soit au format zip soit avec l'outil `git`
- `git clone https://github.com/Nirij3m/SIEM_Stack.git`

## Lancement
Dans le répertoire local du projet cloné, exécuter la commande suivante pour lancer les conteneurs.
- `docker compose build`
- `docker compose up`

Les images des conteneurs seront automatiquement tirées puis configurées.

Pour stopper les conteneurs (sans les supprimer):
- `docker compose stop`

Pour supprimer les conteneurs:
- `docker compose down`

### Configuration
Une fois les conteneurs lancés, il vous faudra configurer Kibana pour communiquer avec la base ElasticSearch.
Pour cela, il faut se connecter à l'interface Kibana exposée du conteneur sur la machine hôte:
- `http://localhost:5601`

Kibana demandera un token de connexion pour réaliser la liaison avec ElasticSearch. Il faut le générer à partir d'un invite de commande:
- `docker exec -it elasticsearch /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana` 
```shell
~/SIEM_Stack$ docker exec -it elasticsearch /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
...SNAP...
eyJ2ZXIiOiI4LjE0LjAiLCJhZHIiOlsiMTcyLjE4LjAuMjo5MjAwIl0sImZnciI6ImYyNjNjNDY4OTdhN2E2M2NiYTA4ZGJiNGM0MzU2MzliZjYxNmFkYTEyMmQ5YTJhZTEwN2JmMDM4NzVkZTBjZGMiLCJrZXkiOiJUOE1INFprQmhiVVhyN2NYWVppdTpZUHh2dnJBQkpIVER2SGYtU012dmdBIn0=
```
Kibana demandera ensuite un code de confirmation à récupérer:
- `docker exec -it kibana /usr/share/kibana/bin/kibana-verification-code`
```shell
~/SIEM_Stack$ docker exec -it kibana /usr/share/kibana/bin/kibana-verification-code
Your verification code is:  710 006
```

Une fois la configuration terminée, il faudra modifier les identifiants du compte superadministrateur `elastic`:
- `docker exec -it elasticsearch /usr/share/elasticsearch/bin/elasticsearch-reset-password -i -u elastic`
```shell
~/SIEM_Stack$ docker exec -it elasticsearch /usr/share/elasticsearch/bin/elasticsearch-reset-password -i -u elastic
This tool will reset the password of the [elastic] user.
You will be prompted to enter the password.
Please confirm that you would like to continue [y/N]y

Enter password for [elastic]: elastic
Re-enter password for [elastic]: elastic
Password for the [elastic] user successfully reset.
```

Choisissez comme mot de passe `elastic`. Ces identifiants sont utilisés par défaut par le conteneur syslogng pour initier une connexion authentifiée vers ElasticSearch et enregistrer les logs 

Si vous souhaitez utiliser des identifiants personnalisés, il vous faudra modifier le fichier de configuration de syslogng: `./configs/syslog-ng.conf` du répertoire local `SIEM_Stack`:
```shell
            destination d_elasticsearch_https {
                elasticsearch-http(
                    url("https://elasticsearch:9200/_bulk")
                    index("logs")
                    type("")
            		user("elastic") <------------------ Remplacer par le nom utilisateur choisi  ----
            		password("elastic") <------------------ Remplacer par le mot de passe choisi ----
            		#template("$(format-json --scope rfc5424 --scope dot-nv-pairs
                    #--rekey .* --shift 1 --scope nv-pairs
                    #--exclude DATE @timestamp=${ISODATE})")
                    #tls(
                    #    ca-file("/config/certs/http_ca.crt")
                    #    peer-verify(yes)
                    #)
            		tls(peer-verify(no))
                );
            };
```
Puis relancez le conteneur syslogng pour appliquer les changements:
- `docker compose restart syslogng`

**Attention**, par défaut l'IDPS Suricata écoute le trafic sur l'interface par défaut _eth0_ du conteneur Debian. Si jamais votre instance de Docker Engine attribue un autre nom d'interface réseau pour ce conteneur, l'IDPS ne pourra pas inspecter les paquets et donc fonctionner correctement. Pour remédier à cela, il vous faudra identifier le nom d'interface attribué au conteneur Debian puis de modifier le fichier Dockerfile situé à l'emplacement `/configs/dockerfile_debian.txt` ligne 25 avec: 
- `...SNAP... /bin/suricata -c /etc/suricata/suricata.yaml -vvv -i NOUVELLE_INTERFACE ...SNAP...`

## Utilisation
Vous pouvez désormais vous connecter à ElasticSearch et configurer les index et visualisations depuis l'interface Kibana à l'adresse:  `http://localhost:5601`. 
Les index pour stocker les logs devraient être automatiquement créées, si ce n'est pas le cas vous pouvez en créer un manuellement dans l'onglet `Index Management` de ElasticSearch. L'index devra s'appeler `logs` et un autre `alerts`. Si vous souhaitez un nom d'index personnalisé vous devrez éditer le champ de la configuration ci-dessus: `index("VOTRE_NOM_INDEX")` et redémarrer le conteneur syslogng.

Par défaut, le conteneur `syslogng` collecte les logs du serveur web apache2 `80:80` et du serveur ssh `2222:22` du conteneur hôte `debian`. Vous pourrez interagir avec ces services et générér des logs à ces adresses:
- `http://localhost:80` pour le serveur apache2
- `ssh root@localhost -p 2222` pour se connecter en ssh à l'hôte `debian`. Le mot de passe du compte `root` est `root` par défaut.

Pour plus de détails sur la pile applicative et les configurations appliquées, veuillez-vous référer au rapport de projet.




