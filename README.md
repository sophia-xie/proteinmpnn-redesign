# ProteinMPNN Redesign
Implementation of the ProteinMPNN redesign approach by [Sumida et al.](https://pubs.acs.org/doi/10.1021/jacs.3c10941) on the SickKids HPC cluster.

### Steps:
1. Fix active site residues
2. Fix evolutionarily conserved residues
3. Redesign using ProteinMPNN
4. Validate using AlphaFold2

# TEV Protease
Control enzyme, from [Sumida et al.](https://pubs.acs.org/doi/10.1021/jacs.3c10941).

### Active Site Residues
Fix residues containing:
* **Backbone** atoms within **7** angstroms of the substrate, or
* **Sidechain** atoms within **6** angstroms of the substrate

### Evolutionarily Conserved Residues
Determined through Multiple Sequence Alignment (MSA) with four iterative [HHblits](https://toolkit.tuebingen.mpg.de/tools/hhblits) searches against the UniRef30 database. Final result filtered with [HHfilter](https://toolkit.tuebingen.mpg.de/tools/hhfilter).
1. HHBlits template sequence with E = **1e-50**
2. Forward to HHblits with E = **1e-30**
3. Forward to HHblits with E = **1e-10**
4. Forward to HHblits with E = **1e-4**
5. Copy Query MSA to HHfilter with Maximal Sequence Identity = **90%**, Minimal Sequence Identity = **30%**, Minimal Coverage = **50%**
6. Rank the residues by their most frequent amino acid frequencies
```
python scripts/get_conserved_residues.py <hhfilter_output_msa_fasta> <output_dir>
```

7. Extract the top percent% most conserved positions and create .jsonl for ProteinMPNN
```
python scripts/extract_top_percent.py <name> <chain> <conserved_resi.txt> <percent> <output_dir> <active_site_jsonl>
```

### ProteinMPNN

Fix active site and evolutionarily highly conserved residues. **Cysteine** was excluded from possible amino acids that could be designed. Three temperatures (**0.1**, **0.2**, **0.3**) were sampled.

* 8 x 3 = **24** sequences generated with **only active site** residues fixed
* 8 x 3 = **24** sequences generated with the active site and **30%** most highly conserved positions fixed
* 16 x 3 = **48** sequences generated with the active site and **50%** most highly conserved positions fixed
* 16 x 3 = **48** sequences generated with the active site and **70%** most highly conserved positions fixed

**Note:** The ProteinMPNN output file follows the name of the input PDB. All 1lvm PDBs below are identical, just copied and renamed to reflect the fixed positions of each run. Thus, the input PDB name MUST be unique.
```
sbatch scripts/proteinmpnn.sh <INPUT_PDB> <CHAINS_TO_DESIGN> <FIXED_POSITIONS_JSONL> <NUM_SEQS_PER_TARGET> <OUTPUT_DIR>

sbatch scripts/proteinmpnn.sh inputs/1lvm_active_only.pdb A inputs/active_only.jsonl 8 outputs/TEVd/
sbatch scripts/proteinmpnn.sh inputs/1lvm_top30.pdb A inputs/top30.jsonl 8 outputs/TEVd/
sbatch scripts/proteinmpnn.sh inputs/1lvm_top50.pdb A inputs/top50.jsonl 16 outputs/TEVd/
sbatch scripts/proteinmpnn.sh inputs/1lvm_top70.pdb A inputs/top70.jsonl 16 outputs/TEVd/
```

### Parse FASTA

ProteinMPNN outputs all sequences into a single FASTA file, but AlphaFold2 requires each sequence to be in a separate file with a unique basename. To parse a FASTA file into individual files for each sequence:

```
bash scripts/parse_fasta.sh <INPUT_FASTA> <OUTPUT_DIR>
```

Output FASTAs will be named according to input FASTA name, temperature, and sample number. One will be the original sequence.

### AlphaFold2

Model 3, 6 recycles. MSA of the parent sequence was used for all designs.
