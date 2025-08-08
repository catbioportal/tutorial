## Customizing cBioPortal for non-human species
### Introduction

This tutorial will give you an overview of how to take the [cBioportal](https://cbioportal.org) codebase and extend it to support custom, non-human, species. This guide resulted from the effort to create [catBioPortal](https://catbioportal.org) for feline cancer genomics. Broadly, this will require four primary phases: preparing your data, modifying the codebase, building/publishing artifacts and containers, and loading your data into the instance. We will cover each step in detail below.

### Preparing your data
You will need to acquire the following files in order to seed the cBioportal database for your new reference.
 - [reference_genome_file](https://github.com/cBioPortal/cbioportal/blob/master/docs/Import-reference-genome.md) A TSV file describing the reference you intend to import, including the species, build name, URL, and release date.
 
 - gene_info file: This can be obtained from the NCBI FTP server at   ftp.ncbi.nih.gov. If there is not one specific to your species available, you can filter down the `All_Mammalia.gene_info` file by using the taxon id of your species like so: `cat All_Mammalia.gene_info | grep "^9685" > felis_catus.gene_info`
 
- gtf file: This can be obtained via the UCSC downloads server. Find your species [here](https://hgdownload.soe.ucsc.edu/downloads.html) and then look under for the `refGene.gtf` file. (The exact directory layout can vary between species)

- IGV reference track: If your species already has a hosted reference track for IGV [listed here](https://igv.org/doc/igvjs/#Reference-Genome/) you don't need to acquire any additional files for this. If it is not you will need `.fa`, `fa.fai` , and `bands.tsv` files for your species. The `fa` files can be acquired from the [UCSC downloads server](https://hgdownload.soe.ucsc.edu/downloads.html) as well. You may need to generate a bands file yourself. It is a TSV file with the following headers:

```
#chrom chromStart chromEnd name gieStain
```

- Chromosome sizes JSON: You will need a JSON object in the format below that maps each chromosome name to its length in bases.
```
"yourSpecies": { "1": 1234, "2": 3456 }
```

### Modifying the codebase
There are four repositories you will need to fork in order to get set up. They are:

 - [https://github.com/cBioPortal/cbioportal-core](https://github.com/cBioPortal/cbioportal-core) This repo contains the validation and loading scripts for importing references, studies, etc.
 - [https://github.com/cBioPortal/cbioportal](https://github.com/cBioPortal/cbioportal) This is the main web application backend and the primary repository that you will use to build the final Docker images.
 - [https://github.com/cBioPortal/cbioportal-frontend](https://github.com/cBioPortal/cbioportal-frontend) This is the react.js frontend of the web application
 - [https://github.com/cBioPortal/cbioportal-docker-compose](https://github.com/cBioPortal/cbioportal-docker-compose) This contains docker-compose configuration that we will use to boot the application.

#### Modifying the frontend
In `env/master.sh`  you will want to find the line that says`export CBIOPORTAL_URL=`. To run the frontend against your local cbioportal you will want to update it to say `http://localhost:8080`. For your release build you will want to update it to the url/domain of your production site.

In `src/shared/lib/IGVUtils.ts` you will want to add a helper method that points to the IGV reference track files you prepared in the first step. It can either point to the officially hosted ones if they already existed or it can point to the ones you acquired yourself. It should look something like this but with your species specific information:

```
export function defaultfelCat9ReferenceProps() {
   return {
     id: 'felCat9',
     name: "Feline (felCat9/Felis_Catus_9.0)",
     fastaURL: 'https://catbioportal.org/reactapp/igv/felCat9.fa',
     indexURL: 'https://catbioportal.org/reactapp/igv/felCat9.fa.fai',
    cytobandURL: 'https://catbioportal.org/reactapp/igv/felCat9_bands.tsv',
  }
}
```

In `src/shared/lib/referenceGenomeUtils.tsx`, you will need to update several functions and dictionaries to be aware of your new reference. Adding support for cat  looks something like this:

```
src/shared/lib/referenceGenomeUtils.tsx --- 1/4 --- TypeScript TSX
16 16         NCBI: 'GRCm38',
17 17         UCSC: 'mm10',
18 18     },
.. 19     felcat9: {
.. 20         NCBI: 'Felis_catus_9.0',
.. 21         UCSC: 'felCat9',
.. 22     },
19 23 };
20 24 
21 25 export const GENOME_ID_TO_GENOME_BUILD = {

src/shared/lib/referenceGenomeUtils.tsx --- 2/4 --- TypeScript TSX
27 31     hg38: 'GRCh38',
28 32     GRCm38: 'GRCm38',
29 33     mm10: 'GRCm38',
.. 34     felCat9: 'felCat9',
30 35 };
31 36 
32 37 export function isMixedReferenceGenome(studies: CancerStudy[]): boolean {

src/shared/lib/referenceGenomeUtils.tsx --- 3/4 --- TypeScript TSX
39 44     const isAllStudiesGRCm38 = _.every(studies, study => {
40 45         return isGrcm38(study.referenceGenome);
41 46     });
.. 47     const isAllStudiesFelCat9 = _.every(studies, study => {
.. 48         return isFelcat9(study.referenceGenome);
.. 49     });
42 50     // return true if there are mixed studies
43 51     return !(
.. 52         isAllStudiesGRCh37 ||
.. 53         isAllStudiesGRCh38 ||
.. 54         isAllStudiesGRCm38 ||
.. 55         isAllStudiesFelCat9
.. 56     );
44 57 }
45 58 

src/shared/lib/referenceGenomeUtils.tsx --- 4/4 --- TypeScript TSX
88 101     );
89 102 }
.. 103 
.. 104 export function isFelcat9(genomeBuild: string) {
.. 105     return (
.. 106         REFERENCE_GENOME.felcat9.NCBI.match(new RegExp(genomeBuild, 'i')) ||
.. 107         REFERENCE_GENOME.felcat9.UCSC.match(new RegExp(genomeBuild, 'i'))
.. 108     );
.. 109 }
90 110 
91 111 export function formatStudyReferenceGenome(genomeBuild: string) {
92 112     if (isGrch37(genomeBuild)) {
```
#### Modifying the core
In `scripts/importer/chromosome_sizes.json` you will need to add an entry for the reference build you intend to support. In our case the addtional 
value looked like this:
```
"felCat9": { "A1": 242100913, "C1": 222790142, "B1": 208212889, "A2": 171471747, "C2": 161193150, "B2": 155302638, "B3": 149751809, "B4": 144528695, "A3": 143202405, "X": 130557009, "D1": 117648028, "D3": 96884206, "D4": 96521652, "D2": 90186660, "F2": 85752456, "F1": 71664243, "E2": 64340295, "E1": 63494689, "E3": 44648284 }
```

In `scripts/importer/cbioportal_common.py` you will need to add the name of your reference to the `valid_segment_reference_genomes` array as demonstrated [here](https://github.com/catbioportal/cbioportal-core/blob/main/scripts/importer/cbioportal_common.py#L979).

Likewise, you will need to make the script at `scripts/importer/validateData.py` aware of your new reference. This includes inside [load_chromosome_lengths](https://github.com/catbioportal/cbioportal-core/blob/main/scripts/importer/validateData.py#L1076) and [reference_genome_map](https://github.com/catbioportal/cbioportal-core/blob/main/scripts/importer/validateData.py#L5764). 

This script also contains logic aliasing the X and Y chromosomes to 23 and 24, if this does not apply to your organism, you will need to change that logic as we have [here](https://github.com/catbioportal/cbioportal-core/blob/main/scripts/importer/validateData.py#L3632).

In `src/main/java/org/mskcc/cbio/portal/model/ReferenceGenome.java` you will want to add the appropriate constants for your organism. In our case that looked like this:

```
public static String FELIS_CATUS = "cat";
public static String FELIS_CATUS_DEFAULT_GENOME_NAME = "felCat9";
public static String FELIS_CATUS_DEFAULT_GENOME_BUILD_PREFIX = "Felis_catus_";
```

Depending on what types of data you intend to import, you will likely have to modify the cBioPortal model files representing that data. For example, we have to modify the `ReferenceGenomeId` enum in ` src/main/java/org/mskcc/cbio/portal/model/CopyNumberSegmentFile.java` to include felCat9.

Other examples include removing logic in `src/main/java/org/mskcc/cbio/portal/scripts/ImportExtendedMutationData.java` and `src/main/java/org/mskcc/cbio/portal/scripts/ImportCopyNumberSegmentData.java` that [asserts that all chromosomes will be numeric](https://github.com/catbioportal/cbioportal-core/blob/main/src/main/java/org/mskcc/cbio/portal/scripts/ImportExtendedMutationData.java#L195) and modifying `src/main/java/org/mskcc/cbio/portal/scripts/ImportGeneData.java` to [skip importing genes from taxons other than cat](https://github.com/catbioportal/cbioportal-core/blob/main/src/main/java/org/mskcc/cbio/portal/scripts/ImportGeneData.java#L89). 

#### Modifying the webapp

Fortunately, only minimal modifications to the main API webserver 
are required. Primarily, you need to modify the routes configuration in ` src/main/java/org/cbioportal/WebAppConfig.java` to make it aware of any custom pages you have added to your instance.

### Building artifacts

 1. To build the cbioportal-core JAR file, simply run `mvn package` in the root directory of the repository. This will produce a file named something like `core-1.0.8.jar` in the same directory. Upload this fie somewhere accessible. We recomend attaching it to a GitHub release.
 
 2. You will need to have the cbioportal Docker image include your customized cbioportal-core jar file. To do this modify the `Dockerfile` at `docker/web-and-data/Dockerfile`.  Look for the line below  and change it to point to the customized the jar file you published  in the previous step instead of the official cbioportal repository.
 ```
 RUN wget https://github.com/cBioPortal/cbioportal-core/releases/download/1.0.6/core-1.0.6.jar -P core/ ; cd core ; jar -xf core-1.0.6.jar scripts/ requirements.txt ; chmod -R a+x scripts/ ; cd ..;
```


 3. You will also need to point the cbioportal build process at your newly customized cbioportal-frontend release. You can do this by updating the dependency in the `pom.xml` in the main cbioportal repository. 
 Look for the block below  and replace the `groupId` with your GitHub repository and the `version` with either a [commit SHA or a tagged release](https://docs.cbioportal.org/development/build-different-frontend/).

 ```
<frontend.groupId>com.github.cbioportal</frontend.groupId>
<frontend.version>v6.0.5</frontend.version>
 ```

4. With that complete, you can build and push your customized Docker image. For this, you will need a [DockerHub](https://hub.docker.com) account. In the below commands, substitute your Docker username and your preferred image name. You will only need the `linux/arm64` platform if you are on an Apple Silicon Mac.
```
docker buildx build --platform linux/amd64,linux/arm64 -t my-docker-username/mycustombioportal:latest -f docker/web-and-data/Dockerfile .

docker push  my-docker-username/mycustombioportal:latest
```

### Booting up your instance 

 1. In the cbioportal-docker-compose repository you cloned earlier, update the `DOCKER_IMAGE_CBIOPORTAL` variable in the `.env` file to point to the image you built and pushed in the previous steps.
 
 2. In `data/init.sh` you will need to modify the line that looks like this `docker run --rm -it $VERSION cat /cbioportal/db-scripts/src/main/resources/cgds.sql > cgds.sql` unless you have published your `DOCKER_IMAGE_CBIOPORTAL` image version number to match the corresponding version number from cbioportal. If you have not, you can remove that line and instead add 
 ```
  wget -O "https://github.com/cBioPortal/cbioportal/raw/v6.0.3/src/main/resources/db-scripts/cgds.sql"
  ```
  Where `v6.0.3` is the version of cbioportal that you have forked your repo from.

3. You will want to download the schema and demo data by running `./init.sh` in the docker compose repository root.

4. In the `application.properties` file, find the block below and update it to match the reference you intend to import.
``` 
species=human
ncbi.build=37
ucsc.build=hg19
```

5. You can now boot up your instance using Docker compose
```
docker compose -f docker-compose.yml -f dev/open-ports.yml up

#if you are on an Apple Silicon Mac, add this extra flag:
-f docker-compose.arm64.yml
```
 
 Your instance should boot up in a few minutes and be accessible at `http://localhost:8080`
  
### Loading your reference data
After your instance has booted up, you can begin loading the reference data you prepared earlier. Make sure all your data files are available in a directory that is mounted into your container. By default `/study` is, so that is a good choice.

The initialization scripts will have loaded in some human reference files which we do not want. The first thing we should do is delete them. This can be done by opening up a prompt to the SQL database with the following command:
 
```
 docker compose exec cbioportal-database sh -c 'mysql -hcbioportal-database -u"$MYSQL_USER" -p"$MYSQL_PASSWORD" "$MYSQL_DATABASE"'
```

You will want to run the following SQL commands to clean up the human genes that were previously imported. The applicaiton will not behave correctly if you have multiple genes with the same symbol. This mostly follows the procedure documented [here](https://docs.cbioportal.org/updating-gene-and-gene_alias-tables/#mysql-steps) with some additional cleanup added.

```
DELETE FROM cosmic_mutation;
DELETE FROM mutation;
DELETE FROM mutation_event;
TRUNCATE TABLE gene_alias;
DELETE FROM gene;
DELETE FROM genetic_entity;
DELETE FROM geneset_hierarchy_node;
DELETE FROM gene;
ALTER TABLE `genetic_entity`  AUTO_INCREMENT  =  1;
ALTER TABLE `geneset_hierarchy_node`  AUTO_INCREMENT  =  1;
ALTER TABLE `geneset`  AUTO_INCREMENT  =  1;
``` 

You can now exit the mysql prompt and get a bash prompt in the main docker container like so `docker exec -it cbioportal-container /bin/bash`

First, import your reference genome file. Assuming you have your data files available under `/study` that can be done like this: 

```
mkdir /data
export PORTAL_DATA_HOME=/data
cd /cbioportal/core/src/main/scripts
./importReferenceGenome.pl --ref-genome /study/reference_genome_file
```

Likewise, we can now import our genes:
```
 export PORTAL_DATA_HOME=/data ./importGenes.pl --genes \
 /study/my_species.gene_info --gtf /study/my_species.refGene.gtf \ 
 --genome-build "Build Name From Reference File" \
 --species species_name_from_reference_file
```


### Loading a study
Now that you have an operational instance that is aware of your new reference, you can load study data as described in the [official documentation](https://docs.cbioportal.org/data-loading/).  This primarly consists of preparing your data files in the formats described in the documentation and then running some variation of `./metaImport.py -s /data/study/ -u http://localhost:8080 -v` to load the study data into your instance.

While we described some example modifications to the loading scripts in the cbioportal-core repository to accommodate your new reference, further modifications may be required depending on what types of data you load. You can follow the same pattern as above to publish a new version of cbioportal-core, point cbioportal to it, and rebuild your container.

### Deploying to production
With our modifications in place, we can deploy our customized cBioPortal to an Amazon EC2 instance.

First you will want to set up Docker:

```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install git nginx docker docker-compose-plugin
sudo groupadd docker
sudo usermod -aG docker ${USER}
```

Next, you will want to check out the Docker compose configuration. You can start from the canonical one at `https://github.com/cBioPortal/cbioportal-docker-compose.git` or the one you have already modified locally.

Start up your instance with `docker compose -f docker-compose.yml -f dev/open-ports.yml up -d`. This should boot up all the services you need.

The final step is to expose the running instance to the internet and set up a certificate for HTTPS access. We installed `nginx` in a previous step so we will use that.

Open up `/etc/nginx/sites-available/default` and add the following lines:

```
server {
        listen 80;
        listen [::]:80;

        server_name your-custom-domain.com;
        location / {
          proxy_set_header  X-Forwarded-For $remote_addr;
          proxy_set_header  Host $http_host;
          proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header  X-Forwarded-Proto $scheme;
          proxy_set_header  X-Forwarded-Ssl on; # Optional
          proxy_set_header  X-Forwarded-Port $server_port;
          proxy_set_header  X-Forwarded-Host $host;
          proxy_pass        "http://127.0.0.1:8080";
        }
}

```

Restart `nginx` with `sudo service nginx restart`. It will now proxy requests to your domain to the cBioPortal instance running on the machine. Finally, you can set up an SSL certificate by running

```
sudo snap install --classic certbot
sudo certbot
```

and following the prompts. You should now be able to access your instance at https://your-custom-domain.com.

