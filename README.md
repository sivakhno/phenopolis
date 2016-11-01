# Phenopolis: an open platform for harmonization and analysis of sequencing and phenotype data

Preprint on [biorxiv](http://biorxiv.org/content/early/2016/10/31/084582).

You can access a demo version of the server at:
https://phenopolis.github.io
username/password:
demo/demo123

### Installation

Phenopolis requires:
* a running mongo database
* a running Phenotips server (needed to provide the HPO phenotypes per patient)
* (optionally) a running Exomiser stand-alone server, which can be obtained on request as it being developed separately by [Julius Jacobsen](https://github.com/julesjacobsen).

You will then be able to run the python Flask server.

The first step is to clone the repository.

```
git clone git@github.com:pontikos/phenopolis.git
```
Download Phenotips:
```
https://phenotips.org/Download
```
Install latest version of mongo:
```
https://www.mongodb.com/download-center#community
```
If you wish to download the Exomiser stand-alone server, please get in touch with [Julius Jacobsen](https://github.com/julesjacobsen).

### Creating database, importing data

First make sure mongoDB is running:
```
DBPATH=
mongod --dbpath $DBPATH --port 27017 --smallfiles
```

#### Creating and importing data from JSON

The variants found in the VCF files are processed with [Variant Effect Predictor (VEP)](http://www.ensembl.org/info/docs/tools/vep/) and the output is written to JSON standard output.
The standard output is piped into another python script, ```postprocess_VEP_json.py``` (available from another [repository](https://github.com/UCLGeneticsInstitute/DNASeq_pipeline)), which adds further annotation, formatting and writes output to JSON, which is then imported with mongoimport into the variants collection:

```
git clone https://github.com/UCLGeneticsInstitute/DNASeq_pipeline
python DNASeq_pipeline/annotation/postprocess_VEP_json.py | grep '^JSON:' | sed 's/^JSON://' > VEP_chr${chr}.json
mongoimport --db $DBNAME --collection variants --host $HOST < VEP_chr${chr}.json
```

The bash command to run the VEP, assuming your variant files are ```chr${chr}.vcf.gz```:
```
ensembl=/cluster/project8/vyp/AdamLevine/software/ensembl/
VEP_DIR=/cluster/project8/vyp/Software/ensembl-tools-release-82/scripts/
DIR_CACHE=/SAN/vyplab/NCMD_raw/VEP/cache/
DIR_PLUGINS=${DIR_CACHE}/Plugins
for chr in `seq 1 22` X
do
############### VEP_mongo chr${chr}
# split single lines
zcat chr${chr}.vcf.gz | python DNASeq_pipeline/annotation/annotation/multiallele_to_single_gvcf.py --headers CHROM,POS,ID,REF,ALT,QUAL,FILTER,INFO > chr${chr}_for_VEP.vcf
####CONFIGURE SOFTWARE SHORTCUTS AND PATHS
reference=1kg
ensembl=/cluster/project8/vyp/AdamLevine/software/ensembl/
export PERL5LIB=${PERL5LIB}:${ensembl}/src/bioperl-1.6.1::${ensembl}/src/ensembl/modules:${ensembl}/src/ensembl-compara/modules:${ensembl}/src/ensembl-variation/modules:${ensembl}/src/ensembl-funcgen/modules:${ensembl}/Plugins
# RUN VEP
/share/apps/perl/bin/perl ${VEP_DIR}/variant_effect_predictor/variant_effect_predictor.pl --offline \
--ASSEMBLY GRCh37 --fasta /SAN/vyplab/UKIRDC/reference/human_g1k_v37.fasta \
--cache --dir_cache ${DIR_CACHE} \
--dir_plugins ${DIR_PLUGINS} \
--sift b --polyphen b --symbol --canonical --check_existing --check_alleles  \
--fork 4 --maf_esp --gmaf --maf_1kg --maf_exac \
--no_progress --quiet \
--custom /cluster/project9/IBDAJE/VEP_custom_annotations/1kg/1kg/chr${chr}_EUR.vcf.gz,1KG_EUR,vcf,exact \
--custom /cluster/project9/IBDAJE/VEP_custom_annotations/1kg/1kg/chr${chr}_AFR.vcf.gz,1KG_AFR,vcf,exact \
--custom /cluster/project9/IBDAJE/VEP_custom_annotations/1kg/1kg/chr${chr}_AMR.vcf.gz,1KG_AMR,vcf,exact \
--custom /cluster/project9/IBDAJE/VEP_custom_annotations/1kg/1kg/chr${chr}_ASN.vcf.gz,1KG_ASN,vcf,exact \
--custom /cluster/project9/IBDAJE/VEP_custom_annotations/1kg/esp/chr${chr}_EA.vcf.gz,ESP_EA,vcf,exact \
--custom /cluster/project9/IBDAJE/VEP_custom_annotations/1kg/esp/chr${chr}_AA.vcf.gz,ESP_AA,vcf,exact \
--custom /cluster/project9/IBDAJE/VEP_custom_annotations/1kg/CADD/chr${chr}.vcf.gz,CADD,vcf,exact \
--custom /cluster/scratch3/vyp-scratch2/reference_datasets/Kaviar/Kaviar-160204-Public/hg19/VEP_annotation.vcf.gz,Kaviar,vcf,exact \
--plugin Condel,${DIR_PLUGINS}/config/Condel/config,b \
--plugin Carol \
--plugin CADD,/cluster/project9/IBDAJE/VEP_custom_annotations/1kg/CADD/chr${chr}.vcf.gz \
--no_stats \
--hgvs \
--pubmed \
--plugin HGVSshift \
--plugin SameCodon \
--input_file ${output}_VEP/chr${chr}_for_VEP.vcf \
--json \
--output_file STDOUT | python  DNASeq_pipeline/annotation/postprocess_VEP_json.py | grep '^JSON:' | sed 's/^JSON://' > VEP_chr${chr}.json
bgzip -f -c chr${chr}_for_VEP.vcf > chr${chr}_for_VEP.vcf.gz
tabix -f -p vcf chr${chr}_for_VEP.vcf.gz
rm chr${chr}_for_VEP.vcf
done
```
Load individual for individual page (this is currently tedious, we are going to streamline this):
```
 python views/load_individual.py --individual $ID --auth Admin:$PASSWORD
```

#### Running pubmedScore

The pubmedscore, written by [Jing Yu](https://github.com/logust79), scores genes based on their pubmed relevance.

The scripts can be found in `./pubmedScore`

Before running the script, it is preferable to write patients ids in `patients.txt`, which `pubmedScore.py` takes by default.

```
python pubmedScore.py
    -i patients.txt
    -g ABCA4 (if specified, will ignore -i and -p)
    -p patientID_1 (if specified, will ignore -i)
    -k retina,retinal,retinitis,blindness,macula,macular,stargardt,pigmentosa (Keywords to search on pubmed. Displayed is default)
```

#### Running phenogenon

The phenogenon, written by [Jing Yu](https://github.com/logust79), does an enrichment test (Fisher test) per gene and HPO term.

The scripts can be found in `./phenogenon`

First, the user has to run `python snapshot_patient_hpo.py` to take a snapshot of patients' HPO at the time. Since the phenogenon analysis will take some time, this is to avoid any inconsistency that might be introduced by editing patients' HPO in the database when phenogenon is running.

Second, `python get_hpo_freq.py` will produce an HPO frequency file that phenogenon will use for its analysis.

Phenogenon can then be run as `python gene_hpo_analysis --chrom X` per chromosome. This feature can be utilised to parallelise the jobs on chromosomes. It uses `ExAC_freq` and `CADD_phred` scores to help filter the variants. The defaults are `ExAC_freq <= 0.01 and CADD_phred >= 15` for _recessive_ inheritance mode, and `ExAC_freq <= 0.001 and CADD_phred >= 15` for _dominant_ inheritance mode. It will produce a JSON file for each gene.

If one wishes to change the cutoffs to filter the variants after phenogenon is done, one can use `python recalculate_p.py --chrom X` to do the job quickly, without having to re-extracting info using the slow `gene_hpo_analysis.py`

After this, `python hpo_gene_anlaysis.py` will extract all genes with significant p values for each valid HPO term, and write to a JSON file for each HPO term.

## Running servers

Run Phenotips:
```
wget https://nexus.phenotips.org/nexus/content/repositories/releases/org/phenotips/phenotips-standalone/1.3-milestone-2/phenotips-standalone-1.3-milestone-2.zip
unzip phenotips-standalone-1.3-milestone-2.zip
cd phenotips-standalone-1.3-milestone-2
bash start.sh
```

Run Exomiser standalone:
```
EXOMISER_DATA=
cd $EXOMISER_DATA
wget ftp://ftp.sanger.ac.uk/pub/resources/software/exomiser/downloads/exomiser/exomiser-cli-7.2.1-data.zip
unzip exomiser-cli-7.2.1-data.zip
java -jar exomiser-rest-prioritiser-7.3.0-SNAPSHOT.jar --exomiser.data-directory=$EXOMISER_DATA
```

Run Phenopolis:
```
cd phenopolis
python run_server.py
```


### Acknowledgment

This code was originally forked from the [ExAC browser](https://github.com/konradjk/exac_browser) but has since diverged considerably.


