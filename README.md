# PEDroid

This is the artifact for the paper *Automatically Extracting Patches from Android App Updates* submitted to ECOOP 2022.
It can be download at [here](https://github.com/huawanbibi/PEDroid/releases/download/master/pedroid.zip)


## Requirements

RAM > 16G


## Getting Started

An update is placed in `/dataset` including two Apk files. We take it as an example to run the artifact.

Notice that the input file must be named as `pkgname-x.x.x-yyy.apk`, especially, `x.x.x` represents its version.

1. extracting the compressed file


2. running dockerfile

   ```
   docker build -t ecoop/pedroid .
   docker run -it ecoop/pedroid /bin/bash
   ```

3. running differential analysis

   The directory `dataset` includes two Apk files as input.

   The `/pedroid/config.ini` stores configuration.

   The results will be output to `/home/example/diff_result`.

   ```
   cd /pedroid/diff
   pip3 install -r requirements.txt
   mkdir /home/example
   python3 app_diff.py -o /home/example/diff_result -c /pedroid/config.ini /dataset/
   ```

4. running patch identification

   ``` 
   cd /pedroid/patch
   python3 patch.py -c /pedroid/config.ini -o /home/example/patch_result /home/example/diff_result/
   ```


## Contents

The core program is as follows.
  - /pedroid/diff: python script and compiled files to diff two versions of an App.
    - app_diff.py: runs `baksmali`  to disassembly APK files and then calls the  python compiled files in DexDiff
    - DexDiff/: stores a set of compiled files. The files can be invoked to diff smali files of two verisons of an App. 
  - /pedroid/patch: python script, Java jar, and other binaries to identify patches.
    - patch.py: script used to call other binary files to implement the function of patch identification.
    - spotbugs-4.4.0/plugin/onlytaint.jar: a Java jar running as a plugin of spotbug that analyzes the call sites of similar methods (in Section 5.1).
    - difftaint.jar: a Java jar to compare the dependencies and generate the reports (in Section 5.2).


## Evaluation

- `Section 4 Differential Analysis` in the paper describes the design of the artifacts component in `/pedroid/diff`.

  It takes APK files as input and generates matching relations between two versions of an App. 

  - **Run diffing on dBench**
  
  The updates in dBench are stored under `/data/dBench`. The following command is used for analyzing all updates automatically (see 3 in Getting Started).

  ```
  python3 app_diff.py -c /pedroid/config.ini -o /home/diff_result /data/dBench
  ```

  The results of an update is saved in `<pkgname>/<version>/` in output directory.

  - **Result format**

  The results of an update generated by the component include a set of files. 

  ```
  result : overall results
  similar_orign.json : similar classes and methods
  new_smali_files : new classes in new version APK
  dele_smali_files : deleted classes in old version APK
  methodhash.db : signature and feature of each method
  record_time : time cost
  ...
  ```

  - **Data Involved in Evaluation**

  **To evaluate the effectiveness and accuracy of the component**, we ran it on the `dBench` and stored the results in the directory `/data/dBench_result/diff_result`. We evaluated the accuracy using the ground truth and compared the results with previous works, as Table 4 shows. It involves results generated by `SimiDroid` and `Androdiff`. We converted these data into a unified format to be parsed and placed them under `/data/compare/andro_result`, `/data/compare/simiDroid`. The data in Table 4 can be obtained by running the script `cmp.py` in `/data/compare/`.


    ```
    [816, 525, 291, 105, 133, 186, 16, 0.4411764705882353]
    [2111, 1550, 561, 138, 100, 423, 18, 0.5798319327731093]
    [429, 196, 234, 221, 17, 13, 13, 0.9285714285714286]
    ```

    To evaluate the generated results above (in **Run diffing on dBench**) rather than the pre-stored ones used in our paper, the command should indicate the path of the output:
    ```
    python3 cmp.py /home/diff_result
    ```

- `Section 5 Patch Identification` in the paper describes the design of the artifacts component in `/pedroid/patch`.

  It takes APK files and results of `diff` as input and then identifies whether the modified methods are patched. 

    - **Identify patches in dBench**

    Since the diffing results are under `/home/diff_result/`, the following commands are used for identifying patches based on these results:

    ``` 
    cd /pedroid/patch
    python3 patch.py -c /pedroid/config.ini -o /home/patch_result /home/diff_result/
    ```

  - **Result format**

  In the output directory, the file `SecurityPatch_7.txt` stores patch reports of each update. 
  
  In the file, the entry shown below means that the changes of the new version of the method with signature `<net.gsantner.opoc.util.ContextUtils: rstr(I)Ljava/lang/String;>` and old version of the method with signature `<net.gsantner.opoc.util.ContextUtils: rstr(I)Ljava/lang/String;>` contains a bug fix. 
  
  ```
  Potential security patches in:
          Src Signature <net.gsantner.opoc.util.ContextUtils: rstr(I)Ljava/lang/String;>
          Dst Signature <net.gsantner.opoc.util.ContextUtils: rstr(I)Ljava/lang/String;>
          Src tainted status : [0]
          Dst tainted status : [0]
          Src caller : <net.gsantner.opoc.util.ContextUtils: showDonateBitcoinRequest(IIII)V>
          Dst caller : <net.gsantner.opoc.util.ContextUtils: showDonateBitcoinRequest(IIII)V>
  ```
  
  - **Data Involved in Evaluation**
  
  **To evaluate the accuracy of patch identification**, we tested it on the dataset `dBench` with the results of diffing. The reports are pre-stored in the directory `/data/dBench_result/patch_report`. We manually checked whether the result corresponds to the commits in the appendix (Table 7 and Table 8) to determine whether the report is right. The details are discussed in Section 6.3.4.1.
  
  **To discuss the effect of each stage in the design,** the intermediate results are collected. It is described in Section 6.3.1 and can be obtained by running script `inter_result.py` in `/data`. The meaning of each line of numbers is as follows:
  
  ```
  iden  : identical packages
  r1    : similar packages matched by parent-child and sibling-sibling relationships 
  r2    : similar packages 
  identical : identical classes
  similar   : similar classes
  new       : new classes
  dele      : deleted classes
  new_cs      : call sites found in updated version
  new_method  : methods having call sites in updated version
  old_cs      : call sites found in original version
  old_method  : methods having call sites in original version
  patches     : patched methods
  ```
  
  Running the script `timecost.py` can collect the time cost and write results in a CSV file named `statistics.csv`. The meaning of each column is as follows:
  
  ```
  pkg: app_name
  version: version tag
  dissam: time cost of disassembly
  diff cost: time cost of differential analysis
  cscost: time cost of call site analysis
  incost: time cost of dependency comparison
  ```
  
  Similarly, the input can be set as generated results. Especially, the time cost may have some difference from Figure 5 because of different environments.

  ```
  python3 inter_result.py /home/diff_result /home/patch_result
  python3 timecost.py /home/diff_result /home/patch_result
  ```
