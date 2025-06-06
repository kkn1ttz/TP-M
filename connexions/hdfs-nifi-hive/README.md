# Instructions Connexion HDFS vers HIVE via Nifi

## Prérequis

Avoir complété :

* [installs/Apache Nifi](https://github.com/kkn1ttz/TP-M/tree/master/installs/Apache%20Nifi)
* [installs/Hadoop HDFS](https://github.com/kkn1ttz/TP-M/tree/master/installs/Hadoop)
* [connexions/mysql-nifi-hive](https://github.com/kkn1ttz/TP-M/tree/master/connexions/mysql-nifi-hive)

## Utilisation template

* Si vous souhaitez importer directement le template du flow nifi :
  * Click droit sur le `canvas` dans l'interface nifi
  * Choisir `Upload Template`
  * Choisir le fichier `hdfs-nifi-hive.xml`

* Les instructions pour reproduire le flow Nifi sont les suivantes.

## Mise en place initiale

* Lancer HDFS

```
start-dfs.sh
```

## Mise en place des données dans HDFS 
* Mettre le fichier Accident_road_Characteristics.csv qui est dans `datasets/hdfs` , dans un dossier sur vagrant , par exemple `TpBigData/hdfs`
* Créer un dossier source dans HDFS et ajouter le fichier CSV Accident_road_Characteristics.csv qui est dans `TpBigData/hdfs`

```bash
hdfs dfs -mkdir -p /data/transport/roads/

hdfs dfs -put /vagrant/TpBigData/hdfs/Accident_Road_Characteristics.csv /data/transport/roads/

```

* Vérifier les données

```bash
hdfs dfs -cat /data/transport/roads/Accident_Road_Characteristics.csv | head -n 5
```

## Connexion avec Apache Nifi

* Lancer `Apache Nifi` dans la VM

```bash
/opt/nifi/bin/nifi.sh start
```

> `restart` si déjà lancé

* Naviguer vers [http://localhost:8080/nifi](http://localhost:8080/nifi) sur votre navigateur

* Ajouter les processeurs suivants :

### 1. Ajouter un processeur `ListHDFS`

* Paramétrer :

  * `Directory` = /data/transport/roads
  * `Recurse Subdirectories` = true
  * `Minimum File Age` = 0 sec

### 2. Ajouter un processeur `FetchHDFS`

* Aucun paramétrage spécial nécessaire, il récupère automatiquement les fichiers listés.

### 3. Ajouter un processeur `PutHDFS`

* Paramétrer :

  * `Haddop Configuration Ressource`= /opt/nifi/conf/core-site.xml,/opt/nifi/conf/hdfs-site.xml
  * `Directory` = /user/hive/warehouse/accident_road_characteristics
  * `Conflict Resolution Strategy` = replace
  * `Compression Codec` = NONE

* Relier les processeurs les 3 proccesseurs sur Success: `ListHDFS` -> `FetchHDFS` -> `PutHDFS` (le dernier processeur PutHDFS doit etre relier a lui meme)

* Configurer tous les processeurs pour qu'ils soient `valid`

  * Dans `Relationships`, choisir `terminate` pour `failure`

* Click droit sur le canvas puis `Start`
> Vous pouvez `disable` les autres processeurs du canvas pour eviter qu'ils se lancent avec ce flow.

## Retour dans la VM - Vérification

* Lister si le dossier `/user/hive/warehouse/accident_road_characteristics` a été créé par Nifi

```
hdfs dfs -ls /user/hive/warehouse/ext_people_hdfs
```
* Si le fichier a ete creer mais qu'il y a un `.`devant le fichier creer , vous pouvez le renomer en executant la commmande : 

```
hdfs dfs -mv /user/hive/warehouse/accident_road_characteristics/.Accident_Road_Characteristics.csv /user/hive/warehouse/accident_road_characteristics/Accident_Road_Characteristics.csv
```
> Hive ne prendra pas en compte les fichiers ayant `.`devant son nom donc les donnees ne seront pas inserer si le nom du fichier n'est pas renommer

* Lire le contenu du fichier

```
hdfs dfs -cat /user/hive/warehouse/accident_road_characteristics/Accident_Road_Characteristics.csv
```
## Warning
#### Pour ne pas surcharger le flow Nifi
- Configurer `Schedulling` du processor `ListHDFS` ou `FetchHDFS`
  - `Run schedule` = 59 min
  - C'est l'intervalle entre laquelle le processor s'execute

## Importation des donnees dans Hive

* Configurer le flow nifi utiliser precedement en fonction des configurations.

* Démarrer Hive
```
nohup hive --service metastore > /dev/null &
nohup hiveserver2 > /dev/null &
```

* Démarrer Beeline
```
beeline
```

* Se connecter 
```
!connect jdbc:hive2://localhost:10000

user : oracle
password : welcome1
``` 

* Créer la table **externe** Hive et inserer les donnees

```sql
CREATE EXTERNAL TABLE Accident_Road_Characteristics (
    OBJECTID INT,
    TWAY_ID STRING,
    TWAY_ID2 STRING,
    ROUTE INT,
    ROUTENAME STRING,
    FUNC_SYS INT,
    FUNC_SYSNAME STRING,
    RD_OWNER INT,
    RD_OWNERNAME STRING,
    NHS INT,
    NHSNAME STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/user/hive/warehouse/accident_road_characteristics/'
TBLPROPERTIES ('skip.header.line.count'='1');
```

* Pour verifier que la tables creer est bien une table externe a Hive
```
DESCRIBE FORMATTED accident_road_characteristics;
```
> Dans la section `Table Type`, si c'est une table interne , ca doit etre `MANAGED_TABLE`et si c'est une table externe , ca doit etre `EXTERNAL_TABLE`


* Vérifier que les données sont bien inserer

```sql
SELECT * FROM accident_road_characteristics;

ou 

SELECT count(*) FROM accident_road_characteristics;
```

* La tables est maintenant accessible via python
