---
title: "Relevé de consommation d'éléctricité avec un Sonoff Basic"
date: 2020-11-28
draft: false
---
This first article will exceptionally be written in French as users of Linky electrical counter are mostly French :)  
Let me know if you want the English version.

Les compteurs Linky offre la possibilité de suivre sa consommation éléctrique directement via le site www.edf.fr.  
Cette option n'est pas la meilleure pour tout le monde sachant qu'avoir une consommation détaillée necessite de donner à EDF le droit d'exploiter ses données de consommation à des fins commerciales. Aussi, les anciens compteurs n'ont pas cette possibilité.

Une solution alternative est d'utiliser la téléinformation disponible sur tous les compteurs, Linky ou non, du moment que que les sorties I1 et I2 sont présentes sur les compteurs.  
Cela permet de garder ses données chez soi et d'avoir des mesures de consommation avec une meilleure granularité.  
En effectuant une mesure toutes les minutes par exemple.

Pour pouvoir faire ces mesures, nous allons utiliser un Sonoff Basic qui est un relais Wifi basé sur un ESP8265.

Cette solution a plusieurs avantages:
* Le firmware du Sonoff Basic peut être facilement remplacé par un firmware que nous allons compiler
* Le circuit d'alimentation via du 220v est déjà tout fait
* Les Pins VCC, RX et GND sont facilement accessible. Ce qui nous permettra de récupérer la téléinformation

Pour le firmware nous allons utilisé ESPHome: https://github.com/esphome/esphome qui permet de compiler son propre firmware via un fichier de configuration écrit en yaml.
L'avantage de cette solution est que nous pouvons ajouter dans le firmware que ce qui est nécessaire à la différence de Tasmota, ESPurna ou ESPeasy.

Pour commencer, il faut démonter le Sonoff Basic et souder des fils sur les pins RX, TX, GND et VCC.
Il existe pleins de tutoriels pour le faire donc je ne vais pas m'y attarder.  
Par exemple: https://github.com/xoseperez/espurna/wiki/Hardware-Itead-Sonoff-Basic

Une fois les fils soudés, vous pouvez vous munir d'un convertisseur USB-UART 3v3 et vous pouvez flash esphome (https://esphome.io/index.html).

Les patches pour la téléinformation étant dans la branche de dévelopemment pour le moment, nous devons cloner le repository git.

```bash
git clone https://github.com/esphome/esphome.git
git checkout 5a2b14cfa47a722053b36b5116ddc5d9d9d39597
```

Le commit `5a2b14cfa47a722053b36b5116ddc5d9d9d39597` étant le commit apportant la téléinformation.

Une fois fait, je recommande d'utiliser un virtualenv pour installer esphome comme suit:

```bash
cd esphome
virtualenv .
source bin/activate
```
Nous pouvons maintenant installer les dépendances et esphome dans le virtualenv:

```bash
pip install -r requirements.txt 
pip install .
```

Puis créer notre projet `teleinfo.yaml`.
Nous pouvons utiliser le wizard pour cela:

```bash
$ esphome teleinfo.yaml wizard
<snip>

(name): teleinfo
Great! Your node is now called "teleinfo".

<snip>

Please enter either ESP32 or ESP8266.
(ESP32/ESP8266): ESP8266
Thanks! You've chosen ESP8266 as your platform.

Next, I need to know what board you're using.
Please go to http://docs.platformio.org/en/latest/platforms/espressif8266.html#boards and choose a board.

For example "nodemcuv2".
Options: d1, d1_mini, d1_mini_lite, d1_mini_pro, esp01, esp01_1m, esp07, esp12e, esp210, esp8285, esp_wroom_02, espduino, espectro, espino, espinotee, espresso_lite_v1, espresso_lite_v2, gen4iod, heltec_wifi_kit_8, huzzah, inventone, modwifi, nodemcu, nodemcuv2, oak, phoenix_v1, phoenix_v2, sparkfunBlynk, thing, thingdev, wifi_slot, wifiduino, wifinfo, wio_link, wio_node, xinabox_cw01
(board): esp8285
Way to go! You've chosen esp8285 as your board.
<snip>
```

Voici le yaml généré:

```yaml
esphome:
  name: teleinfo
  platform: ESP8266
  board: esp8285

wifi:
  ssid: "myssid"
  password: "mypass"

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Teleinfo Fallback Hotspot"
    password: "v55JM6GajDlp"

captive_portal:

# Enable logging
logger:

# Enable Home Assistant API
api:

ota:
```

Nous devons le modifier pour ajouter la téleinformation sur la broche RX (GPIO3) mais aussi désactiver le logger sur l'UART car cela ne fonctionne pas en même temps que la téléinformation.
De plus nous ne pouvons pas utiliser une autre broche que le RX relié au bloc hardware UART car l'implémentation Software de l'UART (sur une broche différente de GPIO3) n'est pas assez efficace.

Dans le yaml ci dessous, le Sonoff Basic va envoyer les données de téléinformation à un serveur MQTT et envoyer seulement les étiquettes `HCHP`, `HCHC` et `PAPP` qui correspondent respectivement aux consommations en heures pleines, en heures creuses et à la puissance apparente.
Le logger est désactivé et les logs sont envoyés à un serveur mqtt:

```yaml
esphome:
  name: teleinfo
  platform: ESP8266
  board: esp8285

wifi:
  ssid: "myssid"
  password: "mypass"

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Teleinfo Fallback Hotspot"
    password: "v55JM6GajDlp"

captive_portal:

# Disable logger over UART
logger:
  baud_rate: 0

# Enable Home Assistant API
api:

ota:

mqtt:
  broker: name_of_your_broker
  log_topic: log

uart:
  id: uart_bus
  rx_pin: GPIO3
  tx_pin: GPIO1
  baud_rate: 1200
  parity: EVEN
  data_bits: 7

sensor:
  - platform: teleinfo
    tags:
     - tag_name: "HCHC"
       sensor:
        name: "hchc"
        unit_of_measurement: "Wh"
        icon: mdi:flash
     - tag_name: "HCHP"
       sensor:
        name: "hchp"
        unit_of_measurement: "Wh"
        icon: mdi:flash
     - tag_name: "PAPP"
       sensor:
        name: "papp"
        unit_of_measurement: "VA"
        icon: mdi:flash
    update_interval: 60s
    historical_mode: true
```

Pour un détail de toutes les étiquettes possibles, je vous conseil de regarder la documentation de la téléinformation:
https://www.enedis.fr/sites/default/files/Enedis-NOI-CPT_54E.pdf

Il y aussi l'option `historical_mode` qui permet de préciser si le compteur EDF est en mode historique ou standard. Vous pouvez trouver la configuration de votre Linky directement via le menu sur le linky.

Maintenant que le yaml est prêt, il suffit de le flasher après avoir relié votre convertisseur UART vers USB entre le Sonoff et votre ordinateur.
```bash
esphome teleinfo.yaml run
```

Votre Sonoff maintenant flashé, il faut créer un petit circuit permettant de démoduler le signal de téléinformation et le relié à la pin RX du Sonoff.
Pour cela, nous allons utiliser un optocoupleur SFH620A et une résistance de 1k que nous allons soudé sur une plaque.
Il y a plusieurs montage possible mais c'est le montage que j'utilise depuis quelques mois et je n'ai eu aucun problèmes de stabilité.  
Voici un schéma du montage:  
![alt text](/schema.jpg)

Voici une photo du circuit avec le sfh620a relié au linky et au sonoff:

![alt text](/sfh620a_linky_little.jpg)


Et les fils qui partent du Sonoff via un trou au niveau du capot:  
![alt text](/sonoff_sfh_little.jpg)


Une fois que tout est relié et connecté au compteur, votre Sonoff devraient envoyer les informations au server MQTT.  
J'utilise une raspberry avec mosquitto comme serveur MQTT.
Pour voir les données reçu, nous pouvons utiliser:
```bash
$ mosquitto_sub -v -h localhost -t '#'
log [D][sensor:092]: 'hchc': Sending state 7309004.00000 Wh with 0 decimals of accuracy
teleinfo/sensor/hchc/state 7309004
log [D][sensor:092]: 'hchp': Sending state 10155192.00000 Wh with 0 decimals of accuracy
teleinfo/sensor/hchp/state 10155192
log [D][sensor:092]: 'papp': Sending state 450.00000 VA with 0 decimals of accuracy
teleinfo/sensor/papp/state 450
```

Dans un prochain post, nous allons voir comment traiter les données dans homeassistant et génerer des graphiques de consommation.
