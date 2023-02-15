# InfluxDB 1.8-vers-2.6

## Le contexte:

Je possède une base InfluxDB 1.8  (appelée **V1** pour la suite) sur un container LCX Proxmox qui enregistre mes valeurs long termes depuis debut 2021.

Je souhaite passer en version 2.6.1 actuelle à date (appelée **V2** pour la suite) mais un container Docker

Pas de souci particulier pour la création du container docker pour V2 avec ce compose
```yaml
version: '3.3'
services:
    influxdb2:
        container_name: influxdb2
        ports:
            - '8086:8086'
        volumes: 
          - /home/docker/influxdb2:/var/lib/influxdb2
        image: 'influxdb:2.6.1'
        environment:
          TZ: Europe/Paris
  ```
  
## Structure de V1:
V1 contient une seule base de donnés qui se nomme ``edf``

## Démarrage de notre InfluxDB2 
Depuis le navigateur : `` http://192.168.1.10:8686 ``

Création du Bucket :

Load Data -> Buckets -> Create Bucket

Création du Token à conserver car il sera utilisé plus tard:

Load Data -> Api Tokens -> Generate Api Tokens -> All acces Api Token



## Export des données de la base V1
Voici la commande à lancer sur la machine de V1 avec une connexion ssh:

`` influx_inspect export -database "edf" -datadir "/var/lib/influxdb/data" -waldir "/var/lib/influxdb/wal" -out "/home/export" ``

Cette commande envoie la totalité de vos enregistrements dans un fichier qui sera nommé ``export`` et positionné dans le répertoire ``/home``

SI votre base est conséquente, il est possible de scinder vos exports en plusieurs fichiers par mois ou année

J'ai fait par année avec ces 3 lignes: 
```
influx_inspect export -database "edf" -datadir "/var/lib/influxdb/data" -waldir "/var/lib/influxdb/wal" -out "/home/export/2021" -start "2021-01-01T00:00:00Z" -end "2022-01-01T00:00:00Z"
influx_inspect export -database "edf" -datadir "/var/lib/influxdb/data" -waldir "/var/lib/influxdb/wal" -out "/home/export/2022" -start "2022-01-01T00:00:00Z" -end "2023-01-01T00:00:00Z"
influx_inspect export -database "edf" -datadir "/var/lib/influxdb/data" -waldir "/var/lib/influxdb/wal" -out "/home/export/2023" -start "2023-01-01T00:00:00Z" -end "2024-01-01T00:00:00Z"
```
Je me retrouve donc avec 3 fichiers `` 2021 ; 2022 et 2023 `` dans mon répertoire `` /home/export `` de ma machine V1

Ttransfert ces 3 fichiers sur la machine docker où est le container V2

Commande faite avec _scp fichier_source UserSshV2@IP-LocalDeV2:fichier_destination_ depuis la machine V1

``` scp /home/export/202* user1@192.168.1.10:/home/import/ ```

Il faut mettre ce ou ces fichiers dans le container docker de V2 (le container se nomme Influxdb2)

Voici la ligne type: _docker cp fichier ID_container:/fichier_ depuis la machine V2
```  
docker cp /home/import/2021 Influxdb2:/home/import
docker cp /home/import/2022 Influxdb2:/home/import
docker cp /home/import/2023 Influxdb2:/home/import

```
Nos 3 fichier se trouve dans le container dans ``/home/import``

## Importation des données dans V2

Nous passons sur le container de V2 avec cette commande depuis l'hote de V2:

`` docker exec -it influxdb2 bash``

Pour l'importation des données, on lance ces 3 lignes
```
influx write -o my_org -b my_new_bucket -t my_token -f /home/import/2021
influx write -o my_org -b my_new_bucket -t my_token -f /home/import/2022
influx write -o my_org -b my_new_bucket -t my_token -f /home/import/2023
```
Voilà, Il ne vous reste plus qu'à mettre à jour système pour alimenter votre nouvelle base InfluxDB2

