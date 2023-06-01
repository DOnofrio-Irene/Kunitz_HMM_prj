# Kunitz_HMM_prj
Laboratory of Bioinformatics project aiming at the generation of an HMM model for the annotation of the Kunitz domain. 
In this repository there are all the datasets, python scripts and the entire pipeline. The steps of the entire pipeline can also be found inside the knz_prj.sh file.
To run this pipeline the following programms need to be installed:
- HMMER  3.3.2
- CD-HIT (version 4.8.1)
- BLAST+


## 1. SELECTION OF THE TRAINING SET
### Advanced search on PDB DATABASE
The selection of a representative training set of structurally defined proteins was carried out by means of an advanced search in the RCSB PDB database (wwPDB consortium, 2019). The constraints used were: 
+ Identifier - Pfam Protein Family = PF00014
+ Refinement Resolution <= 2.50
+ Polymer Entity Sequence Lenght =  49-90 

A tabular report was customized with the following field: ```PDB ID``` and ```Auth Asym ID```.
The tabular report of the list of retrieved sequences was downloaded in CSV format. This file was cleaned up using the following command:
```
tail -n +3 PDBnogrouping.csv | grep -v "^,," | tr -d \" > PDBnogrouping.tmp && mv PDBnogrouping.tmp PDBnogrouping
```

To account for the redundancy of the PDB structures downloaded, it is necessary to perform a clustering procedure with CD-HIT. CD-HIT is  greedy incremental algorithm that starts with the longest input sequence as the first cluster representative and then processes the remaining sequences from long to short to classify each sequence as a redundant or representative sequence based on its similarities to the existing representatives (Fu et al., 2012).

### FASTA sequences download
Since the command takes in input FASTA files, FASTA sequences of the entities fetched from PDB need to be downloaded:
- extract from the tabular report the PDB ID
```
cut -d "," -f 2 PDBnogrouping.csv  > FASTA_to_download.list
```
- Using ```wget``` download the FASTA files, looping on the list previosuly computed and redirect all the sequences in a unique multi-FASTA file
 ```
for i in  `cat FASTA_to_download.list`; do wget https://www.rcsb.org/fasta/entry/$i ; done 
for i in `cat FASTA_to_download.list` ; do cat $i ; done > PDBnogrouping.fasta  
```
In this multi-FASTA there are the sequences of the entire entities, thus it is necessary to extract only the sequences of the target chain IDs, that can be retrieved from the tabular report:
```
cut -d "," -f 1 PDBnogrouping.csv > PDBchains_nogrouping.list 
```
To filter the multi-FASTA and exclude the chain we are not interested in, run the ``` filteringFASTA.py ``` Python script that can be found inside this repository:
```
python3 filteringFASTA.py PDBchains_nogrouping.list PDBnogrouping.fasta filtered_PDBnogrouping.fasta
```

### CD-HIT clustering
The cluster was performed with a threshold of 95%
```
cd-hit -i filtered_PDBnogrouping.fasta -o seeds.fasta -c 0.95 
```

## MSA AND HMM GENERATION
### Mulitple structure alignment
The representative list was submitted to [PDBeFold v2.59](https://www.ebi.ac.uk/msd-srv/ssm/) alignment program to obtain a multiple structure alignment (Krissinel and Henrick, 2004). 
To compile the list with the ```PDB ID``` + ```Auth Asym ID``` to upload on PDBeFold, it is necessary to extract the two fields from the tabular report. First we extract the entity IDs from the multi-FASTA, and then we retrieve the```PDB ID``` + ```Auth Asym ID``` pairs from the tabular report, using the ```extract_chainIDs_seeds.py```  :
```
grep '^>' seeds.fasta | cut -d "|" -f1 | cut -c2- > entityIDs_seeds.list
python extract_chainIDs_seeds.py entityIDs_seeds.list PDBnogrouping.csv training_seqs.list
```
### HMM generation
The resulting seed MSA is provided as input to hmmbuild function of HMMER 3.3.2. to obtain a profile-HMM (Finn et al., 2011)
```
hmmbuild kunitz.hmm seeds_MSA.seq
```

## TEST SET GENERATION
Download the entire UniProtKB/Swiss-Prot database, containing 569516 sequence entries (release 2023_02 of 03-May-2023), which will be the test set to validate the model. To ensure a fair evaluation of the HMM model, proteins sharing a high level of sequence identity with the representatives need to be excluded from the test set. Identification of redundant proteins was carried out using the blastpgp program (Altschul et al., 1997).
```
makeblastdb -in uniprot_sprot.fasta -out $fasta_db -dbtype prot
blastpgp -i seeds.fasta -d uniprot_sprot.fasta -m 8 -o blastpgp_results.bl8
```
The output file is ranked based on the sequence identity value, and only the sequences that showed a S.I > 95% are selected and their UniProt IDs are copied into a new file.
```
sort -k 3 -n -r  blastpgp_results.bl8 > tmp && mv tmp blastpgp_results.bl8
awk '$3 >= 95.00' blastpgp_results.bl8 > morethan95%.bl8
awk -F '|' '{print $3}' morethan95%.bl8 > redundant_seqs.list
```
To check if all the representatives are detected by blastpgp, it is necessary to compare the list of the UniProt IDs of the representatives with the list of the redundant proteins found by blastpgp.
1. Mapp the UniProt IDs of the representatives using the [ID Mapping tool](https://www.uniprot.org/id-mapping) provided by UniProt. Upload the list of the  ```PDB ID``` + ```Auth Asym ID``` pairs and mapp from PDB to UniProtKB/Swiss-Prot. Download the TSV file (mapped_IDs.tsv).
2. Compare the two lists:
``` comm <(sort redundant_seqs.list) <(sort  $ids_mapped)```
3. Merge the unique results of the two lists together:
``` sort -u mapped_IDs.tsv redundant_seqs.list > toberemoved_seqs.list```
