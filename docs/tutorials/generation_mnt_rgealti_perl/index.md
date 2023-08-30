# Génération d'une pyramide MNT RGEALTI avec les outils perl de la suite rok4
## Construction de l'image docker

L'image docker sera construite d'après le fichier [Dockerfile](./Dockerfile) joint.
Un accès internet par docker est requis lors du build. (Attention donc en cas d'utilisation d'un VPN ou d'un proxy.)

Dans le dossier qui servira de contexte de build, créer un dossier `dpkg/`. Y placer les fichiers debians récupérés sur les pages suivantes.
L'OS cible est `ubuntu 20.04 amd64`, et j'ai testé uniquement sans les librairies ceph.
Entre parenthèses, le lien direct vers le fichier debian non compressé. (Les téléchargements d'archives tar.gz sont parfois possibles.)

* https://github.com/rok4/core-cpp/releases/latest
* https://github.com/rok4/core-perl/releases/latest
* https://github.com/rok4/generation/releases/latest
* https://github.com/rok4/pregeneration/releases/latest
* https://github.com/rok4/styles/releases/latest
* https://github.com/rok4/tilematrixsets/releases/latest

Depuis le dossier de contexte :

```bash
docker build -t local/rok4_perl_gen .
```

Dans l'image résultante, les fichiers du projet rok4 sont répartis ainsi :

* exécutables : `/usr/bin/`
* libs C++ : `/usr/include/rok4`
* libs dynamiques `.so` : `/usr/lib/x86_64-linux-gnu`
* libs Perl : `/usr/lib/perl5/ROK4`
* styles : `/usr/share/rok4/styles`
* TMS : `/usr/share/rok4/tilematrixsets`


## Téléchargement des données de base
### Téléchargement et extraction des données brutes sur les sites de l'IGN

La page de téléchargement des données RGEALTI est : https://geoservices.ign.fr/rgealti
Une archive téléchargée inclut à la foix les données MNT, DST, et SRC.

Les données brutes se trouvent dans des sous-dossier du dossier `RGEALTI/1_DONNEES_LIVRAISON_<suffixe>`. Exemple pour la donnée RGEALTI 1m du Calvados du 03/01/2023 :

```
 $ ll RGEALTI_2-0_1M_ASC_LAMB93-IGN69_D014_2023-01-03/RGEALTI/1_DONNEES_LIVRAISON_2023-01-00244/
total 2168
drwx------ 5 user user   4096 juil. 20 14:59 ./
drwx------ 5 user user   4096 juil. 20 14:58 ../
drwx------ 2 user user 471040 juil. 20 14:59 RGEALTI_DST_1M_ASC_LAMB93_IGN69_D014_20230125/
-rwx------ 1 user user 735048 janv. 26 02:50 RGEALTI_DST_1M_ASC_LAMB93_IGN69_D014_20230125.md5*
drwx------ 2 user user 258048 juil. 20 15:04 RGEALTI_MNT_1M_ASC_LAMB93_IGN69_D014_20230125/
-rwx------ 1 user user 735048 janv. 26 02:48 RGEALTI_MNT_1M_ASC_LAMB93_IGN69_D014_20230125.md5*
drwx------ 2 user user   4096 janv. 26 10:46 RGEALTI_SRC_1M_ASC_LAMB93_IGN69_D014_20230125/
```

Décompresser l'archive dans un dossier choisi.
Pour le type de donnée voulue, identifier le dossier concerné, par exemple `RGEALTI_MNT_1M_ASC_LAMB93_IGN69_D014_20230125` pour le type `MNT`.


### Conversion de ces données en un format exploitable par be4

`be4` ne prend pas en charge le format d'image `ASC`, il faut donc les convertir dans un format pris en charge, par exemple du `GeoTIFF`. Le dossier source de la conversion, `${dossier_images_asc}` sera le dossier identifié juste avant.
Créer un dossier de destination `${dossier_images_tiff}` dédié, puis :

```bash
for image in $(ls ${dossier_images_asc}); do gdal_translate -ot "Float32" -of "GTiff" "${dossier_images_asc}/${image}" "${dossier_images_tiff}/${image/.asc/.tiff}"; done
```

Dans le cas de `RGEALTI_2-0_1M_ASC_LAMB93-IGN69_D014_2023-01-03/RGEALTI/1_DONNEES_LIVRAISON_2023-01-00244/RGEALTI_MNT_1M_ASC_LAMB93_IGN69_D014_20230125`, il y a 5976 fichiers à traiter, donc ça prend du temps.


## Exécution

Les volumes nommés sont là pour indiquer les dossiers utilisés par la configuration fournie, et à mutualiser en cas de parallélisation de la génération sur plusieurs conteneurs. Ne pas hésiter à utiliser des volumes liés en cas de volonté de récupération des fichiers, ou aucun volume en cas d'éxécution sur un seul conteneur sans récupération des données.
Le volume lié référence l'emplacement des données source sur le système hôte.

Le fichier de configuration de be4, [conf.json](./conf.json), est fourni avec ce document.

NB: La propriété `"process"/"style"` ne doit pas être définie pour une sortie à un seul canal comme. A l'heure actuelle, les styles disponibles ne concernent en effet que des sorties à 3 ou 4 canaux.

La propriété `"pyramid"/"name"` est le nom de la pyramide en sortie, à modifier si besoin pour l'unicité des noms.
La propriété `"process"/"parallelization"` définit le nombre de scripts shells de traitement d'images à exécuter pour chaque niveau de la pyramide. 
Un script unique "main.sh" gère l'exécution des différents scripts de traitement d'images.

Pour créer les scripts shell :

```bash
docker run -v rok4gen:/tmp/rok4 -v rok4data:/data/rok4 -v ${dossier_images_tiff}:/data/RGEALTI -v ${fichier_de_configuration}:/tmp/config/conf.json --rm local/rok4_perl_gen be4.pl --conf=/tmp/config/conf.json
```

Puis, pour les exécuter :

```bash
docker run -v rok4gen:/tmp/rok4 -v rok4data:/data/rok4 -v ${dossier_images_tiff}:/data/RGEALTI -v ${fichier_de_configuration}:/tmp/config/conf.json --rm local/rok4_perl_gen bash /tmp/rok4/scripts/main.sh
```

Dans l'exemple de configuration fourni, la parallélisation n'est pas active, et la pyramide en sortie comporte 9 niveaux, ce script gèrera donc l'exécution de 10 sous-scripts : 1 par niveau, puis le script de finalisation de la pyramide.
De manière générale, le nombre de scripts (autres que main.sh) à exécuter est : `process.parallelization * nombre_niveaux + 1`.
Il est possible de ne pas utiliser `main.sh`, et d'exécuter séparément chacun de ces scripts. Dans ce cas, il faut veiller à rendre le dossier des sources accessible en lecture, et les dossiers de travail et de destination accessible en lecture et écriture, et cela pour tous les scripts exécutés, même s'ils sont exécutés en parallèle.Le script de finalisation doit être exécuté quand tout les autres scripts cités sont terminés avec succès.

La pyramide obtenue est ainsi prête à être utilisée par une couche de webservice du serveur de diffusion rok4, une autre génération, ou toute autre application s'appuyant sur le format rok4.

