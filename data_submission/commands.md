# Preamble

In sequencing a genome you usually end up with many files, perhaps the most important of these are the contigs/scaffolds/chromosomes in FASTA format and the gene predictions you have created.
They usually will be in FASTA too - including CDS and protenis as Amino Acids - but they can also be represented by a GFF file, and indeed that is the output of MAKER.

However, NCBI/EBI don't want GFF files and they don't really want FASTA files either, that is, if you want to include annotation information e.g. gene predictions.

They have their own format, and NCBI generally uses the SEQUIN Table format and EBI use the EMBL format.

Therefore you need to convert your FASTA file and file of Gene Features (GFF) to something that might be remotely acceptable for upload, in this case, to EBI.

It's quite an annoying and complex process, and EBIs help guides are woefully inadequate and miss out a lof of quite useful information (it is incredibly frustrating), so woeful the online help; they organise whole days to help people to upload their data.

Luckily some nice people have already gone through much of these complications and created some nice tools:

* [GAG](https://github.com/genomeannotation/GAG.git) - Genome Annotation Generator
  * Reads a GFF3 file and converts it to the NCBI table format, not useful to us here, but it does make for a nicer to deal with gff than the one MAKER gives us.
* [ANNIE](https://genomeannotation.github.io/annie) - ANNotation Information Extractor
  * to give annotation information to GAG - first we need to do InterProScan and/or SwissProt blasts.
* [gff3toembl](https://github.com/sanger-pathogens/gff3toembl) - Converts PROKKA GFF3 files to EMBL files for uploading annotated assemblies to EBI.
  * Although this is aimed at Prokaryotes, it's quite agnostic in input/output files, therefore we can use it to convert GFF+FASTA to EMBL! YAY :)

Before we can upload a set of scaffold/contigs/genome we need to have uploaded the sequence libraries as BioSamples to a BioProject.
Our BioProject was already created - PRJEA61035 - which complicates things further

This means that you have to contact EBI and let them know, and then they do something on their end and that results in me creating a new BioProject and BioSample but referencing the previous BioSample.

# NCBI

We are not doing this for Hyphochytrium as a previous assembly, biosample and bioproject were already created via EBI. Therefore we need to do a not so easy update!

But we can upload the HiSeq data here - why not complicate it more eh? ;)

## 1. Upload Sequence Libraries
 * https://www.ncbi.nlm.nih.gov/sra/docs/submitportal/
 * for a start I can't login within Chrome. It works on FireFox though.
 * First you need to make a "template" for submission - https://submit.ncbi.nlm.nih.gov/biosample/template/
 * Then you need a sample sheet for SRA metadata.
 * Fill in all the information on the web-upload forms and the two .tsv files
 * Set a release date: 18 May 2017
 * Check for submission status here - http://trace.ncbi.nlm.nih.gov/Traces/sra_sub/sub.cgi?view=submissions

 * HiSEq: Submission: SUB1472106 and SRA: SRR3401404 and Sample: SAMN04868781

 * For some reason NCBI's email say that the SRA is SRP073437 http://www.ncbi.nlm.nih.gov/sra/SRP073437

#EBI

* I have been instructed to get ready an EMBL file format version of the data. Although their instructions [here](https://www.ebi.ac.uk/~anat/ENA_GENOME_ASSEMBLY_FILE_TYPES_TABLE.pdf) and [here](https://www.ebi.ac.uk/ena/submit/genomes-sequence-submission) contradicte each other!
* Neverthless it does need to be a "flat file" i.e. the EMBL format.
* Also it needs to be GZIPED. Which I found out only after going through all of the web upload and form filling. NOWHERE does it mention this little requirement. FFS.
* You will also need to prepare MD5 Sums of all your upload files.
* The other thing that is not mentioned ANYWHERE is that they automatically add the "protein_id" and "translation" qualifiers in the EMBL file from the 'gene' features. So you don't have to add proteins or nucleotides for you genes. That's it though, no CDS or anything else. OK.
* Yet another thing they don't mention is the "locus_tag" prefix that needs to be given to each "gene" or other entry in your EMBL file. A locus tag prefix is generated when you create a BioProject e.g. BN6634 for me. and it needs to be added belwo the "\gene" tag for each in your EMBL file. e.g \locus_tag="BN6634_XXX" where XXX is an incrementing number with padding - 001, 002..99x9 etc.

## 2. Genome Submission
 * I want to upload >1Kbp scaffolds + mito
 * https://www.ebi.ac.uk/ena/submit/sequence-submission
 * Create a new BioProject, create a new Sample (indicate in this sample it is a mix of this sample and another sample)
 * create assembly submissions (quietly weep inside at the pain this causes)

## Annotation Steps

We should really remap the default MAKER output names to something more readable/simple

### Create Map
    ~/maker/bin/maker_map_ids --prefix Hypho2016_ hyphochytrium_catenoides_genome_ge_1k_no_seqs.gff > hyphochytrium_catenoides_genome_map_ids.txt

### Remap Fasta and GFF3
    ~/maker/bin/map_fasta_ids hyphochytrium_catenoides_genome_map_ids.txt hyphochytrium_catenoides_predicted_proteins_renamed.fasta

    ~/maker/bin/map_gff_ids hyphochytrium_catenoides_genome_map_ids.txt hyphochytrium_catenoides_genome_ge_1k_no_seqs_renamed.gff

### Remove end of line ';' as they will break ANNIE
    sed -i 's/;$//g' hyphochytrium_catenoides_genome_ge_1k_no_seqs_renamed.gff

Either of:

### SwissProt BLASTp
    blastp -query hyphochytrium_catenoides_predicted_proteins_renamed.fasta -db uniprot_sprot.fasta -outfmt 6 -num_threads 24 -evalue 1e-10 -out all_maker_proteins_vs_swissprot_1e-10.tsv

### InterProScan
    interproscan.sh -f TSV,XML,GFF3 -goterms -i hyphochytrium_catenoides_predicted_proteins_renamed.fasta -pa | tee interproscan.log

## Annie

Again, either of:

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

I was going to use this:
* EMBOSS tool SEQRET

    seqret -sequence genome.fasta -feature -fformat gff -fopenfile genome.gff -osformat embl -auto

But it is not great.
* Therefore I am using [gff3toembl](https://github.com/sanger-pathogens/gff3toembl) instead! Hooray!
  * gff3toembl expects the gff3 file to have the fasta sequences at the end, so put them back after GAG!

* It is supposed to look something like [this](https://www.ebi.ac.uk/ena/submit/scaffold-flat-file)
  * However, it doesn't make it clear if the "XXX" marks should be filled in or left blank!
