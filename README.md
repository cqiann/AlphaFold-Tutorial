# Quick AlphaFold Introduction
Proteins are macromolecules comprised of one or more chains of amino acids. They are essential to life as they perform a variety of functions, including catalyzing reactions, involving in DNA replication, etc... 

AlphaFold is an AI system developed by Google DeepMind that predicts a protein's 3D structure from its amino acid sequence.

# Running AlphaFold Simulations
To run an AlphaFold simulation, you need to first connect to one of the shared RCC clusters through SSH. There are multiple shared clusters:
| Host name   | SSH host address               
| ----------- | -------------------------------
| Midway2     | `midway2.rcc.uchicago.edu`
| Midway3     | `midway3.rcc.uchicago.edu`
| Midway3-AMD | `midway3-amd.rcc.uchicago.edu`
| DaLI        | `dali-login.rcc.uchicago.edu`

Open a Terminal window ("Terminal" app for Mac and "PowerShell for Windows) and enter the command (ignore the <> when you type):

`ssh <your RCC username>@<host username>`

Your `RCC username` should just be your `CNETID` and you can use can one of the RCC cluster host addresses as the `host username`. I recommend using `Midway2` or `Midway3`.

For example, if I were to SSH into Midway3, I type in `ssh christineqian@midway3.rcc.uchicago.edu` and press `enter` on Windows or `return` on Apple keyboards. Press `enter` or `return` after each command you type into the terminal window.

If this is your first time signing into an RCC cluster, SSH will ask you `Are you sure you want to continue connecting?`. Type `yes` and then press `enter` or `return` on your keyboard to proceed.

Then, you will be asked for a `password`, which should just be your `CNETID password`. Type in your password and then press `enter` or `return`. (Note: You won't see any characters as you type in your password. That's alright!)

Then, you will be asked for a DUO two-factor authentication. Go ahead and complete the TFA. After completing the TFA, you will see `Success. Logging you in...` in the Terminal window, and you're in.

Now, you will go to your `scratch` directory in midway. Whenever you want to go into a directory, you will use the command `cd` followed by the name or path of the directory. In your terminal, type in the command:

`cd /scratch/<the RCC cluster you chose>/<your CNETID>`

For example, I chose `Midway3`, so I will type `cd /scratch/midway3/christineqian`.

Then, you will create a new directory with the command `mkdir`, followed by the name you want to give to the directory. You can name the directory however you want. For this activity, I'll go with the name `alphafold`. To create a directory called "alphafold", the complete command is:

`mkdir alphafold`

Then, go into the directory you just created, again with the command `cd` and the name of the directory. (`cd alphafold` in my case). 

## Create a FASTA file

You will provide a fasta file containing your amino acid sequence when running AlphaFold. A sequence in FASTA format begins with a single line description marked by a '>' followed by lines of the actual amino acid sequence. The general command to create a fasta file is:

`vim <name of protein>.fasta`

You'll see an empty file named `<name of protein>.fasta`. To edit the file, press `a` on your keyboard.

In this activity, we will predict the structure of amyloid beta (ABeta) oligomers. There are two major forms of ABeta, __40-residue__ (Abeta40) and __42-residue__ (Abeta42). 

The Abeta40 __monomer__ fasta file contains:

```
>2M4J_1|Chains A, B, C, D, E, F, G, H, I|Amyloid beta A4 protein|Homo sapiens (9606)
DAEFRHDSGYEVHHQKLVFFAEDVGSNKGAIIGLMVGGVV
```

The Abeta42 __monomer__ fasta file contains:

```
>2MXU_1|Chains A, B, C, D, E, F, G, H, I, J, K, L|Amyloid beta A4 protein|Homo sapiens (9606)
DAEFRHDSGYEVHHQKLVFFAEDVGSNKGAIIGLMVGGVVIA
```

For multimers, simply copy and paste the corresponding Abeta monomer fasta file lines to the desired number (For example, if I were to run an Abeta42 trimer, I'll just have three of the Abeta42 monomer description line + sequence).

__For this activity, we will run and analyze 42r multimers because they are more toxic.__

After you're done editing the fasta file, press `esc` on your keyboard to return to normal mode (instead of editing/appending), and then save the file by typing in `:wq` and pressing `enter`.

## Create the AlphaFold submission script

Create an AlphaFold submission script with the command:

`vim alphafold2.3.2-submit.sh`

You'll see an empty file named `alphafold2.3.2-submit.sh`. To edit the file, press `a` on your keyboard. You can copy and paste the following into the file:

```
#!/bin/bash
#SBATCH --job-name=alphafold2
#SBATCH --account=workshop-aiml
#SBATCH --partition=gpu
#SBATCH --nodes=1
#SBATCH --time=10:00:00
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=16
#SBATCH --gres=gpu:2
#SBATCH --mem=64G

module load alphafold/2.3.2 cuda/11.3

# shamelessley ripped from https://github.com/kalininalab/alphafold_non_docker/blob/main/run_alphafold.sh
usage() {
        echo ""
        echo "Please make sure all required parameters are given"
        echo "Usage: $0 <OPTIONS>"
        echo "Required Parameters:"
        echo "-f <fasta_paths>      Path to FASTA files containing sequence(s). If a FASTA file contains multiple sequences, then it will be folded as a multimer. To fold more sequences one after another, write the files separated by a comma"
        echo "Optional Parameters:"
        echo "-t <max_template_date>    Maximum template release date to consider (ISO-8601 format - i.e. YYYY-MM-DD). Important if folding historical test sets (Default: 2022-1-1)"
        echo "-p <use_precomputed_msas> Denotes if the program should use MSAs that have already been generated by a previous run. (Default: false)"
	echo "-o <output_dir>           Path to a directory that will store the results. (Default: current directory)"
        echo ""
        exit 1
}

# Default Parameters
max_template_date="1000-02-20"
use_precomputed_msas=true
DOWNLOAD_DATA_DIR=/software/alphafold-data-2.3

while getopts ":d:o:f:t:g:r:e:n:a:m:c:p:l:b:" i; do
        case "${i}" in
        d)
                data_dir=$OPTARG
        ;;
        o)
                output_dir=$(realpath $OPTARG)
        ;;
        f)
                fasta_path=$(realpath $OPTARG)
        ;;
        t)
                max_template_date=$OPTARG
        ;;
        p)
                use_precomputed_msas=$OPTARG
        ;;
	m)
                model_preset=$OPTARG
        ;;
        esac
done

# Parse input and set defaults
if [[ "$fasta_path" == "" ]] ; then
    usage
fi

if [[ "$output_dir" == "" ]] ; then
    output_dir="."
fi

if [[ "$data_dir" != "" ]] ; then
    DOWNLOAD_DATA_DIR=$data_dir
fi

if [[ "$model_preset" != "monomer" && "$model_preset" != "monomer_casp14" && "$model_preset" != "monomer_ptm" && "$model_preset" != "multimer" ]] ; then
    echo "Unknown model preset! Using default ('mutimer')"
    model_preset="multimer"
fi

# Parse database paths which are different for monomer / multimer
if [[ $model_preset == "multimer" ]]; then
	database_paths="--uniprot_database_path=$DOWNLOAD_DATA_DIR/uniprot/uniprot_sprot.fasta --pdb_seqres_database_path=$DOWNLOAD_DATA_DIR/pdb_seqres/pdb_seqres.txt"
else
	database_paths="--pdb70_database_path=$DOWNLOAD_DATA_DIR/pdb70/pdb70"
fi

echo "Running Alphafold with arguments: model=${model_preset}, use_precomputed_msas=${use_precomputed_msas}, max_template_date=${max_template_date}, fasta_file=${fasta_path}"

python /software/alphafold-2.3.2-el8-x86_64/run_alphafold.py  \
  --data_dir=$DOWNLOAD_DATA_DIR  \
  $database_paths \
  --uniref90_database_path=$DOWNLOAD_DATA_DIR/uniref90/uniref90.fasta  \
  --mgnify_database_path=$DOWNLOAD_DATA_DIR/mgnify/mgy_clusters_2022_05.fa  \
  --bfd_database_path=$DOWNLOAD_DATA_DIR/bfd/bfd_metaclust_clu_complete_id30_c90_final_seq.sorted_opt  \
  --uniref30_database_path=$DOWNLOAD_DATA_DIR/uniref30/UniRef30_2021_03 \
  --template_mmcif_dir=$DOWNLOAD_DATA_DIR/pdb_mmcif/mmcif_files  \
  --obsolete_pdbs_path=$DOWNLOAD_DATA_DIR/pdb_mmcif/obsolete.dat \
  --model_preset=$model_preset \
  --max_template_date=${max_template_date} \
  --db_preset=full_dbs \
  --use_gpu_relax=true \
  --models_to_relax=all \
  --use_precomputed_msas=${use_precomputed_msas} \
  --output_dir=$output_dir \
  --fasta_paths=$fasta_path
  ```

After pasting in the above code (or anytime you're done editing the file), press `esc` on your keyboard to return to normal mode (instead of editing/appending), and then save the file by typing in `:wq` and pressing `enter`.

Now that you have both the fasta file containing your sequences and the AlphaFold script, run the following command in your terminal to submit a job:

`sbatch alphafold2.3.2-submit.sh -f <your fasta>.fasta -o .`

To check the progress of the job you have submitted, run the following command:

`squeue --user=<CNetID>`

When AlphaFold finishes running, you can find the outputs in a directory named \<name of your fasta file\> in your current directory. 

You can check the outputs by `cd`ing into that directory and then typing `ls`. (Anytime you want to check the files contained in a directory, you can `cd` into that directory and use the command `ls`).

The output files you want end in `.pdb` format, AlphaFold ranks the structures based on confidence. The one with the best confidence is named `ranked_0.pdb`.
You can load those .pdb files in VMD to visualize their 3D structures.  111

## AlphaFold Parameters
There are several parameters you have to provide and/or can modify when running an AlphaFold simulation. They are discussed in details below.

### Running AlphaFold with or without template

You have the choice to run your AlphaFold simulation with or without template by modifying the `max_template_date` parameter in line 30 of the script.

AlphaFold will search for available template structures from the Protein Data Bank (PDB) that are similar in sequence to the sequence you provide __BEFORE__ the date specified by the `--max_template_date` parameter.

Generally, if you want the predict the structure of a protein that has not been experimentally solved (ABeta multimer for example), it is recommended to run the simulation __without__ templates. If the protein structure is known and providing templates from PDB would benefit the prediction of the structure, you should provide templates.

In the script you have, the default max_template_date is `"1000-02-20"`, which means the default is to run the simulation __without__ any templates.

If you want to run the simulation __with__ templates, the convention is to set the `max_template-date` to __today's date__ To make the modification, type in the command `vim alphafold2.3.2-submit.sh` again, go into edit mode by typing `a` on the keyboard, modify the date in line 30, exit out of edit mode with `esc`, and saving the change with `:wq`.

### Model Preset
You can run simulations on both monomers and multimers. The default in your script is to run it as a __multimer__. 

If you want to run your fasta file as a monomer, you would run the command:

`sbatch alphafold2.3.2-submit.sh -f <your fasta>.fasta -m monomer -o .`

### Running AlphaFold with precomputed MSAs
The most time-consuming step of the AlphaFold job is to create the MSAs (Multiple Sequence Alignments). To speed up the run time for this activity, I will provide you the MSAs generated from the previous jobs that I ran. 

To run your jobs with precomputed MSAs, you need to make sure the `msa/` directory generated by a previous run is in your AlphaFold output directory, and set the parameter `--use_precomputed_msas` to `true`. The protein used in the previous run needs be identical to the one used in the current run. I have already set the default of `--use_precomputed_msas` to `true` for you in the script, so you don't need to modify the script yourself. In the `alphafold` directory you created, simply run the following commands to submit an AlphaFold job with precomputed MSAs:

```
mkdir <name of your fasta file>
cp /scratch/midway3/christineqian/Haddadian-Lab-Docs/alphafold-scripts/Abeta/42r.<number of chains in the Abeta that you are running>c/msas ./<name of your fasta file>
sbatch alphafold2.3.2-submit.sh -f <name of your fasta file>.fasta -o <name of your fasta file>
```

For example, if you want to run a 42-residue trimer, and you made a fasta file `42r-trimer.fasta`, you can run the following commands:

```
mkdir 42r-trimer
cp -r /scratch/midway3/christineqian/Haddadian-Lab-Docs/alphafold-scripts/Abeta/42r.3c/msas ./42r-trimer
sbatch alphafold2.3.2-submit.sh -f 42r-trimer.fasta -o 42r-trimer
```

# Obtaining a local copy of your AlphaFold output

Open a new terminal window and run the command:

`scp <your CNETID>@<your RCC cluster>.rcc.uchicago.edu:<the path to your alphafold output> .`

You can obtain the path to your output by going into the directory that contains your output and running the command `pwd`. Then, simply copy that path and then add `\<your output file>`, where `your output file` is going to be `ranked_x.pdb`.

# References
Jumper, J. et al. “Highly accurate protein structure prediction with AlphaFold.” Nature, 596, pages 583–589 (2021). DOI: 10.1038/s41586-021-03819-2

Viles, John H. “Imaging Amyloid-β Membrane Interactions: Ion-Channel Pores and Lipid-Bilayer Permeability in Alzheimer's Disease.” Angewandte Chemie (International ed. in English) vol. 62,25 (2023): e202215785. doi:10.1002/anie.202215785

https://rcc-uchicago.github.io/user-guide/ssh/main/

https://rcc-uchicago.github.io/user-guide/software/apps-and-envs/alphafold/?h=alpha
