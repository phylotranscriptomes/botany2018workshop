
This tutorial is for the Botany 2018 workshop "Transcriptome analyses for non-model plants: phylogenomics and more" Jul 22, 2018. All python scripts were written in python 2.7. 

### Part 1: Lauch Atmosphere Image

Log in the Cyverse website and launch the Atmosphere.

Click on the *Images* tab and search for *botany2018*. Click in the image called *Botany2018_transcriptome_wksp* and then in blue *Launch* icon. Select the appropriate instance size (depending on the resources allocated by Cyverse) and click in *Launch instance*

Wait until the *status* of the instance is active and an *IP Address* has been assigned. Copy the IP address.

Open the terminal in your laptop and connect to the instance you just launched using the IP address:

$ ssh 'your_cyverse_user'@'ip_address'

eg:

$ ssh dfmoralesb@128.196.142.84

Enter your cyverse password.

### Part 2: Download workshop materials and run the setup script

Once you logged in the Atmosphere instance type:

$ ls

You'll see only a folder called *Desktop*

Change directory to *Desktop* and clone git repository with materials.

$ cd Desktop/

$ git clone https://dfmoralesb@bitbucket.org/dfmoralesb/botany_2018.git

When the cloning is done do:

$ cd botany_2018/
$ bash run_this_first.sh

This will setup and install missing dependencies. When this process is done, log out from the instance and log in back, so the changes make effect:

$ exit
$ ssh 'your_cyverse_user'@'ip_address'
Enter your cyverse password.

Now you're ready to run any script from the workshop.

#### To load GUI (mostly to open FigTree, but you can also use the Terminal here)

Download [VNC viewer](https://www.realvnc.com/en/connect/download/viewer/) and install the corresponding flavor for your OS.

Open VNC viewer

Click in the **File*** tab and ***New conection***

In *VNC server* add the IP address following of **:1**. eg: 128.196.142.84:1 and click OK

Double click in the screen icon that has IP address that you just entered

In the Authentication window enter your Cyverse's user and password


### Part 3: Read processing

In this section we'll show you how to process the reads before _de novo_ assembly. We use this pipeline either for *SRA* or newly sequence data. 
For this example we subsample a library of _Iresine__rhizomatosa_. The two .gz files contain read pairs from a stranded mRNA library prepared using the KAPA stranded RNA-seq kit with poly-A enrichment. 

$ cd Desktop/botany_2018/examples/read_proc_assembly_trans/data/

The pipeline does:

1. Random sequencing error correction with [Rcorrector](https://github.com/mourisl/Rcorrector)
2. Removes read pairs that cannot be corrected
3. Remove sequencing adapters and low quality sequences with [Trimmomatic](http://www.usadellab.org/cms/?page=trimmomatic)
4. Filter organelle reads (cpDNA, mtDNA or both) with [Bowtie2](http://bowtie-bio.sourceforge.net/bowtie2/index.shtml). Files containing only organelle reads will be produced which can be use to assemble for example the plastomes with [Fast-Plast](https://github.com/mrmckain/Fast-Plast)
5. Runs [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/) to check read quality and detect over-represented reads
6. Remove Over-represented sequences

The script *filter_fq.py* will run all this steps for a given mRNA library. For our example run:

$ python ~/Desktop/botany_2018/scripts/filter_fq.py SRR6435359red_1.fastq SRR6435359red_2.fastq Caryophyllales both 8 .

The first two arguments are the read files. *Caryophyllales* is the plant Order that bowtie2 will used to create a database to filter the organelle reads and can be replaced with any plant Order where you study group belongs. The argument 'both' speciefies to filter cpDNA and mtDNA (can be used only one of those if wanted). *8* is the number of cpus or threads to use and the final argument *"."* represent the output directory, in this case the current directory.

To see the argument needed to run a scripts type call the script without arguments like:

$ python ~/Desktop/botany_2018/scripts/filter_fq.py

This apply for all scripts provided.

This will produced the filtered filles called *SRR6435359red_1.overep_filtered.fq* and *SRR6435359red_2.overep_filtered.fq* and the files containing the organelle reads called *SRR6435359red_1.org_reads.fq* and *SRR6435359red_1.org_reads.fq*

This script also produces several intermediate files that won't be used anymore and that are pretty large and need to be removed. For this run the same command used previously plust the *clean* at the end. This option can be used from the beginning, but I like to make sure that the final output is correct before removing intermediate files. 

$ python ~/Desktop/botany_2018/scripts/filter_fq.py SRR6435359red_1.fastq SRR6435359red_2.fastq Caryophyllales both 8 . clean


In case any of this processing steps want to be avoided, individual wrappers for each of the 6 steps can be found in the *scripts* folder.


### Part 4: _de novo_ assembly with [Trinity](https://github.com/trinityrnaseq/trinityrnaseq/wiki)
	
Choose a taxonID for each data set. This taxonID will be used throughout the analysis. Use short taxonIDs with 4-6 letters and digits with no special characters. Here we use *Irerhi* as the taxonID, which is short for _Iresine__rhizomatosa_.

The assembly can take several hours (even days, depending of the size of the library) and uses a lot of resources. Here we will try to assemble the subsample library of _Iresine__rhizomatosa_. Remember that you need to used the filtered reads for the _de novo_ assembly.

$ python ~/Desktop/botany_2018/scripts/trinity_wrapper.py SRR6435359red_1.overep_filtered.fq SRR6435359red_2.overep_filtered.fq Irerhi 8 30 stranded .

The first two arguments are the filtered reads. *Irerhi** is the taxonID. *8* is the number of threads. *30* is maximun amount of memory RAM to used. *stranded* specifies if the library is stranded or not (stranded or non-stranded). The final argument *"."* represent the output directory, in this case the current directory.

If the assembly is taking to long you can stop it and find the final output in ~/Desktop/botany_2018/examples/read_proc_assembly_trans/assembly


### Part 5: Transcript filtering and Translation using [TransDecoder](https://github.com/TransDecoder/TransDecoder/wiki) with blastp for choosing orfs

cd to the folder contining the transcriptome assembly

$ cd ~/Desktop/botany_2018/examples/read_proc_assembly_trans/assembly

#### Assembly quality

Now we're going to perform a _de novo_ assembly quality analysis with [Transrate](http://hibberdlab.com/transrate/)

$ python ~/Desktop/botany_2018/scripts/transrate_wrapper.py Irerhi.Trinity.fasta ../filtered_reads/SRR6435359red_1.overep_filtered.fq.gz ../filtered_reads/SRR6435359red_2.overep_filtered.fq.gz  16 .

The results will be found in folder Irerhi.Trinity_transrate_results

#### Assembly filtering 

We're gonna used the transrate results to filter "bad" or low supported transcripts, based on three out of four transrate score components 

$ python ~/Desktop/botany_2018/scripts/filter_transcripts_transrate.py Irerhi.Trinity.fasta Irerhi.Trinity_transrate_results/Irerhi.Trinity/contigs.csv .

This will produced a file with only the good transcripts and short names calles Irerhi.good_transcripts.short_name.fa (so it doesn't produce a error with blastx in latter steps)

#### Chimera removal

Now we're gonna remove chimeric transcripts (using the method from Yang, Y. and S.A. Smith Optimizing de novo assembly of short-read RNA-seq data for phylogenomics. BMC Genomics 2013, 14:328 doi:10.1186/1471-2164-14-328). This a blast-based methods that depending of the size of the trascriptome can take a up to a couple of hours.

$ python ~/Desktop/botany_2018/scripts/run_chimera_detection.py Irerhi.good_transcripts.short_name.fa ~/Desktop/botany_2018/databases/Beta.fa 8 .

This will produced a file called *Irerhi.filtered_transcripts.fa** and the Irerhi.chimera_transcripts.fa

#### Transcript clustering with [Corset](https://github.com/Oshlack/Corset)

Corset clusters transcripts from the same putative gene based in read share and we use it to extract one representative transcripts per gene. In this case the transcript to be keept is the largest within a cluster.

##### Run corset and extract representative transcript

$ python ~/Desktop/botany_2018/scripts/corset_wrapper.py Irerhi.filtered_transcripts.fa ~/Desktop/botany_2018/examples/read_proc_assembly_trans/filtered_reads/SRR6435359red_1.overep_filtered.fq.gz ~/Desktop/botany_2018/examples/read_proc_assembly_trans/filtered_reads/SRR6435359red_2.overep_filtered.fq.gz 8 . salmon

This will produced the file *Irerhi_salmon-clusters.txt* that is gonna be used to select the representative trascript

$ python ~/Desktop/botany_2018/scripts/filter_corset_output.py Irerhi.filtered_transcripts.fa Irerhi_salmon-clusters.txt .

This will produced the file *Irerhi.largest_cluster_transcripts.fa*. This is the final filtered transcript file that will be used for translation.

### Translation with Transdecoder

TransDecoder provides two options for choosing the orfs to retain after scoring candidate orfs. Ya benchmarked blastp vs. PfamAB vs. both and found that since we have closely related, high quality proteomes available for the Caryophyllales, blastp using a custom blast database is much faster (hours vs. days) and more sensitive than Pfam. You should try both options and see the difference for your data sets. Here we use the blastp-only option.

First we need to create a custom blast database using closely-related, high quality proteomes from Arabidopsis thaliana and Beta vulgaris:

This was already done and the db can be found in ~/Desktop/botany_2018/databases/. But here is how was created.

$ cat Atha.fa Beta.fa > db
$ makeblastdb -in ./db -parse_seqids -dbtype prot -out db

Then we need to find candidate orfs, BLAST all candidate orfs against reference we created and output the final orfs preferentially retaining the orfs with blast hits. This will be done with the script transdecoder_wrapper.py

$ python ~/Desktop/botany_2018/scripts/transdecoder_wrapper.py Irerhi.largest_cluster_transcripts.fa 8 stranded .

This will produced the CDS and PEP files *Irerhi.cds.fa* and *Irerhi.pep.fa*. This are the files that have to be used for homology search.

Notice that the format of the sequences if taxonID@seqID. The special character "@" is used to separate taxonID and seqID.



### Part 2: all-by-all homology search

The input sequences for homology inference can be cds or peptides depending on how closely-related the focal taxa are. We provided a small data set of five taxa in the directory ~/Desktop/botany_2018/examples/homology_inference/data. Because these five species are fairly closely-related (< 50 my), using cds for mcl can avoid creating large clusters without loosing sequences.

Before homology search, it always worth spending some time to make sure that the sequence names are formated correctly and peptides and cds have matching sequence names. Check for duplicated names, special characters other than digits, letters and "_", all names follow the format taxonID@seqID, and file names are the taxonID. It's good to check especially when some of the data sets were obtained from elsewhere. Most peptides and CDS files from genome annotation contain long names, spaces and special characters and should be eliminated before clustering.

$ cd ~/Desktop/botany_2018/examples/homology_inference/

$ python ~/Desktop/botany_2018/scripts/check_names.py data/ fa

#### Clustering (we ran the clustering ahead of time already)
Reduce redundancy for each CDS file using cd-hit-est (use -r 0 since these should all be positive strand after translation):

$ cd ~/Desktop/botany_2018/examples/homology_inference/data

$ cd-hit-est -i Beta.cds.fa -o Beta.fa.cds.cdhitest -c 0.99 -n 10 -r 0 -T 2
$ cd-hit-est -i HURS.cds.fa -o HURS.fa.cds.cdhitest -c 0.99 -n 10 -r 0 -T 2
$ cd-hit-est -i NXTS.cds.fa -o NXTS.fa.cds.cdhitest -c 0.99 -n 10 -r 0 -T 2
$ cd-hit-est -i RNBN.cds.fa -o RNBN.fa.cds.cdhitest -c 0.99 -n 10 -r 0 -T 2
$ cd-hit-est -i SCAO.cds.fa -o SCAO.fa.cds.cdhitest -c 0.99 -n 10 -r 0 -T 2

You can open the directory ~/Desktop/botany_2018/examples/homology_inference/1_cd-hit-est to see the ouput files. Concatenate all the .cdhitest files into a new file all.fa

$ cat *.cdhitest >all.fa

Move the all.fa file all-by-all blast to a new directory 2_clustering and navigate into this new directory. We ran these commands ahead of time to carry out the all-by-all blast

$ makeblastdb -in all.fa -parse_seqids -dbtype nucl -out all.fa

$ blastn -db all.fa -query all.fa -evalue 10 -num_threads 2 -max_target_seqs 30 -out all.rawblast -outfmt '6 qseqid qlen sseqid slen frames pident nident length mismatch gapopen qstart qend sstart send evalue bitscore'

You can open the output file all.rawblast to see what the results look like. The columns are as specified by the -outformat flag. Filter raw blast output by hit fraction (proportion blast hit coveration) and prepare input file for mcl. I usually use 0.3 or 0.4 for hit\_fraction_cutoff. A lower hit-fraction cutoff will output clusters with more incomplete sequences and larger and sparser alignments, whereas a high hit-fraction cutoff gives tighter clusters but ignores incomplete or divergent sequences.

$ python ~/Desktop/botany_2018/scripts/blast_to_mcl.py all.rawblast 0.4

It outputs between taxa blast hits that are nearly identical in all.rawblast.ident. These can be from contamination, but can also be bacteria and fungal sequences. Double check when two samples have an unusually high number of identical sequences and this can be a sign of contamination. 

Run mcl. "--te" specifies number of threads, "-I" specifies the inflation value, and -tf 'gq()' specifies minimal -log transformed evalue to consider, and "-abc" specifies the input file format.

$ mcl all.rawblast.hit-frac0.4.minusLogEvalue --abc -te 2 -tf 'gq(5)' -I 1.4 -o hit-frac0.4_I1.4_e5

The file hit-frac0.4_I1.4_e5 contains the clusters output from mcl, one cluster per line. Write fasta files for each cluster from mcl output that have all 5 taxa. Make a new directory 3_clusters to put the thousands of output fasta files.

$ python ~/Desktop/botany_2018/scripts/write_fasta_files_from_mcl.py all.fa hit-frac0.4_I1.4_e5 5 ../3_clusters/

Now we have a new directory with fasta files that look like cluster1.fa, cluster2.fa and so on. 

#### Build homolog trees (we ran it ahead of time already)

Align each cluster, trim alignment, and infer a tree. For tutorial purpose let's only use 10% of the clusters (the ones with cluster ID end with 0). For cluster with less than 1000 sequences, it is aligned with mafft (--genafpair --maxiterate 1000), trimmed by a minimal column occupancy of 0.1 and tree inference using raxml. For larger clusters it is aligned with pasta, trimmed by a minimal column occupancy of 0.01 and tree inference using fasttree. The ouput tree files look like clusterID.raxml.tre or clusterID.fasttree.tre for clusters with 1000 or more sequences. 

$ python ~/Desktop/botany_2018/scripts/fasta_to_tree_pxclsq.py 3_clusters 2 dna n

You can visualize some of the trees and alignments in 3_clusters. You can see that tips that are 0.4 or more are pretty much junk. There are also some tips that are much longer than near-by tips that are probably results of assembly artifacts.

Trim spurious tips with [TreeShrink](https://github.com/uym2/TreeShrink)

$ python ~/Desktop/botany_2018/scripts/tree_shrink_wrapper.py 3_clusters .tre

It outputs the tips that were trimmed in the file .txt and the trimmed trees in the files .tt 

Next, mask both mono- and (optional) paraphyletic tips that belong to the same taxon, and keep the tip that has the most un-ambiguous charactors in the trimmed alignment. 

$ python ~/Desktop/botany_2018/scripts/mask_tips_by_taxonID_transcripts.py 3_clusters 3_clusters y

The results are the .mm files. You can open up some of the larger tree files, such as cluster10.raxml.tre, cluster10.raxml.tre.tt, cluster10.raxml.tre.tt.mm and compare them. You may also notice that there are some long branches separating orthogroups. In this simple example these long branches are not particularly bad but with larger data set cutting long internal branches often significantly improve alignments.

Cut long internal branches longer than 0.2. 

$ python ~/Desktop/botany_2018/scripts/cut_long_internal_branches.py  3_clusters/ .mm 0.2 5 4_refine/

The output are .subtree files in 4_refine. Write fasta files from these .subtree files and repeat the alignment and tree building in 4_refine. 

$ python ~/Desktop/botany_2018/scripts/write_fasta_files_from_trees.py 2_clustering/all.fa 5_homolog .subtree 5_homolog

Calculate the final homolog trees and bootstrap in 5_homolog

$ python ~/Desktop/botany_2018/scripts/fasta_to_tree_pxclsq.py 5_homolog 2 dna y

#### Paralogy pruning to infer orthologs (try them out!).

####1to1: only look at homologs that are strictly one-to-one. No cutting is carried out.

$ cd ~/Desktop/botany_2018/examples/homology_inference
$ mkdir ortho_121
$ mkdir ortho_121/tre
$ python ~/Desktop/botany_2018/scripts/filter_1to1_orthologs.py 5_homolog .tre 5 ortho_121/tre

The script will write one-to-one ortholog trees to the directory ortho_121/tre. 

####RT: prune by extracting ingroup clades and then cut paralogs from root to tip. If no outgroup, only use those that do not have duplicated taxa. Compile a list of ingroup and outgroup taxonID, with each line begin with either "IN" or "OUT", followed by a tab, and then the taxonID.

$ cd ~/Desktop/botany_2018/examples/homology_inference
$ mkdir ortho_RT
$ mkdir ortho_RT/tre
$ python ~/Desktop/botany_2018/scripts/prune_paralogs_RT.py 5_homolog .tre ortho_RT/tre 4 in_out

All the orthologs have full taxon occupancy (5 for 1-to-1 and 4 for RT). 

