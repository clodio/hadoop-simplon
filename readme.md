# Hadoop

## prérequis

bien penser à lancer start-hadoop.sh  au démarrage

## En binôme définir une arborescence pour le stockage dans du fichier (Temps 1 : durée 15min)

* localities/YYYY/MM
* localities/YYYY/MM/headers/headers.txt
* localities/YYYY/MM/csv/localities.csv
* localities/YYYY/MM/parquet/localities.parquet

```bash
hdfs dfs -mkdir localities
hdfs dfs -mkdir localities/2022
hdfs dfs -mkdir localities/2022/10
hdfs dfs -mkdir localities/2022/10/headers
hdfs dfs -mkdir localities/2022/10/csv
hdfs dfs -mkdir localities/2022/10/parquet
```

## Restitution des différentes arborescences (Temps 2 : durée 20min)

```bash
hdfs dfs -ls -R localities 
```

drwxr-xr-x   - simplon supergroup          0 2022-10-11 09:44 localities/2022
drwxr-xr-x   - simplon supergroup          0 2022-10-11 10:07 localities/2022/10
drwxr-xr-x   - simplon supergroup          0 2022-10-11 10:17 localities/2022/10/csv
drwxr-xr-x   - simplon supergroup          0 2022-10-11 10:18 localities/2022/10/headers

## Temps 3 : Travail Individuel (durée 40min)

Lancer la VM
Ouvrir un terminal
Lancer le système de stockage distribué à l’aide du script startup-hadoop.sh

### Partie 1 HDFS

* Récupérer le fichier depuis HDFS sur votre poste local
* Extraire du fichier la 1er ligne contenant l’en-tête dans un fichier nommé cities_headers.txt
* En local créer un fichiers cities_2022.csv (laposte_hexasmal_content.csv) ne contenant pas la ligne d’en-tête
* Sur HDFS créer l’arborescence cible qui contiendra votre fichier
* Envoyer ce fichier sur HDFS

```bash
hdfs dfs -mkdir /localities
hdfs dfs -mkdir /localities/2022
hdfs dfs -mkdir /localities/2022/10
hdfs dfs -mkdir /localities/2022/10/headers
hdfs dfs -mkdir /localities/2022/10/csv

rm -f laposte_hexasmal.csv
rm -f laposte_hexasmal_headers.csv
rm -f laposte_hexasmal_content.csv
hdfs dfs -get laposte_hexasmal.csv 
head -n1 laposte_hexasmal.csv > laposte_hexasmal_headers.csv
hdfs dfs -put laposte_hexasmal_headers.csv /localities/2022/10/headers/laposte_hexasmal.csv
tail -n +2 laposte_hexasmal.csv > laposte_hexasmal_content.csv
hdfs dfs -put laposte_hexasmal_content.csv /localities/2022/10/csv/laposte_hexasmal.csv
rm -f laposte_hexasmal.csv
rm -f laposte_hexasmal_headers.csv
rm -f laposte_hexasmal_content.csv

```

### HDFS Optionnel

* sur la machine local créer un groupe data_analyst
* sur la machine local créer un utilisateur data_analyst et le ratacher au groupe data_analyst
* créer un répertoire data_analyst dans le dossier /user de HDFS
* sur HDFS changer le propriétaire et groupe du dossier /user/data_analyst pour data_analyst
* déplacer le fichier cities_2022.csv (laposte_hexasmal_content.csv) et faire en sorte que seule les data_analyst puissent y accéder en lecture uniquement

```bash
sudo groupadd data_analysts
sudo useradd data_analyst
sudo usermod -a -G data_analysts data_analyst
sudo mkdir /user
sudo mkdir /user/data_analyst
sudo chown -R data_analyst:data_analysts /user/data_analyst
sudo cp laposte_hexasmal_content.csv /user/data_analyst/laposte_hexasmal_content.csv
sudo chown -R data_analyst:data_analysts /user/data_analyst/laposte_hexasmal_content.csv
```

## Temps 4 Partie 2 Hive (durée 1H30)

### dans Hive créer une table cities_2022 qui pointe sur votre fichier cities_2022.csv

/!\ dans le cas de création de tables internes et pas EXTERNAL, les données de hdfs sont supprimées

```hive
CREATE SCHEMA localities;
DROP TABLE localities.cities_2022;
CREATE EXTERNAL TABLE IF NOT EXISTS  localities.cities_2022 ( code_commune_insee String, nom_de_la_commune String, code_postal String, ligne_5 String, libelle_d_acheminement String, coordonnees_gps String) ROW FORMAT DELIMITED FIELDS TERMINATED BY ';' LINES TERMINATED BY '\n';

LOAD DATA INPATH '/localities/2022/10/csv/laposte_hexasmal.csv' OVERWRITE INTO TABLE localities.cities_2022;
```

### dans Hive créer une table cities_2022_parquet qui stockera les données de cities_2022 mais au format parquet

```hive
CREATE TABLE localities.cities_2022_parquet(code_commune_insee string, nom_de_la_commune string, code_postal string, ligne_5 string, libelle_d_acheminement string, coordonnees_gps string) STORED AS Parquet;
```

### Utiliser Hive pour peupler la table cities_2022_parquet à partir de la table cities_2022

```hive
INSERT INTO TABLE localities.cities_2022_parquet SELECT * from localities.cities_2022;
```

### Hive Optionnel

* dans Hive définir une table cities partitionné par année qui contiendra les données du fichiers cities_2022 au format parquet
* ajouter la partition pour l’année 2022
* peupler la table à l’aide de la table cities_2022

```hive
CREATE TABLE localities.cities_2022_partitionned_by_year_parquet(code_commune_insee string, nom_de_la_commune string, code_postal string, ligne_5 string, libelle_d_acheminement string, coordonnees_gps string) PARTITIONED BY (year int) STORED AS Parquet;
/* pas forcement nécessaire : ALTER TABLE localities.cities_2022_partitionned_by_year_parquet Add Partition (year=2022); */
INSERT INTO TABLE localities.cities_2022_partitionned_by_year_parquet PARTITION(year=2022) SELECT * from localities.cities_2022;
```

## Tables internal vs external

| Internal | External |
|---|---|
|  les tables SONT mises dans le dossier warehouse | les tables NE SONT PAS mises dans le dossier warehouse |
| un drop supprime les tables et les metadatas | le drop supprime seulement les metadatas |
| TRUNCATE possible | pas de TRUNCATE |
| ACID | pas ACID |
| Query Cache | pas de query Cache |
