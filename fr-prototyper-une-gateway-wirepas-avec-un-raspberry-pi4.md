# Prototyper une gateway Wirepas avec un Raspberry Pi4

Tout au long de ce tutoriel, nous allons voir comment prototyper une gateway Wirepas avec un Raspberry Pi4.

## Matériel

Pour le matériel, nous allons avoir besoin de :

* 1 Raspberry Pi4 4GB+ avec Raspbian Lite installé : [https://www.raspberrypi.org/products/raspberry-pi-4-model-b/#find-reseller](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/#find-reseller)
* 1 dongle USB u-blox NINA-B1 avec Wirepas 5 : [https://store.clemanis.com/fr/iot/1238-dongle-usb-u-blox-nina-b1-avec-wirepas-40.html#/710-option_protection_usb-oui/727-wirepas-wirepas_50](https://store.clemanis.com/fr/iot/1238-dongle-usb-u-blox-nina-b1-avec-wirepas-40.html#/710-option_protection_usb-oui/727-wirepas-wirepas_50)
* 3 tags Ruuvi avec Wirepas 5 : [https://store.clemanis.com/fr/iot/1202-ruuvi-wirepas.html#/727-wirepas-wirepas_50](https://store.clemanis.com/fr/iot/1202-ruuvi-wirepas.html#/727-wirepas-wirepas_50)

## Logiciels

Pour la partie logicielle, nous allons utiliser :

* le sinkService de Wirepas pour collecter les messages envoyés par les capteurs
* le TransportService de Wirepas pour envoyer les messages à un broker MQTT
* Mosquitto faisant office de broker MQTT
* Node-RED pour gérer les flux d'extraction et de transformation.

## Préparation du Raspberry Pi4

Après avoir récupéré la dernière version de Raspbian Lite ([https://www.raspberrypi.org/downloads/raspberry-pi-os/](https://www.raspberrypi.org/downloads/raspberry-pi-os/)), il faut flasher l'image, contenue dans le zip, sur une carte micro-SD. Pour cela, vous pouvez utiliser balenaEtcher qui a l'avantage d'être multi-plateforme : [https://www.balena.io/etcher/](https://www.balena.io/etcher/).

Avant de débrancher votre carte micro-SD, pensez à créer un fichier vide nommé `ssh` sur la partition `boot`. Cela vous permettra d'accéder à votre RPi4 au moyen de SSH sans avoir besoin de brancher d'écran, ni de clavier (mode headless).

Après un premier démarrage, il est préférable de mettre à jour le système en lançant les commandes suivantes :
    
    sudo apt-get update
    sudo apt-get upgrade

## Installation de la gateway Wirepas (sinkService et TransportService)
A partir de maintenant, vous pouvez brancher le dongle USB.

Pour commencer, il faut installer quelques outils Python :

    sudo apt-get install python3 python3-dev python3-gi libsystemd-dev

Puis installez `pip` avec cette commande :

    wget https://bootstrap.pypa.io/get-pip.py && sudo python3 get-pip.py && rm get-pip.py && sudo pip3 install --upgrade pip
    
Enfin, il nous reste à créer un dossier `wirepas` dans le dossier utilisateur `/home/pi`, dans lequel nous allons nous placer pour le reste du tutoriel :

    mkdir /home/pi/wirepas
    cd /home/pi/wirepas

### Installation du sinkService
Pour procéder à l'installation du sinkService, nous allons utiliser les binaires de la dernière version disponible à cette adresse : [https://github.com/wirepas/gateway/releases](https://github.com/wirepas/gateway/releases)

Le package à récupérer s'appelle `sinkService-arm.tar.gz`. Vous pouvez utiliser les commandes suivantes à mettant à jour l'URL, si besoin :

    wget https://github.com/wirepas/gateway/releases/download/v1.3.0/sinkService-arm.tar.gz
    tar -xzvf sinkService-arm.tar.gz

L'exécutable `sinkService` est maintenant disponible dans le dossier à partir duquel vous venez de récupérer le package et fait l'extraction.

Ensuite, il faut configurer le sinkService. Pour cela, nous allons récupérer un fichier de configuration standard que nous allons mettre à jour pour qu'il s'exécute avec l'utilisateur par défaut de Raspbian, à savoir `pi` :

    wget https://raw.githubusercontent.com/wirepas/gateway/master/sink_service/com.wirepas.sink.conf
    sed -i 's/user="wirepas"/user="pi"/g' com.wirepas.sink.conf
    sudo mv com.wirepas.sink.conf /etc/dbus-1/system.d/

### Lancement du sinkService

Si vous n'avez pas encore branché le dongle USB, commencez par éteindre le RPi4, branchez le dongle USB puis rallumez le RPi4.

A partir de maintenant, il est possible de lancer le sinkService avec cette commande, en vous plaçant dans le dossier où se trouve l'exécutable (`/home/pi` normalement) :

    /home/pi/wirepas/sinkService -p /dev/ttyUSB0 -b115200 -i 0

Si tous ce passe bien, vous devriez obtenir un résultat similaire à celui-ci :

    2020-09-10 17:09:53,243 | [INFO] Main:Bitrate set to 115200
    2020-09-10 17:09:53,244 | [INFO] Main:Starting Sink service:
    	-Port is /dev/ttyUSB0
    	-Bitrate is 115200
    	-Dbus Service name is com.wirepas.sink.sink0
    2020-09-10 17:09:53,249 | [DEBUG] SERIAL:Custom bitrate set: 115200
    2020-09-10 17:09:53,250 | [INFO] wpc_int:WPC initialized
    2020-09-10 17:09:53,254 | [INFO] msap:Status is 0x00
    2020-09-10 17:09:53,400 | [INFO] Main:Node is running mesh API version 13

### Création du service sinkService

Afin que le sinkService puisse démarrer en même temps que le système, nous allons créer un service pour `systemd`.

Il suffit de créer le fichier `wirepasSink.service` dans le dossier `/etc/systemd/system/` et de mettre le contenu suivant :

    [Unit]
    Description=Wirepas sink manager for sink connected to /dev/ttyUSB0
    Requires=getty.target

    [Service]
    Type=simple
    User=pi
    ExecStart=/home/pi/wirepas/sinkService -b 115200 -p /dev/ttyUSB0 -i 0
    Restart=always
    RestartSec=5

    [Install]
    WantedBy=multi-user.target

ATTENTION, la création et l'édition doivent être faits en mode `superuser` au moyen de `sudo`.

Maintenant, il est possible d'activer le service au moyen de :

    sudo systemctl enable wirepasSink.service
    
Pour démarrer le service, il suffit de lancer :

    sudo systemctl start wirepasSink.service
    
Pour vérifier que le service fonctionne correctement, vous pouvez utiliser la commande :

    systemctl status wirepasSink.service
    

### Installation de la gateway Wirepas (TransportService)

Pour l'installation de la gateway Wirepas (TransportService), il faut récupérer et installer le fichier wheel correspondant à la version de votre `sinkService` :

    sudo pip3 install wirepas_gateway

Cela devrait mettre un peu de temps à s'installer et par moment l'installation peut sembler figée. Tout dépend de votre connexion internet et des performances de votre carte microSD.

L'exécutable `wm-gw` devrait se trouver dans `/usr/local/bin`.

### Configuration de TransportService

Pour configurer le TransportService, il faut créer un fichier `settings.yml` dans le dossier `wirepas` dnas le dossier utilisateur (`/home/pi/wirepas`) par exemple, avec le contenu suivant :

    # MQTT brocker Settings
    #
    mqtt_hostname: localhost
    mqtt_port: 1883
    mqtt_username: wirepas
    mqtt_password: wirepas123
    mqtt_force_unsecure: true 
    # Gateway settings
    #
    gateway_id: 1
    gateway_model: 1
    gateway_version: 1 
    # Filtering Destination Endpoints
    #
    ignored_endpoints_filter: [0]

Vous pouvez modifer le `mqtt_username` et le `mqtt_password` à votre convenance. Ils seront utilisés lorsque nous configurerons le broker MQTT Mosquitto.

Pour exécuter `mw-gw`, il suffit de lui fournir le fichier de configuration :

    wm-gw --settings /home/pi/wirepas/settings.yml

Vous devriez obtenir un résultat similaire à celui-ci :

    ***********************************************************************
    * WARNING:
    *     You are using the pure python protobuf implementation.
    *     For better results, please consider using the cpp version.
    *     For more information please consult:
    *     https://github.com/protocolbuffers/protobuf/tree/master/python
    ***********************************************************************

    2020-09-11 09:42:56,553 | [INFO] wirepas_gateway@transport_service.py:187:Version is: 1.3.0
    2020-09-11 09:42:56,573 | [INFO] wirepas_gateway@sink_manager.py:481:New sink added with name sink0
    2020-09-11 09:42:56,573 | [INFO] wirepas_gateway@dbus_client.py:74:Starting dbus client with c extension
    Traceback (most recent call last):
      File "/usr/local/bin/wm-gw", line 8, in <module>
    sys.exit(main())
      File "/usr/local/lib/python3.7/dist-packages/wirepas_gateway/transport_service.py", line 832, in main
    TransportService(settings=settings, logger=logger).run()
      File "/usr/local/lib/python3.7/dist-packages/wirepas_gateway/transport_service.py", line 213, in __init__
    last_will_message,
      File "/usr/local/lib/python3.7/dist-packages/wirepas_gateway/protocol/mqtt_wrapper.py", line 77, in __init__
    keepalive=MQTTWrapper.KEEP_ALIVE_S,
      File "/usr/local/lib/python3.7/dist-packages/paho/mqtt/client.py", line 839, in connect
    return self.reconnect()
      File "/usr/local/lib/python3.7/dist-packages/paho/mqtt/client.py", line 962, in reconnect
    sock = socket.create_connection((self._host, self._port), source_address=(self._bind_address, 0))
      File "/usr/lib/python3.7/socket.py", line 727, in create_connection
    raise err
      File "/usr/lib/python3.7/socket.py", line 716, in create_connection
    sock.connect(sa)
    ConnectionRefusedError: [Errno 111] Connection refused

L'erreur obtenue est normale : nous avons pas encore installé de broker MQTT.

###### Note sur la sécurité
Ici, nous utilisons simplement un utilisateur et un mot de passe, mais il est possible de définir aussi un certificat SSL, afin de se connecter de manière plus sécurisée. Dans notre cas, le broker MQTT est sur le même système, il n'apparait pas nécessaire d'utiliser cette fonctionnalité. Dans le cas où le broker MQTT serait distant, cela deviendrait une nécessité de passer par une connexion SSL/TLS : par défaut `wm-gw` utilise du TLS 1.2 lorsque l'on active cette option.

### Création du service TransportService
Afin que le TransportService puisse démarrer en même temps que le système, nous allons créer un service pour `systemd`.

Il suffit de créer le fichier `wirepasTransport.service` dans le dossier `/etc/systemd/system/` et de mettre le contenu suivant :

    [Unit]
    Description=Wirepas Transport Process
    Requires=network.target

    [Service]
    Type=simple
    User=pi
    ExecStart=/usr/local/bin/wm-gw --settings=/home/pi/wirepas/settings.yml
    Restart=always
    RestartSec=5

    [Install]
    WantedBy=multi-user.target

ATTENTION, la création et l'édition doivent être faits en mode `superuser` au moyen de `sudo`.

Maintenant, il est possible d'activer le service au moyen de :

    sudo systemctl enable wirepasTransport.service
    
Pour démarrer le service, il suffit de lancer :

    sudo systemctl start wirepasTransport.service
    
Pour vérifier que le service fonctionne correctement, vous pouvez utiliser la commande :

    systemctl status wirepasTransport.service

Le `status` du service devrait renvoyer une erreur tant que le broker MQTT n'est pas installé.

## Installation du broker MQTT Mosquitto

Pour installer Mosquitto, il suffit d'utilise cette commande :

    sudo apt-get install mosquitto
    
Mosquitto s'installe en tant que service `systemd`, vous pouvez voir son status :

    systemctl status mosquitto.service

Maintenant, il reste à configurer Mosquitto, pour qu'il utilise le nom d'utilisateur et le mot de passe que l'on a configuré dans le fichier `settings.yml`.

Pour cela, nous allons créer l'utilisateur :

    sudo mosquitto_passwd -c /etc/mosquitto/passwd wirepas

Vous pouvez changer `wirepas`par le nom d'utilisateur que vous avez mis dans `settings.yml`. Il vous est demandé saisir et confirmer le mot de passe.

Enfin, il faut désactiver l'accès anonyme et indiquer le fichier de mot de passe dans la configuration de Mosquitto. Il faut éditer, en mode `superuser`(`sudo`) le fichier, `/etc/mosquitto/mosquitto.conf` ajouter les lignes suivantes :

    allow_anonymous false
    password_file /etc/mosquitto/passwd
    
Pour terminer, il faut redémarrer le service pour que la configuration soit prise en compte :

    sudo systemctl restart mosquitto.service

## Vérification de l'installation

Avant d'aller plus loin, il est nécessaire de vérifier que les tags Ruuvi nous envoie bien des données et qu'elles arrivent jusqu'au broker MQTT.

Pour cela, il faut allumer les tags Ruuvi :

* en ouvrant le capot supérieur
* en retirant la languette de plastique entre l'électrode et la pile : une led rouge et une led verte devraient s'allumer brièvement.

Vous pouvez refermer le tag Ruuvi en plaçant l'ouverture circulaire du capot supérieur dans l'alignement de l'électrode de la pile. C'est à ce niveau que se situe le capteur d'humidité : juste derrière l'électrode sous la forme d'un petit composant carré noir. C'est important de bien replacer le capot supérieur pour obtenir des mesures d'humidité plus précises.

Pour vérifier que des messages sont bien envoyés par les capteurs, vous pouvez la commande suivante :

    wm-dbus-print
    
Vous devriez obtenir un résultat similaire à celui-ci :

    ***********************************************************************
    * WARNING:
    *     You are using the pure python protobuf implementation.
    *     For better results, please consider using the cpp version.
    *     For more information please consult:
    *     https://github.com/protocolbuffers/protobuf/tree/master/python
    ***********************************************************************

    2020-09-11 10:30:16,839 | [INFO] wirepas_gateway.dbus_print_client@dbus_print_client.py:40:[2020-09-11 09:30:16] Sink sink0 FROM XXXXXXXX TO YYYYYYYYYY on EP 112 Data Size is 2
    2020-09-11 10:30:16,842 | [INFO] wirepas_gateway.dbus_print_client@dbus_print_client.py:40:[2020-09-11 09:30:16] Sink sink0 FROM XXXXXXXX TO YYYYYYYYYY on EP 114 Data Size is 2
    2020-09-11 10:30:16,847 | [INFO] wirepas_gateway.dbus_print_client@dbus_print_client.py:40:[2020-09-11 09:30:16] Sink sink0 FROM XXXXXXXX TO YYYYYYYYYY on EP 116 Data Size is 4

`XXXXXXXX` correspond à l'identifiant du capteur qui devrait se trouver sur une étiquette en dessous du capteur.

Pour vérifier que les messages arrivent bien jusqu'au broker MQTT, vous pouvez utiliser un explorateur MQTT comme MQTT explorer qui est multiplateforme : [http://mqtt-explorer.com/](http://mqtt-explorer.com/). Il suffit de vous connecter avec l'adresse IP du RPi4 ainsi que l'utilisateur et le mot de passe que avez défini dans Mosquitto. 

Vous devriez voir un noeud `gw-event`dans l'arborescence sur la gauche de la fenêtre. En déroulant de noeud, vous devriez tomber sur l'arborescence suivante `received_data > 1 > sink0 > 51966 > 111` suivi de plusieurs noeuds.

`sink0` correspond à l'identifiant du sinkService que l'on a configuré : voir l'option `-i 0`.

`51966` correspond au `network id`.

`111` correspond au `source endpoint`. Il s'agit ici de l'identifiant du fabricant.

Les noeuds suivants correspondent au `destination endpoint` et permettent de savoir de quel type de message il s'agit. Quelques exemples :

* `112` : température
* `114` : humidité
* `116` : pression atmosphérique
* `127` : détection de choc - message étendu

Vous pouvez retrouver la liste complète des `destination endpoint` sur cette page : 

Si vous souhaitez obtenir plus d'informations sur les topics publiés par la gateway Wirepas, vous pouvez consulter cette page (en anglais) : [API between a Gateway and Wirepas Backends](https://github.com/wirepas/backend-apis/tree/master/gateway_to_backend)

## Installation de Node-RED

Pour commencer, il faut ajouter un nouveau depôt pour `NodeJS` au moyen de cette commande :

    curl -sL https://deb.nodesource.com/setup_12.x | sudo bash -

Vous pouvez mettre à jour le numéro de version avec celui que correspondant à la dernière version de NodeJS en LTS : ici il s'agit de la version 12.

Puis, il faut s'assurer que certains prérequis sont bien présents :

    sudo apt install build-essential git

Pour installer, Node-RED, il suffit d'utiliser cette commande et de répondre `y` (oui) aux questions :

    bash <(curl -sL https://raw.githubusercontent.com/node-red/linux-installers/master/deb/update-nodejs-and-nodered)
    
D'après la [documentation officielle de Node-RED](https://nodered.org/docs/getting-started/raspberrypi), cette commande permet :

 * supprimer les versions packagées de Node-RED et NodeJS si elles sont déjà présentes.
 * installer la dernière version LTS de NodeJS au moyen des dépôts NodeSource. Si NodeJs est déjà installé à partir de NodeSource, elle va s'assurer que la version est au moins la 8 et ne la désinstallera pas
 * installe la dernière version de Node-RED en utilisant npm.
 * installe, éventuellement, une collection de noeud spécifiques pour les Raspberry Pi.
 * Configure Node-RED pour s'exécuter en tant que service `systemd`

Une fois l'installation terminée, il faut penser à l'activer et à le démarrer le service `nodered` :

    sudo systemctl enable nodered
    sudo systemctl start nodered

Maintenant, vous devriez pouvoir accéder à votre Node-RED à cette adresse :

    http://{your_pi_ip-address}:1880

## Préparation de Node-RED

Afin de pouvoir extraire les données des messages envoyés par les capteurs, il est nécessaire d'ajouter plusieurs extensions à Node-RED. Il est possible de le faire via l'insterface Web (menu en haut à droite > Manage palette) mais il est préférable de le faire en ligne de commande : cela s'est avéré plus efficace.

    sudo npm i -g node-red-contrib-binary node-red-contrib-protobuf node-red-node-base64 node-red-node-msgpack node-red-node-random node-red-node-suncalc
    sudo systemctl restart nodered

Maintenant, il faut récupérer les `prototypes` de messages afin que l'on puisse extraire les données des messages. Pour cela, il suffit de cloner le dépôt [backend-apis](https://github.com/wirepas/backend-apis) de wirepas, toujours depuis le dossier `/home/pi/wirepas` :

    git clone https://github.com/wirepas/backend-apis.git

## Utilisation de Node-RED

Pour simplifier cette partie, vous pouvez importer (menu en haut à droite > import) le fichier [generic-message-flow.json](generic-message-flow.json).

Il ne vous reste plus qu'à modifier le noeud `LocalMosquitto` en double-cliquant dessus, puis en éditant le serveur en cliquant sur l'icône crayon. Dans cette nouvelle fenêtre d'édition, il faut sélectionner l'onglet `Security`, puis il faut indiquer l'utilisateur et le mot de passe que vous avez paramétrer précédemment dans Mosquitto. Enfin, il faut cliquer sur `Update`, puis `Done`.

Il ne reste plus qu'à cliquer sur `Deploy` pour appliquer les changements sur le flow et le lancer.

Vous pouvez voir les données arriver en ouvrant la console de debug : l'icône `bug` en haut à droite. Dans l'objet affiché, la propriété `wirepas` correspond aux données brutes extraites par le noeud `GenericMassage to JSON` et la propriété `data` correspond aux données extraites par les différents `Extract ...`.

Son fonctionnement est simple : il extrait les principales données envoyées par les tags Ruuvi.

Vous pouvez modifier le flow à votre convenance pour modifier les données ou les envoyer vers des applciations tierces telles que Grafana par exemple.

## FAQ

## Dans la console Node-RED, il y a des erreurs concernant un fichier `proto`

Pour commencer, vous pouvez vérifier que le chemin d'accès du fichier `proto` utilisé par le noeud `GenericMessage to JSON` correspond au chemin d'accès dans lequel se trouve les fichiers `proto` du dépôt GitHub `backend-apis` de Wirepas : `backend-apis/gateway_to_backend/protocol_buffers_files/`

Après avoir redéployé le flow, vous obtenez toujours une erreur, veillez à mettre les droits suivants sur le dossier contenant les fichiers `proto` :

    chmod -R 755 /home/pi/wirepas/backend-apis/gateway_to_backend/protocol_buffers_files

