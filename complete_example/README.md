<a name="complete_example"></a>
Complete example of _de novo_ strain level analysis from metagenome data
========================================================================

## Table of Contents  
[Getting started](#getting_started)  

[Read assembly, mapping and binning](#assembly)   

[Contig genome assignments](#genome_assignment)  

[Contig taxonomic assignments](#taxa_assignment)

[Determine core genes](#core_genes)

[Determine variants](#determine_variants)

[Infer strains](#infer_strains)

[Validate strains](#validate_strains)

[Assign accessory genomes](#assign_acessory)


![alt tag](../desmans.jpg)

<a name="getting_started"></a>

Getting started
---------------

To provide an in depth illustration of how to use Deman we will give a complete worked example from a subset of the synthetic community used in the [bioRxiv preprint](http://biorxiv.org/content/early/2016/09/06/073825. We have provided 16 samples, subsampled to 1 million reads from the 64 samples with 11.75 million reads used 
originally. This example is therefore more tractable but the following analysis assumes you have access to a multi-core server. We also assume that you have some standard and not so standard sequence analysis software installed. To make things a bit simpler we include installation for each of these assuming a Linux Ubuntu distribution if you are 
not running Ubuntu these will have to be adapted accordingly:

You will use need to make a local bin directory and a repos dir:
```bash
mkdir ~/bin ~/repos
```
if not already present and add to your path by adding this line to your .bashrc:
```bash
export PATH=~/bin:$PATH
```
and also install GSL:
```bash
sudo apt-get update
sudo apt-get install build-essential libgsl0-dev
```

1. [megahit](https://github.com/voutcn/megahit): A highly efficient metagenomics assembler currently our default for most studies
    
    ```bash
    sudo apt-get install zlib1g-dev
    cd ~/repos
    git clone https://github.com/voutcn/megahit
    cd megahit && make
    cp megahit* ~/bin
    ```

2. [bwa](https://github.com/lh3/bwa): Necessary for mapping reads onto contigs

    ```bash
    cd ~/repos
    git clone https://github.com/lh3/bwa.git
    cd bwa && make
    cp bwa ~/bin
    ```

3. [bam-readcount](https://github.com/genome/bam-readcount): Used to get per sample base frequencies at each position

    ```bash
    cd ~/repos
    sudo apt-get install build-essential git-core cmake zlib1g-dev libncurses-dev patch
    git clone https://github.com/genome/bam-readcount.git
    mkdir bam-readcount-build && cd $_
    cmake ../bam-readcount && make
    cp bin/bam-readcount ~/bin
    ```

4. [samtools](http://www.htslib.org/download/): Utilities for processing mapped files. The version available through apt will *NOT* work instead...

    ```bash
    cd ~/repos
    wget https://github.com/samtools/samtools/releases/download/1.3.1/samtools-1.3.1.tar.bz2
    tar -xjf samtools-1.3.1.tar.bz2 
    cd samtools-1.3.1/ 
    sudo apt-get install libcurl4-openssl-dev libssl-dev
    ./configure --enable-plugins --enable-libcurl --with-plugin-path=$PWD/htslib-1.3.1
    make all plugins-htslib
    cp samtools ~/bin
    ```

5. [bedtools](http://bedtools.readthedocs.io/en/latest/): Utilities for working with read mappings

    ```bash
    sudo apt-get install bedtools
    ```

6. [prodigal](https://github.com/hyattpd/prodigal/releases/): Used for calling genes on contigs

    ```bash
    wget https://github.com/hyattpd/Prodigal/releases/download/v2.6.3/prodigal.linux 
    cp prodigal.linux ~/bin/prodigal
    chmod +x ~/bin/prodigal
    ```

7. [gnu parallel](http://www.gnu.org/software/parallel/): Used for parallelising rps-blast

    ```bash
    sudo apt-get install parallel
    ```

8. [standalone blast](http://www.ncbi.nlm.nih.gov/books/NBK52640/): Need a legacy blast 2.5.0 which we provide as a download:

    ```bash
    wget https://desmandatabases.s3.climb.ac.uk/ncbi-blast-2.5.0+-x64-linux.tar.gz
    tar -xzf ncbi-blast-2.5.0+-x64-linux.tar.gz
    cp ncbi-blast-2.5.0+/bin/* ~/bin
    ```
    
9. [diamond](https://github.com/bbuchfink/diamond): BLAST compatible accelerated aligner

    ```bash
    cd ~/repos
    mkdir diamond && cd $_
    wget http://github.com/bbuchfink/diamond/releases/download/v0.8.31/diamond-linux64.tar.gz
    tar -xzf diamond-linux64.tar.gz
    cp diamond ~/bin
    ```
    
10. [R](https://www.r-project.org/) Finally we need R we followed these steps 
[how to install r on linux ubuntu 16-04 xenial xerus](https://www.r-bloggers.com/how-to-install-r-on-linux-ubuntu-16-04-xenial-xerus/):
and installed the additional packages: gplots ggplot2 getopt reshape


We also need some database files versions of which that are compatible with the pipeline 
we have made available through s3. Below we suggest downloading them to a databases directory:

1. COG RPS database: ftp://ftp.ncbi.nih.gov/pub/mmdb/cdd/little\_endian/ Cog databases
    
    ```bash
    mkdir ~/Databases && cd $_
    wget https://desmandatabases.s3.climb.ac.uk/rpsblast_cog_db.tar.gz
    tar -xzf rpsblast_cog_db.tar.gz
    ```
2.  NCBI non-redundant database formatted in old GI format downloaded 02/08/2016 02:07:05. We provide 
this as fasta sequence so that you can diamond format it yourself to avoid any version issues:
    
    ```bash
    cd ~/Databases
    mkdir NR && cd NR
    wget https://desmandatabases.s3.climb.ac.uk/nr.faa
    diamond makedb --in nr.faa -d nr
    ```
or we also provide a pre-formatted version:
    ```
    wget https://nrdatabase.s3.climb.ac.uk/nr.dmnd
    ```
    
3. GI to Taxaid and lineage files for the above:
    
	```bash
	wget https://desmandatabases.s3.climb.ac.uk/gi_taxid_prot.dmp
	wget https://desmandatabases.s3.climb.ac.uk/all_taxa_lineage_notnone.tsv
	```

We then install both the [CONCOCT](https://github.com/BinPro/CONCOCT) and [DESMAN]((https://github.com/chrisquince/DESMAN)) repositories. Concoct is Python 2.7 whereas Desman is now python3 they require the following modules:

```bash
sudo apt-get -y install python-pip python3-pip 
sudo pip install cython numpy scipy biopython pandas pip scikit-learn pysam bcbio-gff
sudo pip3 install cython numpy scipy biopython pandas pip scikit-learn pysam bcbio-gff
```

Then install the repos and set their location in your .bashrc:
```bash
cd ~/repos
git clone https://github.com/BinPro/CONCOCT.git
cd CONCOCT
sudo python ./setup.py install
cd ~/repos
git clone https://github.com/chrisquince/DESMAN.git
cd DESMAN
sudo python3 ./setup.py install
```

Then add this lines to .bashrc:

```bash
export CONCOCT=~/repos/CONCOCT
export DESMAN=~/repos/DESMAN
```

To begin make working directory and obtain the reads from Dropbox:

```bash
cd ~
mkdir DesmanExample && cd $_
wget https://www.dropbox.com/s/l6g3culvibym8g7/Example.tar.gz
```

Rename, untar and unzip the example directory and move into it:

```bash
tar -xzf Example.tar.gz
cd Example
```
<a name="assembly"></a>
Read assembly, mapping and binning
----------------------------------

Then assemble the reads. We recommend megahit for this:
```bash
nohup megahit -1 $(<R1.csv) -2 $(<R2.csv) -t 36 -o Assembly > megahit.out&
```
This will take a while so we have set megahit running on 36 threads (adjust to your system) and 
run in background with nohup.

We will now perform CONCOCT binning of these contigs. As explained in [Alneberg et al.](http://www.nature.com/nmeth/journal/v11/n11/full/nmeth.3103.html) 
there are good reasons to cut up contigs prior to binning. We will use a script from CONCOCT to do this. For convenience we 
will create environmental variables points to the CONCOCT and DESMAN install directories:

```bash
export DESMAN_EXAMPLE=$HOME/mypathtoDesmanExample/Example
```

Then cut up contigs and place in new dir:

```bash
cd ~/DesmanExample/Example
mkdir contigs
python $CONCOCT/scripts/cut_up_fasta.py -c 10000 -o 0 -m Assembly/final.contigs.fa > contigs/final_contigs_c10K.fa
```

Having cut-up the contigs the next step is to map all the reads from each sample back onto them. First index the contigs with bwa:

```bash
cd contigs
bwa index final_contigs_c10K.fa
cd ..
```

Then perform the actual mapping you may want to put this in a shell script:

```bash
mkdir Map
for file in *R1.fastq; do 
   echo ${stub:=${file%_R1.fastq}}
   file2=${stub}_R2.fastq
   bwa mem -t 32 contigs/final_contigs_c10K.fa $file $file2 > Map/${stub}.sam
done
```

Here we are using 32 threads for bwa mem '-t 32' you can adjust this to whatever is suitable for your machine.
Then we need to calculate our contig lengths using one of the Desman scripts.

```bash
python3 $DESMAN/scripts/Lengths.py -i contigs/final_contigs_c10K.fa > contigs/final_contigs_c10K.len
```

Then we calculate coverages for each contig in each sample:

```bash
for file in Map/*.sam; do
    echo ${stub:=$(basename $file .sam)}
    { 
	samtools view -h -b -S $file > ${stub}.bam
	samtools view -b -F 4 ${stub}.bam > ${stub}.mapped.bam
	samtools sort -m 1000000000 ${stub}.mapped.bam -o ${stub}.mapped.sorted.bam
	bedtools genomecov -ibam ${stub}.mapped.sorted.bam -g contigs/final_contigs_c10K.len > ${stub}_cov.txt
    } &
done && wait # for background jobs to finish
```

and use awk to aggregate the output of bedtools:

```bash
for file in Map/*_cov.txt; do 
   echo ${stub:=$(basename $file _cov.txt)} 
   awk -F"\t" '{l[$1]=l[$1]+($2 *$3);r[$1]=$4} END {for (i in l){print i","(l[i]/r[i])}}' $file > Map/${stub}_cov.csv
done
```

and finally run the following perl script to collate the coverages across samples, where we have simply adjusted the format 
from csv to tsv to be compatible with CONCOCT:

```bash
$DESMAN/scripts/Collate.pl Map | tr "," "\t" > Coverage.tsv
```

and run CONCOCT:
```bash
mkdir Concoct && cd $_
mv ../Coverage.tsv .
concoct --coverage_file Coverage.tsv --composition_file ../contigs/final_contigs_c10K.fa
cd ..
```

<a name="genome_assignment"></a>
Contig genome assignments
-------------------------

In this case we know which contig derives from which of the 20 genomes and so we can compare the assignment of 
contigs to clusters with those genome assignments. To get the genome assignments we first need the 
strain genomes:

```bash
wget https://www.dropbox.com/s/9ozp0vvk9kg2jf0/Mock1_20genomes.fasta
mkdir AssignGenome && mv Mock1_20genomes.fasta $_
```

We need to index our bam files:
```bash
for file in Map/*mapped.sorted.bam; do
    echo $(basename $file .bam)	
    samtools index $file
done
```

Then we run a script that extracts the mock genome ids out of the fastq ids of the simulated reads:
```bash
cd AssignGenome
python3 $DESMAN/scripts/contig_read_count_per_genome.py ../contigs/final_contigs_c10K.fa Mock1_20genomes.fasta ../Map/*mapped.sorted.bam > final_contigs_c10K_genome_count.tsv
cd ..
``` 
This file contains counts of unambiguous and ambiguous reads mapping to each of the genomes for each of the 
contigs. We simplify the genome names and filter these counts:

```bash
$DESMAN/scripts/MapGHeader.pl $DESMAN/complete_example/Map.txt < AssignGenome/final_contigs_c10K_genome_count.tsv > AssignGenome/final_contigs_c10K_genome_countR.tsv
```

Then we get assignments of each contig to each genome:
```bash
$DESMAN/scripts/LabelSMap.pl Concoct/clustering_gt1000.csv AssignGenome/final_contigs_c10K_genome_countR.tsv > AssignGenome/clustering_gt1000_smap.csv
```

This enables to compare the CONCOCT clusterings with these assignments:
```bash
$CONCOCT/scripts/Validate.pl --cfile=Concoct/clustering_gt1000.csv --sfile=AssignGenome/clustering_gt1000_smap.csv --ffile=contigs/final_contigs_c10K.fa 
```
This should generate output similar too:

```
N	M	TL	S	K	Rec.	Prec.	NMI	Rand	AdjRand
9869	9869	6.1623e+07	20	24	0.983277	0.983970	0.984496	0.998533	0.986840
```

The exact results may vary but the overall accuracy should be similar.
We can also plot the resulting confusion matrix:
```bash
$CONCOCT/scripts/ConfPlot.R -c Conf.csv -o Conf.pdf
```

![CONCOCT clusters](complete_example/Conf.pdf)

From this it is apparent that five clusters: D1, D9, D11, D15, and D18 represent the *E. coli* pangenome. In general, 
it will not be known *a priori* from which taxa a cluster derives and so not possible to link them in this way.
However, in many analyses the pangenome will be contained in a single cluster or a contig taxonomic classifier 
could be used to determine clusters deriving from the same species. We illustrate how to do this below.
In your particular run these assignments may vary and the code below must be **changed** accordingly.


<a name="taxa_assignment"></a>
## Taxonomic classification of contigs

There are many ways to taxonomically classify assembled sequence. We suggest a gene based approach. The first step is 
to call genes on all contigs that are greater than 1,000 bp. Shorter sequences are unlikely to contain complete 
coding sequences. 

Set the environment variable NR\_DMD to point to the location of your formatted NR database:
```bash
export NR_DMD=$HOME/Databases/NR/nr.dmnd
```

Then we begin by calling genes on all contigs greater than 1000bp in length.
```bash
cd ~/DesmanExample/Example
mkdir Annotate_gt1000 && cd $_
python3 $DESMAN/scripts/LengthFilter.py -m 1000 ../contigs/final_contigs_c10K.fa > final_contigs_gt1000_c10K.fa
prodigal -i final_contigs_gt1000_c10K.fa -a final_contigs_gt1000_c10K.faa -d final_contigs_gt1000_c10K.fna  -f gff -p meta -o final_contigs_gt1000_c10K.gff
cd ..
```

```bash
mkdir AssignTaxa && cd $_
cp ../Annotate_gt1000/final_contigs_gt1000_c10K.faa .
diamond blastp -p 32 -d $NR_DMD -q final_contigs_gt1000_c10K.faa -a final_contigs_gt1000_c10K > d.out
diamond view -a final_contigs_gt1000_c10K.daa -o final_contigs_gt1000_c10K_nr.m8
```

To classify the contigs we need two files a gid to taxid mapping file and a mapping of taxaid to full lineage:

1. gi\_taxid\_prot.dmp

2. all\_taxa\_lineage\_notnone.tsv

These can also be downloaded from the Dropbox:
```bash
wget https://www.dropbox.com/s/x4s50f813ok4tqt/gi_taxid_prot.dmp.gz
wget https://www.dropbox.com/s/honc1j5g7wli3zv/all_taxa_lineage_notnone.tsv.gz
```

The path to these files are default in the ClassifyContigNR.py script as the variables:
```
DEF_DMP_FILE = "/home/chris/native/Databases/nr/FASTA/gi_taxid_prot.dmp"

DEF_LINE_FILE = "/home/chris/native/Databases/nr/FASTA/all_taxa_lineage_notnone.tsv"
```

We calculate the gene length in amino acids before running this.
Then we can assign the contigs and genes called on them:
```bash
python3 $DESMAN/scripts/Lengths.py -i final_contigs_gt1000_c10K.faa > final_contigs_gt1000_c10K.len
python3 $DESMAN/scripts/ClassifyContigNR.py final_contigs_gt1000_c10K_nr.m8 final_contigs_gt1000_c10K.len -o final_contigs_gt1000_c10K_nr -l /mypath/all_taxa_lineage_notnone.tsv -g /mypath/gi_taxid_prot.dmp
```

Then we extract species out:
```bash
$DESMAN/scripts/Filter.pl 8 < final_contigs_gt1000_c10K_nr_contigs.csv | grep -v "_6" | grep -v "None" > final_contigs_gt1000_c10K_nr_species.csv
```

These can then be used for the cluster confusion plot:
```bash
$CONCOCT/scripts/Validate.pl --cfile=../Concoct/clustering_gt1000.csv --sfile=final_contigs_gt1000_c10K_nr_species.csv --ffile=../contigs/final_contigs_c10K.fa --ofile=Taxa_Conf.csv
```
Now the results will be somewhat different...
```
N	M	TL	S	K	Rec.	Prec.	NMI	Rand	AdjRand
9869	1746	1.5257e+07	17	24	0.985459	0.999801	0.990303	0.999300	0.996926
```

We then plot the out Conf.csv which contains species proportions in each cluster:
```bash
$CONCOCT/scripts/ConfPlot.R -c Taxa_Conf.csv -o Taxa_Conf.pdf 
```

![CONCOCT clusters against taxa](Taxa_Conf.pdf)

This confirms from a *de novo* approach that D0, D14, D17, D18 and D21 represent the *E. coli* pangenome. In your own analysis it will probably be a different set of clusters and hence the code below will have to be adjusted accordingly.

<a name="core_genes"></a>
## Identifying *E. coli* core genes

We now determine core genes single copy genes within these four clusters through annotation to COGs. First lets split the contigs 
by their cluster and concatenate togethers those from D0, D14, D17, D18 and D21 into one file ClusterEC.fa. If your clustering 
gave different bins associated with *E. coli* then change the files selected below as appropriate:

Go back to 
the top level example directory and then:

```bash
mkdir Split && cd $_
$DESMAN/scripts/SplitClusters.pl ../contigs/final_contigs_c10K.fa ../Concoct/clustering_gt1000.csv
cat Cluster0/Cluster0.fa Cluster14/Cluster14.fa Cluster17/Cluster17.fa Cluster18/Cluster18.fa Cluster21/Cluster21.fa > ClusterEC.fa
cd ..
```

Now call genes on the *E. coli* contigs.

```bash
mkdir AnnotateEC && cd $_
cp ../Split/ClusterEC.fa .
prodigal -i ClusterEC.fa -a ClusterEC.faa -d ClusterEC.fna  -f gff -p meta -o ClusterEC.gff
```

Next we assign COGs using the CONCOCT script RPSBLAST.sh. First set location of *your* COG rpsblast database. 
Then run the CONCOCT script. This requires rpsblast and gnu parallel.

```bash
export COGSDB_DIR=~/gpfs/Databases/rpsblast_db
$CONCOCT/scripts/RPSBLAST.sh -f ClusterEC.faa -p -c 8 -r 1
```

and extract out the annotated Cogs associated with called genes:
```bash
python3 $DESMAN/scripts/ExtractCogs.py -g ClusterEC.gff -b ClusterEC.out --cdd_cog_file $CONCOCT/scgs/cdd_to_cog.tsv > ClusterEC.cogs
```

Then we determine those regions of the contigs with core COGs on in single copy using the 982 predetermined *E. coli* core COGs:
```bash
$DESMAN/scripts/SelectContigsPos.pl $DESMAN/complete_example/EColi_core_ident95.txt < ClusterEC.cogs > ClusterEC_core.cogs
```
This contains core COG ids and gene locations i.e:
```
COG0001,k119_1371,8446,9352,k119_1371_9,1
COG0004,k119_5582,920,1970,k119_5582_3,1
COG0012,k119_6421,2,1037,k119_6421_1,-1
COG0013,k119_5506,1310,3941,k119_5506_2,-1
COG0014,k119_3846.0,293,1547,k119_3846.0_2,1
COG0015,k119_7392,581,1952,k119_7392_2,-1
COG0016,k119_10287,405,1389,k119_10287_1,1
COG0017,k119_17329,321,1722,k119_17329_1,-1
COG0018,k119_7639,4300,6034,k119_7639_5,-1
COG0019,k119_6580,570,1833,k119_6580_2,1
```

Now we just reformat the location of core cogs on contigs:

```bash
cut -d"," -f2,3,4 ClusterEC_core.cogs | tr "," "\t" > ClusterEC_core_cogs.tsv
```
This will simply be a bed style tab separated format file i.e.:
```
k119_1371	8446	9352
k119_5582	920	1970
k119_6421	2	1037
k119_5506	1310	3941
k119_3846.0	293	1547
k119_7392	581	1952
k119_10287	405	1389
k119_17329	321	1722
k119_7639	4300	6034
k119_6580	570	1833
```


<a name="determine_variants"></a>
## Determine variants on core COGs

To input into bam-readcount:

```bash
cd ..
mkdir Counts
```

Before doing so though we need to index the contigs fasta file
```bash
samtools faidx contigs/final_contigs_c10K.fa
```

then run bam-readcount:
```bash
for file in Map/*sorted.bam; do
	echo ${stub:=$(basename $file .mapped.sorted.bam)}
	bam-readcount -q 20 -l AnnotateEC/ClusterEC_core_cogs.tsv \
		-f contigs/final_contigs_c10K.fa $file \
		2> Counts/${stub}.err > Counts/${stub}.cnt &
done
```
The above will run each sample in parallel adjust as necessary. To save space we will zip the resulting base frequencies:
```bash
cd Counts
gzip *cnt
cd ..
```

Next we collate the positions frequencies into a single file for Desman, here we use all genes regardless of length:

```bash
python3 $DESMAN/scripts/ExtractCountFreqGenes.py AnnotateEC/ClusterEC_core.cogs Counts --output_file Cluster_esc3_scgs.freq
```

<a name="infer_strains"></a>

## Infer strains with Desman

Now lets use Desman to find the variant positions on these core cogs:
```bash
mkdir Variants && cd $_
mv ../Cluster_esc3_scgs.freq .
Variant_Filter.py Cluster_esc3_scgs.freq
cd ..
```

and run Desman:
```bash
mkdir RunDesman && cd $_
for g in `seq 2 8`; do     
    for r in `seq 0 4`; do             
        desman ../Variants/outputsel_var.csv -e ../Variants/outputtran_df.csv \
		-o ClusterEC_${g}_${r} -r 1000 -i 100 -g $g -s $r \
		> ClusterEC_${g}_${r}.out &
    done
done
cd ..
```
This runs the haplotype inference for between 2 and 8 strains inclusive with 5 replicates. The options used are:
1. _-r 1000_ : The number of randomly selected positions to base inference on
2.  _-i 100_ : The number of Gibbs sampling updates on real data increase this to say 500
3.  _-g $g_ : Number of haplotypes to be inferred
4.  _-s $r_ : Seed for random number generator using integer run index ensures independent replicate runs

First lets have a look at the mean posterior deviance as a function of strain number:
```bash
cat */fit.txt | cut -d"," -f2- > Dev.csv
sed -i '1iH,G,LP,Dev' Dev.csv 
```

which we can plot with a simple R script included in the Desman distribution:
```bash
cd $DESMAN_EXAMPLE
$DESMAN/scripts/PlotDev.R -l RunDesman/Dev.csv -o RunDesman/Dev.pdf
```
![Mean posterior deviance vs. strain number](Dev.pdf)

From this it is not as clear as in the full data set analysed in the paper that 
five strains are present, since on average there is some improvement going 
from five to six strains. However, we can run our heuristic program for determining haplotype number:

```bash
python3 $DESMAN/scripts/resolvenhap.py ClusterEC
```

This gave in our run the following output:

```
6,5,2,0.0165,ClusterEC_6_2/Filtered_Tau_star.csv
```

Which we interpret as the best run had six haplotypes, five of which we are confident in and the average error in those inferences was 1.6%. The best haplotypes are given by the file ClusterEC\_6\_2/Filtered\_Tau\_star.csv. This is what we will use in the analysis below.

<a name="validate_strains"></a>

## Validation of strains

To validate the strain inference we will download pre-identified sequences for each of the 982 single copy core COGs in the five known reference genomes. 

```bash
cd $DESMAN_EXAMPLE
mkdir Validate && cd $_
wget https://www.dropbox.com/s/f6ojp1qt4fz5lzn/Hits.tar.gz
tar -xzf Hits.tar.gz
```

We then select core COGs that were included in our analysis. We reverse those that are reversed on the 
contigs so that positions match and 
then find all variants mapping onto the 0,1 encoding employed in DESMAN.

```bash
mkdir Select
$DESMAN/scripts/Select.sh
$DESMAN/scripts/ReverseStrand.pl ../AnnotateEC/ClusterEC_core.cogs
$DESMAN/scripts/TauFasta.pl
$DESMAN/scripts/CombineTau.pl > ClusterEC_core_tau.csv
$DESMAN/scripts/MapCogBack.pl ../AnnotateEC/ClusterEC_core.cogs < ClusterEC_core_tau.csv > ClusterEC_core_tau_gene.csv
```

We then compare these known assignments to those predicted by DESMAN for our optimum run:
```bash
python3 $DESMAN/scripts/validateSNP2.py ../RunDesman/ClusterEC_6_2/Collated_Tau_mean.csv ClusterEC_core_tau_gene.csv
```

The output should look like:
```
Intersection: 14854
[[9219 4045 9022 6726 3692]
 [8849  625 8784 6124 4190]
 [5127 8765 1349 9308 7850]
 [1155 8738 5002 9148 7975]
 [8793 4451 8713 5843  399]
 [9123 5949 9126  482 5702]]
[[ 0.6206409   0.27231722  0.60737848  0.45280732  0.24855258]
 [ 0.59573179  0.04207621  0.59135586  0.41227952  0.2820789 ]
 [ 0.34515955  0.59007675  0.09081729  0.62663256  0.52847718]
 [ 0.07775683  0.58825905  0.33674431  0.61586105  0.53689242]
 [ 0.59196176  0.29964993  0.58657601  0.39336206  0.02686145]
 [ 0.614178    0.40049818  0.61437996  0.03244917  0.38386966]]
```

This gives for each (row) predicted haplotype (or posterior mean in fact) the fraction of SNPs differing to each 
of the five reference genomes (columns). We see that each strain maps to one genome with an error rate between 2.7% and 9.1%. 

<a name="assign_acessory"></a>

## Determine accessory genomes

Now we need the variant frequencies on all contigs:

```bash
cd $DESMAN_EXAMPLE
mkdir CountsAll
python3 $DESMAN/scripts/Lengths.py -i AnnotateEC/ClusterEC.fa > AnnotateEC/ClusterEC.len
$DESMAN/scripts/AddLengths.pl < AnnotateEC/ClusterEC.len > AnnotateEC/ClusterEC.tsv
for file in Map/*sorted.bam; do
    echo ${stub:=$(basename $file .mapped.sorted.bam)}
    bam-readcount -w 1 -q 20 -l AnnotateEC/ClusterEC.tsv -f contigs/final_contigs_c10K.fa $file \
    	> CountsAll/${stub}.cnt 2> CountsAll/${stub}.err &
done
```

Gzip them up to save space and because the downstream programs expect this format:
```bash
cd CountsAll
gzip *.cnt
cd ..
```

We also need to extract info on all genes in the E. coli clusters:
```bash
python3 $DESMAN/scripts/ExtractGenes.py -g AnnotateEC/ClusterEC.gff > AnnotateEC/ClusterEC.genes
```

Then we collate the count files together filtering to genes greater than 500bp:
```bash
python3 $DESMAN/scripts/ExtractCountFreqGenes.py -g AnnotateEC/ClusterEC.genes CountsAll --output_file Cluster_esc3.freq 
```
The _-g_ flag here tells the script to expect gene positions in a slightly different format to the cog file used above. Now we find variants again, this time insisting on a minimum frequency of 3% and not filtering on sample coverage:
```bash
mkdir VariantsAll && cd $_
mv ../Cluster_esc3.freq .
Variant_Filter.py Cluster_esc3.freq -m 0.0 -v 0.03
```

To assign contigs we also need individual gene coverages, for consistency we generate these from the 
aggregated count files:

```bash
python3 $DESMAN/scripts/CalcGeneCov.py Cluster_esc3.freq > Cluster_esc3_gene_cov.csv
```

Get list of core COGs:
```bash
cut -d"," -f5 ../AnnotateEC/ClusterEC_core.cogs > ClusterEC_core_genes.txt
```

Calculate coverage on core genes:
```bash
python3 $DESMAN/scripts/CalcDelta.py Cluster_esc3_gene_cov.csv ClusterEC_core_genes.txt ClusterEC_core
```

Select run with lowest deviance and 5 strains:
```bash
export SEL_RUN=$DESMAN_EXAMPLE/RunDesman/ClusterEC_6_2/
```

Then we run the gene/contig assignment algorithm.
```bash
python3 $DESMAN/desman/GeneAssign.py \
		ClusterEC_coremean_sd_df.csv $SEL_RUN/Gamma_star.csv Cluster_esc3_gene_cov.csv $SEL_RUN/Eta_star.csv \
		-m 20 -v outputsel_var.csv -o ClusterEC --assign_tau \
		> ClusterEC.cout &
```

This should generate the following output files.

1. ClusterEC\_log\_file.txt: A log file

2. ClusterECeta\_df.csv: The assignments from NMF unmanipulated useful for identifying multicopy genes.

3. ClusterECetaD\_df.csv: As above but discretised NMF predictions.

4. ClusterECetaS\_df.csv: Predictions from the Gibbs sampler selecting run with maximum log posterior.

5. ClusterECetaM\_df.csv: Mean log posterior predictions from Gibbs sampler.


<a name="validate_acessory"></a>

## Validate accessory genomes

We will now compare predictions with known assignments to reference genomes. First we 
use the mapping files to determine number of reads from each genome mapping to each gene.
```bash
python3 $DESMAN/scripts/gene_read_count_per_genome.py \
	../AnnotateEC/ClusterEC.genes ../AssignGenome/Mock1_20genomes.fasta \
	../Map/*mapped.sorted.bam > ClusterEC_gene_counts.tsv
```

As above we will rename the header file to be a bit more presentable:
```bash
$DESMAN/scripts/MapGHeader.pl $DESMAN/complete_example/Map.txt < ClusterEC_gene_counts.tsv > ClusterEC_gene_countsR.tsv
```
and select just unambiguous assignments to E. coli genomes:
```bash
cut -f1-6 < ClusterEC_gene_countsR.tsv > ClusterEC_gene_counts_unamb.tsv
```

We then do a little bit of R to convert the counts into gene assignments to genomes assuming that if more than 
1% of reads mapping to a gene derive from a genome then that gene is present in that genome.
```R
# R code
Gene_eta <- read.table("ClusterEC_gene_counts_unamb.tsv", header=TRUE, row.names=1)
Gene_etaP <- Gene_eta/rowSums(Gene_eta)
Gene_etaP[Gene_etaP > 0.01] <- 1
Gene_etaP[Gene_etaP <= 0.01] <- 0
write.csv(Gene_etaP, "Gene_etaP.csv", quote=FALSE)
```

Finally we compare the mean posterior predictions to those assignments.
```bash
python3 $DESMAN/scripts/CompAssign.py ClusterECetaM_df.csv Gene_etaP.csv
```

Output should look like:

```
0.9791
0.9624
0.9663
0.9431
0.9657
Av. accurracy = 0.963333
```
