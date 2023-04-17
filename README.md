# VKGL Data release

This guide will explain every step in the VKGL Data release process. Please follow it step by step to guarantee
consistent output for every export.

## Prerequisites
- Access/permissions for:
  * Gearshift
  * Accounts umcg-vkgl-alissa, umcg-vkgl-lumc, and umcg-vkgl-radboud on nb-transfer.hpc.rug.nl
  * Install a [GUI](https://docs.gcc.rug.nl/nibbler/dedicated-dt-server-external-collaborators/) to access the three Nibbler accounts
  * ClinVar VKGL lab accounts
  * The downloadserver


## Step-by-step guide
### Preparation
1. Check if the acceptance server https://vkgl-acc.molgeniscloud.org/ is on and if not ask to turn if on
2. Request a backup from https://vkgl.molgeniscloud.org/ to be restored on the acceptance server
3. After the restore, change to password on the acceptance server back to the one in the Vault
4. Make sure you have correctly configured key [ssh key forwarding](http://docs.gcc.rug.nl/gearshift/datatransfers/#configure-ssh-agent-forwarding).
5. Ssh with SSH agent forwarding enabled to gearshift (use option `-A` argument) 
6. Go to `/groups/umcg-gcc/tmp01/projects/VKGL` and create a directory for this release (yyyymm).
7. Get the latest tag of [`data-transform-vkgl`](https://github.com/molgenis/data-transform-vkgl/tags)
   (every release, we create a new tag that uses the newest release in the helper scripts):

```commandline
wget https://github.com/molgenis/data-transform-vkgl/archive/refs/tags/*insert versionnumber*.zip
```
```commandline
unzip *insert versionnumber*.zip
```

8.Go to the `data-release-pipeline` folder of the folder you just unzipped and make sure you have execution permissions
   in the whole directory, by using f.e.: `ls -la`

### WORKAROUND
As long as the latest tag = 2.1.1 replace the run.sh, README.md and add data-files.txt (copy content) from data-transform-vkgl/data-release-pipeline/

### Download, preprocess, transform VKGL export data and generate a simple consensus file
9. Adjust the filenames in `data-files.txt` to fit the filenames of this export
10. Start `run.sh` in a separate screen to ensure that loss of connection won't stop the script from running. The `path/to/files/list` is optional,
    the default is the current directory.
This will take about four hours, depending on the number of elements in the request in the transformation step
```shell
screen
./run.sh --release yyyymm --files-list /path/to/files/list
```

   Detach your screen to make sure your internet connection won't mess up the script run, by pressing ctrl+a d
   In case of run-time errors check [FAQs.md](https://github.com/molgenis/molgenis-py-consensus/FAQs.md)

11. Return to the screen, if run.sh has finished, validate the output it generated by running the validation script:

```commandline
screen
ml PythonPlus
python validator/main.py /folder/created/by/run.sh/
```

12. If all checks are successful, continue, else rerun `run.sh` or pinpoint the issue.

### Install molgenis-py-consensus tool
13. Download the latest version (or the one you need) of molgenis-py-consensus
```commandline
wget https://github.com/molgenis/molgenis-py-consensus/archive/refs/tags/*insert versionnumber*.zip
```
```commandline
unzip 2.1.1.zip
```
OR
```commandline
git clone https://github.com/molgenis/molgenis-py-consensus.git
```

14. Create an input and output folder for this tool in the molgenis-py-consensus folder
15. Create a `config.txt` in the `molgenis-py-consensus/config` directory. Example:

```text
labs=umcg,umcu,nki,amc,vumc,radboud_mumc,lumc,erasmus
prefix=vkgl_
consensus=consensus
comments=consensus_comments
previous=1805,1810,1906,1910,1912,2003,2006,2009,2101,2104,2106,2109,2112,2209
history=consensus_history
input=place/to/store/input/for/molgenis-py-consensus/
output=place/to/store/output/for/molgenis-py-consensus/
```

Make sure to update the `previous`, `input` and `output`. The other values will (almost) always stay the same.
(You can use the previous config and put the latest previous export in there)

16. Copy the `vkgl`-lab files (from the `transformed` folder) to the input dir you specified in the config. 
17. Grant permissions to run the consensus scripts:

```
chmod g+x -R /path/to/molgenis-py-consensus
```
18. Change dir to molgenis-py-consensus
19. Setup and install molgenis-py-consensus

```commandline
python -m venv env
source env/bin/activate
pip install -e .
```

#### Add previous Consensus to the Consensus History
20. Download the current consensus and consensus comments into the input folder:
```commandline
python download/DownLoader.py -s https://*insert VKGL server* -u admin -t vkgl_consensus,vkgl_consensus_comments
```
21. Add YYYYMM of the previous run to these files:
```commandline
mv input/vkgl_consensus.tsv input/vkgl_consensus*YYYYMM-previous*.tsv
mv input/vkgl_consensus_comments.tsv input/vkgl_consensus_comments*YYYYMM-previous*.tsv
```
22. Run the preprocessor (adds comments and ID including lab to the files):
```commandline
python preprocessing/PreProcessor.py
```
23. Run the history writer:
```commandline
python preprocessing/HistoryWriter.py
```
24. Make sure you first deactivate your env and purge your pythonPlus and load Python before installing the commander:
    ```commandline
    deactivate
    module purge PythonPlus
    module load Python
    ```
25. Install the commander (if not done yet), following
    [this guide](https://github.com/molgenis/molgenis-tools-commander/wiki/Installation-guide). 
26. Paste [these files](https://github.com/molgenis/molgenis-py-consensus/tree/master/mcmd_scripts) in `.mcmd/scripts` in your home folder (/home/umcg-USERNAME)
27. Set `import_action` in the `settings` section of `mcmd.yaml` to `add`
28. Add the `output` folder of `molgenis-py-consensus` to `dataset_folders` in  `mcmd.yaml`
29. Add the production and test server as hosts to the commander using the `mcmd config add host` command.
30. Configure molgenis commander host to acceptance server.

``` commandline
mcmd config set host
```
31. Import the consensus history into the acceptance server

```commandline
mcmd import vkgl_consensus_history.tsv
```
32. Check the acceptance server to make sure the history is uploaded. There should be variants from the previous export
    available in the table (might need to wait until indexing is done). 
    If it looks okay, change the mcmd host to production and run the same command.
``` commandline
mcmd config set host
mcmd import vkgl_consensus_history.tsv
```
33. Purge the Python module, load PythonPlus again and activate the virtual env again:

```commandline
module purge Python
ml PythonPlus
source env/bin/activate
```

34. Wait till indexing is done on the production server (about 30 minutes) and then download the complete consensus history into the input folder of the `molgenis-py-consensus`:
```commandline
python download/DownLoader.py -s https://*insert VKGL server* -u admin -t vkgl_consensus_history
```

#### Create the Consensus
35. Make sure you still have a screen running (`screen -ls`). Then run the consensus script:

```commandline
python consensus
```
36. The script takes about 36 hours to run (every export it will take longer), so detach your screen to make sure your 
internet connection won't mess up the script run, by pressing `ctrl+a d`.
37. Check every now and then if your consensus script is still running:

```commandline
screen -r
```

If everything is okay, press `ctrl+a d` again. Now all we need to do is wait until the script is done. 
In the meantime you could start with the ClinVar submissions (see below). After the script is done,
type `deactivate` to deactivate the virtual environment.

### Upload VKGL consensus and lab data into the server
38. Make sure you purge the PythonPlus module again and load Python to run the commander.
```commandline
    module purge PythonPlus
    module load Python
```
39. Change the mcmd host to the acceptance server again. Test if the output works by uploading it to the acceptance server:

```commandline
mcmd run vkgl_cleanup_consensus
mcmd run vkgl_cleanup_labs
mcmd delete --data vkgl_comments -f
mcmd import vkgl_comments.tsv
mcmd run vkgl_import_labs
mcmd import vkgl_consensus_comments.tsv
mcmd import vkgl_consensus.tsv
mcmd delete --data vkgl_public_consensus -f
mcmd import vkgl_public_consensus.tsv
```

If you get a `504` error throughout this process. You can start a mcmd script from a certain line using
the `--from-line` command. Keep retrying until you don't get the `504` anymore. Trust me, it will work.
40. Time to upload to production. Set a message on the homepage by editing the `home` row in `sys_StaticContent` (don't do
    it via the home page, it will mess up everything!):

```html

<div class="alert alert-warning" role="alert">
    We are currently working on the new VKGL data release, this means some data might be missing or incorrect. As soon
    as
    this message is no longer on our homepage, the release is updated and save to use. We thank you for your
    understanding
    and patience.
</div>
```
41. Set the mcmd host to production and run the following commands on production:

```commandline
mcmd run vkgl_cleanup_consensus
mcmd run vkgl_cleanup_labs
mcmd delete --data vkgl_comments -f
mcmd import vkgl_comments.tsv
mcmd run vkgl_import_labs
mcmd import vkgl_consensus_comments.tsv
mcmd import vkgl_consensus.tsv
mcmd delete --data vkgl_public_consensus -f
mcmd import vkgl_public_consensus.tsv
```
42. Update the counts page by editing the `news` row in the `sys_StaticContent` table and paste the `vkgl_counts.html` that
    was produced by `molgenis-py-consensus`.

43. Upload the public consensus to the download server:
    a. Get the data locally: ```rsync -av airlock+gearshift:/groups/umcg-gcc/tmp01/projects/VKGL/*YYYYMM*/molgenis-py-consensus/*outputfolder*/vkgl_public_consensus.tsv .```
    b. Put the data on the download server: ```scp vkgl_public_consensus.tsv molgenis@10.10.1.130:/data/downloads/VKGL/VKGL_public_consensus_*YYYY-MM*.tsv```
44. Update the downloads page by editing the `background` row in the `sys_StaticContent`
45. Update the name of the export in the menu.
46. Remove the message from the homepage.
47. Ask the Team Lead of the RareDisease-team to email the contact persons for acceptance testing.

### Create VIP files
48. Make a `vip` directory in your VKGL release directory.
49. Go to `data-transform-vkgl*insert versionnumber*/data-release-pipeline/utils` and generate the gene id's for the consensus file:

```
./vkgl_consensus_add_gene_id.sh -i /path/to/molgenis-py-consensus/output/vkgl_consensus.tsv -o /path/to/vipdir/vkgl_consensus.tsv -g ../datetimeoftransformeddata/downloads/hgnc_genes_YYYYMMDD.tsv
./vkgl_consensus_add_gene_id.sh -i /path/to/molgenis-py-consensus/vkgl_public_consensus.tsv -o /path/to/vipdir/vkgl_public_consensus.tsv -g ../datetimeoftransformeddata/downloads/hgnc_genes_YYYYMMDD.tsv
```

50. Get the vcf-tsv converter in the utils folder of data-release-pipeline:

```commandline
wget https://github.com/molgenis/tsv-vcf-converter/releases/download/vx.y.z/tsv-vcf-converter.jar
```

51. Do the liftover for the vip data:

```
./liftover_vkgl_consensus.sh /path/to/vipdir/vkgl_public_consensus.tsv /path/to/vipdir/vkgl_public_consensus_hg38.tsv
./liftover_vkgl_consensus.sh /path/to/vipdir/vkgl_consensus.tsv /path/to/vipdir/vkgl_consensus_hg38.tsv
```
52. Notify VIP-team that the data is available on `/groups/umcg-gcc/tmp01/projects/VKGL/YYYYMM/vip/`


### Persist data on the `Gearshift` cluster prm03 folder
53. Go to `/groups/umcg-gcc/prm03/projects/VKGL/`
54. To be able to create folders and put data in this folder do: `sudo -u umcg-gcc-dm bash`
55. Create a new folder with a name like `yyyymm`.
56. In this directory make the following folders:

- clinvar
- molgenis
- raw
- data-transform
- vip

57. Fill the raw folder by copying all raw data you got from Radboud/MUMC, LUMC and all Alissa data from the data-transform-vkgl-*version*/data-release-pipeline/datetimeoftransformeddata/data/ folder 
58. If the files are not yet compressed, compress them (f.e. by doing `tar -czf alissa_YYYYMM.tar.gz vkgl*`)
59. Fill the data-transform folder by copying all data from the data-transform-vkgl*version*/data-release-pipeline/datetimeoftransformeddata
    using cp -r to it
60. In the molgenis folder create an input and an output folder.
61. In the input directory place all content of the input folder of molgenis-py-consensus.
62. In the output directory place all content of the output folder of molgenis-py-consensus.
63. Fill the `vip` folder with all data in the `vip` folder on tmp.

64. Zip files/folders to decrease storage space.

65. Create a `versions.txt` in `/groups/umcg-gcc/prm03/projects/VKGL/yyyymm/` with the following content:
    ```text
    DATA:
    Alissa: yyyymmdd
    Radboud/MUMC: yyyy-mm-dd
    LUMC: yyyy-mm-dd
    HGNC genes file: yyyymmdd

    Scripts:
    data-transform-vkgl and run.sh: data-release-vx.y.z
    tsv-vcf-converter: vx.y.z
    vkgl-clinvar: vx.y.z
    molgenis-py-consensus: x.y.z
    molgenis-tools-emx-downloader: x.y.z
    molgenis-tools-commander: vx.y.z
    ```
    Fill in the versions used for this export.

### Finishing up
66. Once the VKGL-release is accepted by the contact persons. Send the email notifying everyone that the VKGL release is done and the ClinVar submission is in progress.
67. Create a `consensus` folder in the Radboud directory on nibbler (using FileZilla). Place the vkgl_consensus file as generated by  
    `molgenis-py-consensus` in there and notify the Radboud/MUMC contact person.
68. Send the error files to all labs (contactpersons can be found [here](https://vkgl.molgeniscloud.org/menu/main/dataexplorer?entity=vkgl_public_contacts&hideselect=true))
69. Clean up files on the tmp folder


### ClinVar submissions
1. Create the ClinVar files. To do this, get the most recent version (or another
    version, if you wish) of [vkgl-clinvar](https://github.com/molgenis/vkgl-clinvar).

```
wget https://github.com/molgenis/vkgl-clinvar/releases/download/*vx.y.z*/vkgl-clinvar-writer.jar
```


2. Go to ClinVar, select `Submit` in the dropdown and click `Submission portal` (this way you will be redirected to the
    correct overview). Download `SUB*submission id*_(100)_submitter_report_B.txt` of the previous ClinVar submits for
    all labs.

3. Create an output folder for your ClinVar submit files.
4. Run the script:

``` commandline
java -jar vkgl-clinvar-writer.jar -i /output/of/runs.sh/consensus/consensus.tsv -m /path/to/files/from/previous/clinvar/submit/*export name*_DUPLICATED_identifiers.tsv,/path/to/files/from/previous/clinvar/submit/*export name*_REMOVED_identifiers.tsv,/path/to/files/from/previous/clinvar/submit/*export name*_UNCHANGED_identifiers.tsv,/path/to/files/from/previous/clinvar/submit/*export name*_UPDATED_identifiers.tsv -c amc=/path/to/SUB*submission id*_(100)_submitter_report_B.txt,lumc=/path/to/SUB*submission id*_(100)_submitter_report_B.txt,nki=/path/to/SUB*submission id*_(100)_submitter_report_B.txt,umcg=/path/to/SUB*submission id*_(100)_submitter_report_B.txt,radboud_mumc=/path/to/SUB*submission id*_(100)_submitter_report_B.txt,umcu=/path/to/SUB*submission id*_(100)_submitter_report_B.txt,vumc=/path/to/SUB*submission id*_(100)_submitter_report_B.txt,erasmus=/path/to/SUB*submission id*_(100)_submitter_report_B.txt -o /output/folder/for/clinvar/data -r mon_yyyy -f
```
5. Once accepted, do the ClinVar submissions. [Need help?](https://github.com/molgenis/vkgl-clinvar)

6. Save the data on prm03 folder:
7. 62. Go to the ClinVar folder and make a `generated` and `reports` folder in there.
8. Copy all content of the folder with the output of the vkgl-clinvar-writer to the generated folder.
9. When ClinVar is done evaluating, go to the submission portal and download all reports and place them in the reports
   folder. Prefix them with the correct lab name.


 
