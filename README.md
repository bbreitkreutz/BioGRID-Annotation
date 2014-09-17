BioGRID-Annotation
==================

This resource contains tools for creating and maintaining an annotation database that is specifically used for operation with the the BioGRID suite of web applications and desktop software (http://thebiogrid.org). It builds off major protein databases such as REFSEQ and UNIPROT as well as other resources for Gene annotation such as ENTREZ GENE and other model organism databases. 

## Requirements
To use all of the tools contained within, you require at least the following:

+ Python 2.7+
+ MySQL 5.5+
+ wget
+ Approximately 300 GB of HD Space (shared between database and downloaded files)
+ At least 8 GB of Memory (more is much better)

### Required Python Libraries
+ MySQLdb
+ gzip
+ json
+ xml.sax
+ xml.etree
+ glob

### Required Databases (from SQL directory if starting fresh)
+ core_staging - Tables used for staging data before updating.
+ core_stats - Tables for storing statistics on update progress.
+ core - Tables storing the complete annotation dataset.

## Update Process

### Before Getting Started 
Make sure you have a loaded copy of the annotation database tables to use for the new annotation. Load a MySQL dump of the existing CORE tables (in the SQL folder), and load it into a fresh database if starting new. Otherwise, point to a previous version of the database. Once the database is loaded, make sure you add any organisms you want added, or changed, to the organisms table, as that table will be used to fetch your set of annotation.

+ Run the batch/fetchDownloads.sh file to download all the required files for an annotation update. This process will take some time, depending on the speed of your network connection. For example: the Trembl file from Uniprot is currently larger than 55 GB in XML format.

+ Verify that all files are downloaded

+ Go to config/config.json (or create this file modelled on the config.json.example file already in this directory) and adjust the settings in here to point to your setup. Especially modify the paths and the database login credentials to match your current configuration.

#### Prepare STAGING DATABASE

+ Run: **python EG_parseGeneHistoryToStaging.py** - This will load the gene history from ENTREZ GENE into a staging table for later use.

+ Run: **python UNIPROT_parseSwissProtAccessionsToStaging.py** - This will load all the SWISSPROT accession ids into a staging table so we can later quickly determine which ids are from SWISSPROT and which are from TREMBL.

+ Run: **python REFSEQ_fetchProteinIDs.py** - This will load all the protein UIDs for REFSEQ into a staging table based on the organisms we are interested in.

#### Process GENE ONTOLOGY

+ Run: **python GO_parseDefinitions.py** - This will load all the terms and definitions from GO and create a mapping to their GO SLIM subsets.

+ Run: **python GO_parseRelationships.py** - This will load all the is_a relationships between terms from GO.

+ Run: **python GO_buildSubsetPairings.py** - This will build parent child relationship pairings between GO terms and their parent terms based on GO SLIM subsets.

#### Process REFSEQ

+ Run: **python REFSEQ_downloadProteins.py** - This will download protein FASTA files for all the protein IDs downloaded into the staging database. This will take some time as sequences can only be fetched in batches of 10,000 at a time.

+ Run: **python REFSEQ_parseProteins.py** - This will parse the FASTA files downloaded in the step above and load them into the database. If the sequence already exists, its details are updated instead.

#### Process ENTREZ GENE

+ Run: **python EG_updateGeneHistory.py** - This will use _entrez_gene_history_ in the staging database to swap identifiers if they were replaced with an alternative. Also, it will discontinue genes that were merged, so there are no redundancies.

+ Run: **python EG_parseGenes.py** - This will load up all new genes from ENTREZ GENE into the genes table using only the organisms in the _organisms_ table.

+ Run: **python EG_parseAliases.py** - This will load all the aliases from ENTREZ GENE fetching only those we are interested in via previously loaded data stored in the _genes_ table.

+ Run: **python EG_parseExternals.py** - This will load all the external database references from ENTREZ GENE fetching only those we are interested in via previously loaded data stored in the _genes_ table.

+ Run: **python EG_parseDefinitions.py** - This will load all the definition entries from ENTREZ GENE fetching only those we are interested in via previously loaded data stored in the _genes_ table.

+ Run: **python EG_parseGO.py** - This will load the ENTREZ GENE gene2go file for a mapping of GENE ONTOLOGY terms to _genes_.

+ Run: **python EG_parseGene2Refseq.py** - This will parse the Gene2Refseq file from ENTREZ GENE and create a mapping table between _genes_ and _refseq_.

#### Process REFSEQ MISSING

+ Run: **python REFSEQ_downloadMissingProteins.py** - This will download protein FASTA files for any proteins in the _refseq_ table that do not have annotation. These are most likely proteins that were added when running the EG_parseGene2Refseq.py script.

+ Run: **python REFSEQ_parseMissingProteins.py** - This will parse the FASTA files downloaded in the step above and load them into the database.

#### Process MODEL ORGANISM SUPPLEMENTS

+ Run: **python SGD_parseGenes.py** - This will process the SGD Features file and match our _gene_ table with the current official symbol and descriptions for genes. It will also load in "retrotransposon" records which are required by BioGRID but not loaded via ENTREZ_GENE.

+ Run: **python POMBASE_parseGenes.py** - This will process the Pombase file and match our _gene_ table with the current official symbol and descriptions for genes. 

+ Run: **python CGD_fixAnnotation.py** - This will process the CGD file and fix some problems with the ENTREZ GENE version. Specifically, some duplicate entries and non-common place systematic names.

#### Process UNIPROTKB