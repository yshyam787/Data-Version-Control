# DVC for automating the pipeline

## Introduction
Data Version Control (DVC) is a tool designed to track changes in the data, code, and model pipeline in a systematic way, similar to how Git tracks source code. It is generally used for managing and version controlling machine learning (ML) projects. It can integrate with the cloud storage providers like AWS S3, Google Drive, Owncloud, and so on. 

### How does DVC actually work?
DVC relies on Git because it manages code, scripts, and metadata in Git repositories. However, DVC manages large data files outside Git. DVC stores large files(datasets, models, etc.) separately  on cloud storage (Google Drive, AWS S3, Azure, Owncloud, etc) or on local storage while keeping small "pointers" (```.dvc``` files) or references in Git. The pointer files are committed to Git, and dvc handles the pushing and pulling the actual large data files on the choosen storage backend. Technically, DVC can work without Git, however, you will lose the tight integration (data versioning and tracking). In simple language, git tracks your code verison, DVC tracks data and model versions. Think Git as a table of contents for a book, and DVC acts like a delivery service for the external storage.  

### Other relevant questions for DVC
* Are there any hooks for tracking the chanages in Data or running the script after pulling the repo?
    - DVC does not provide Git-like hooks out of the box, but you can set up custom scripts to check for the changes in tracked files. ```dvc status``` checks if the data in the ```.dvc``` file has changed. 
    - You can also automate the runing scripts with a post-pull Git hook:
    ```echo -2 "!bin/bash\n\ndvc pull && dvc repro" > git/hooks/post-merge```
    ```chnod +x .git/hooks/post-merge```
* Does DVC run on Windows/macOS/linux?
    - Yes, DVC is a cross-platform and works on Windows, macOS, and Linux. Installation for all platform is ```pip install dvc```.

* Is DVC only command-line based or also has desktop?
    - DVC is primarily command-line tool. However, for visualization and collaborative features, you can integrate it with tools like Studio by Iterative. It offers a web-based UI for monitoring experiments. 

* What kind of hosting options are there for the DVC?
    - Cloud Storage(Amazon S3, Google Cloud Storage, One drive, etc.)
    - Remote Servers (via protocols): SSH, WebDAV, HTTP/HTTPS
    - Local or shared storage.
    - DVC Remote (DVC-specific hosting)

* How does DVC work with excel files?
    - DVC tracks te Excel files like any other binary files (e.g., ```.xlsx```). It doesn't parse or version the content inside the file. Instead, entire file is treated as a single objet. 

## Hands-on Project

### Set Up
* Install Git on your machine.
* Install DVC using pip: ```pip install dvc.```.
* Initialize Git repository: ```git init```.
* Initialize DVC in the repository: ```dvc init```.

### Configuring Owncloud as DVC Remote.
Since, owncloud provides access via WebDAV protocols, I obtained the URL from the setting and used my access credentials to set up DVC remote. 
    - ```dvc remote add -d owncloud_remote webdav://owncloud.gwdg.de/remote.php/nonshib-webdav```
    - ```dvc remote modify own_cloud_remote user <your-username>```
    - ```dvc remote modify own_cloud_remote password <your-password>```


### Creation of a Project:
* Make a new directory and initialize Git and DVC in it. 
```mkdir dvc_project```
```cd dvc_project```
```git init```
```dvc init```
* Add Some data to the directory. 
```echo "Sample data" > data.txt```
```dvc add data.txt```
    This creates a ```data.txt.dvc``` file.
* Commit the changes to the Git:
```git add .```
```git commit -m "Add data file tracked by DVC"```
* Push data to a repo
```dvc remote add -d myremote owncloud```
```dvc push```
* Add new data and push again.
```echo "New sample data" >> data.txt```
```dvc add data.txt```
```git commit -m "Update data"```
    Until this point, the things were working fine. However, as soon as I pushed some changes in the data to the owncloud using ```dvc push```, I got the error which states that I am unable to push the data the data to the owncloud. ![](https://pad.gwdg.de/uploads/eb953bad-94e0-4165-a265-34d493c48508.png)

> The above is not really correct. When using the browser to connect to that address, then you're trying to connect using HTTP/HTTPS. So the error message rightly says that that endpoint should be accessed by a WebDAV client
> 
> So it might be that DVC is already working with the previous setup, but when you went to check the stuff on the Owncloud side, you weren't using the proper link, which is simply: https://owncloud.gwdg.de
> [name=georgios.kaklamanos@gwdg.de]


Later on I realized that the WebDAV generally doesn't fully support advanced filesystem operations like ```.git``` repositories, which DVC relies on. Therefore, I created another separate directory called ```owncloud``` and mounted OwnCloud storage in it. This worked as my local DVC remote instead of directly using the WebDAVinterface. Finally, I was able to push the changes in my ```dvc_project``` to my local directory. 
```dvc remote add -d owncloud_remote <path-to-my-owncloud dirctory>```
```dvc push```
    Further information for accessing the owncloud files using WebDAV can found here:

    https://doc.owncloud.com/webui/next/classic_ui/files/access_webdav.html#accessing-files-with-kde-and-dolphin-file-manager
    
* Issues with pulling and pushing.
    
    Since, the ```owncloud``` directory on the machine is automatically synced with the actual web-based remote server, I can see the version tracking files in both of the local and remote files. However, the synchronization of OwnCloud with DVC for actual large data files can not be seen directly. It is because DVC and OwnCloud aren't natively integrated for two-way synchronization. Thus, we need to implement hybrid approach to achieve or goal. We can pull the changes from local ```owncloud``` directory to the ```dvc_project``` directory, push our contributions back the ```owncloud``` directory, and let the local owncloud directory synchronize with the web-based server.
    
    - Copy files from the Mounted OwnCloud Directory.
    ```rsync -av --delete ~/owncloud ~/dvc_project/owncloud_files/```
    ```dvc add owncloud_files.```
    ```git add owncloudfiles.dvc```
    ```git commit -m "Recent pulling of files"```
    ```dvc push```
    
> I'm actually also confused about this. To my understanding, the way you've set up DVC, then the local directory `~/dvc_project/owncloud_files/` and the `~/owncloud` directory should have a relationship of "local/remote". 
> Which means if there are any changes on the "remote" `~/owncloud`, if you go to the local directory `~/dvc_project/owncloud_files/`, and do something like `dvc pull`, then it should "download" the files from the "remote"
> [name=georgios.kaklamanos@gwdg.de]





