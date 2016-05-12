# Preamble

We sequenced a genome. Stick a sample of your organism on an Illumina machine, get ATGCs, quality check them and match them together. Done? Nope. Do some gene predictions. Okay. Done? NO!

You got to upload this data to NCBI or the EBI. Easy right? Not really. How long have you got?

NCBI/EBI don't really want GFF files (output from gene prediction programs). They don't really want FASTA files either (your scaffolds/contigs are in this format, but not the annotated ones).

Okay so what do they want? Well their own formats of course. Le sigh.

This is the long needlessly complicated process to go from a FASTA file and a file of Gene Features (GFF) to something that might be remotely acceptable for upload to EBI.

Luckily some nice people have created this [GAG](https://github.com/genomeannotation/GAG.git) - Genome Annotation Generator that will read a GFF3 and convert it to the NCBI table format which is not useful to us, but it doesn't hurt to use for a nicer to deal with gff.

We are also going to use their other program [ANNIE](https://genomeannotation.github.io/annie) - to give annotation information to GAG - but first we need to do InterProScan and SwissProt blasts.

Before we can upload a set of scaffold/contigs/genome we need to have uploaded the sequence libraries as BioSamples to a BioProject. Our BioProject was already created - PRJEA61035 - which complicates things further

You have to contact EBI and let them know and they do something on their end and then I create a new BioProject and BioSample!?!

# NCBI

We are not doing this for Hyphochytrium as a previous assembly, biosample and bioproject were already created via EBI. Therefore we need to do a not so easy update!

But we can upload the HiSeq data here - why not complicate it more eh? ;)

## 1. Upload Sequence Libraries
 * https://www.ncbi.nlm.nih.gov/sra/docs/submitportal/
 * for a start I can't login within Chrome. It works on FireFox.
 * First you need to make a "template" for submission - https://submit.ncbi.nlm.nih.gov/biosample/template/
 * Then you need a sample sheet for SRA metadata.
 * Fill in all the information on the web-upload forms and the two .tsv files
 * Set a release date: 18 May 2017
 * Check for submission status here - http://trace.ncbi.nlm.nih.gov/Traces/sra_sub/sub.cgi?view=submissions

 * HiSEq: Submission: SUB1472106 and SRA: SRR3401404 and Sample: SAMN04868781

 * For some reason NCBI's email say that the SRA is SRP073437 http://www.ncbi.nlm.nih.gov/sra/SRP073437

#EBI

* I have been instructed to get ready an EMBL file format version of the data!?!
* Also it needs to be GZIPED. Which I found out only after going through all of the web upload and form filling. NOWHERE does it mention this little requirement. FFS.

## 2. Genome Submission
 * Upload >1Kbp scaffolds + mito
 * https://www.ebi.ac.uk/ena/submit/sequence-submission
 * Create a new BioProject, create a new Sample (indicate in this sample it is a mix of this sample and another sample)
 * create assembly submissions (quietly weep inside and the pain this cause)
 * nb - if your supervisor ever suggests another genome project; fire bomb the building, change your name and join the foreign legion

## Annotation Steps

We should really remap the default MAKER output names to something more readable/simple

### Create Map
    ~/maker/bin/maker_map_ids --prefix Hypho2016_ hyphochytrium_catenoides_genome_ge_1k_no_seqs.gff > hyphochytrium_catenoides_genome_map_ids.txt

### Remap Fasta and GFF3
    ~/maker/bin/map_fasta_ids hyphochytrium_catenoides_genome_map_ids.txt hyphochytrium_catenoides_predicted_proteins_renamed.fasta

    ~/maker/bin/map_gff_ids hyphochytrium_catenoides_genome_map_ids.txt hyphochytrium_catenoides_genome_ge_1k_no_seqs_renamed.gff

### Remove end of line ';' as they will break ANNIE
    sed -i 's/;$//g' hyphochytrium_catenoides_genome_ge_1k_no_seqs_renamed.gff

### SwissProt BLASTp
    blastp -query hyphochytrium_catenoides_predicted_proteins_renamed.fasta -db uniprot_sprot.fasta -outfmt 6 -num_threads 24 -evalue 1e-10 -out all_maker_proteins_vs_swissprot_1e-10.tsv

### InterProScan
    interproscan.sh -f TSV,XML,GFF3 -goterms -i hyphochytrium_catenoides_predicted_proteins_renamed.fasta -pa | tee interproscan.log

## Annie

### InterproScan
    python3 ~/annie/annie.py -ipr second_pass.all.maker.proteins.fasta.tsv

### BLASTp
    python3 ~/annie/annie.py -b ../all_maker_proteins_vs_swissprot_1e-10.tsv -g ../hyphochytrium_catenoides_genome_ge_1k.gff -db /storage/swissprot/uniprot_sprot.fasta

## GAG
The default output of GFF files from MAKER includes the FASTA sequences. GAG does not like this, you have to remove it from the bottom of the file...
```
python ~/GAG/gag.py --fasta hyphochytrium_catenoides_genome_ge_1k_scaffolds.fasta \
--gff hyphochytrium_catenoides_genome_ge_1k_no_seqs.gff \
--anno annie/annie_blast_swissprot_output.tsv \
--fix_terminal_ns --fix_start_stop --flag_introns_shorter_than 10
```

## Convert GFF and FASTA to EMBL

* use EMBOSS tool SEQRET
    seqret -sequence genome.fasta -feature -fformat gff -fopenfile genome.gff -osformat embl -auto
* there's still more to do for EBI. Why? Because.
* It's got to look something like this... EUGH
* https://www.ebi.ac.uk/ena/submit/scaffold-flat-file
