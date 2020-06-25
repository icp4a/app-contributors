# Integrating Contributions with Application Designer


### Delivering your contribution

Now that you developed and exported your contribution as a .twx file, you must deliver your toolkit to your users. 
In order for users to use your contribution in their Business Automation Studio out-of-the-box, you need to be sure that your contribution is recognized and synchronized in the studio.

The delivery method consists of two main steps.
1. **Packaging**. You must package your toolkit file with your pillar product.
2. **Making your contribution discoverable by the studio**. 

With the packaging step, it is possible to either provide instructions to the end user to manually import the toolkit file into their copy of Application Designer, or to perform an automatic import step instead. In this section, we are mainly focused on explaining the available techniques for the auto import.

### Toolkit push

The Application Designer has a toolkit import REST API endpoint that you can directly invoke from your installer script as part of the deployment. This script will need the end point url and administrator credentials to do the import.
In an ICP environment install of Application Designer, the *contribution installer* job is created by one of the operator tasks and thus has access to the required credentials.  
All you need to do is to upload your toolkit to the toolkits folder in the [contribution installer repository](https://github.ibm.com/dba/contribution_installer).  
Continuous delivery will update the installer image and run it against the Application Designer in the ICP framework.

### Toolkit pull

In this technique, the contributing pillar will advertise, to the Application Designer, the location of a toolkit file and the Application Designer will attempt to automatically pull the file from that location and import it in the background.  
The file location needs to be accessible by the Application Designer.  

The contributor advertises the toolkit by writing a *contribution* entry into the Resource Registry. This entry can be written by the application itself, or by one of its installer tasks using a bash script to call into the Resource Registry. 

Note: Since the Application Designer is always listening for updates regarding the contribution entries in the Resource Registry, you need to make sure that the file is ready to be discovered before writing the entry.  
This is required in order to prevent a **failed to import, file not found** message in the logs.   
Application Designer will attempt to re-download the contribution files when it is restarted.

There are 2 methods to make a contribution file available to Application Designer:

1. Mounted folder (persistent volume).  
  In this method, the pillar copies the file to a persistent volume. Application Designer image should mount this persistent volume as a folder in its file system. For example, `/mnt/toolkits_folder`

2. Download URL.  
The contributor should ensure that Application Designer is able to access that download URL without networking issues, like invalid certificates, invalid credentials, or invalid egress definition in the ICP environment. The contributor will also be responsible for setting up the required file server service as part of their application or as an independent service. 


![Sharing and synchronized contributions](./images/14.png)

The illustration above outlines the interaction between Business Automation Studio, the service registry, and a persistent volume.


**Note** The previous steps assume that your pillar is deployed by using K8s and has access to the required Resource Registry end point and credentials. If your pillar runs outside K8s, your pillar will need to be able to connect to the Resource Registry to write the contribution entries. This will require adding knowledge about the Registry end point and necessary credentials for writing keys.

The following JSON sample illustrates an actual registry entry that was added to the Resource Registry by a BAW pillar. Note that it has 2 contribution toolkits defined.
 

```json
{
  "key": "/dba/appstudio/contributors/IBM_WORKFLOW/<pillar_hostname>",
  "value": {
            "actionConfiguratorInfo": {
                "protocol": "https",
                "hostname": "<pillar_hostname>",
                "port": "<pillar_port>"
            },
            "contributionInfos": [
                {
                    "filename": "Workflow_Services - v1.0.0.2.twx",
                    "filepath": "/mnt/<persistent_volume>/Workflow_Services - v1.0.0.2.twx",
                    "name": "Workflow Services",
                    "id": "2066.be80b56f-6638-4cb7-aa2d-1252d7148010",
                    "type": "TWX",
                    "version": "2064.913f2fad-5f86-4b6d-b498-01964bb03f7e",
                    "url": "https://<pillar_hostname>:<pillar_port>/rest/bpm/wle/v1/resourceregistry/contributions/2066.be80b56f-6638-4cb7-aa2d-1252d7148010/download"
                },
                {
                    "filename": "Start_Process_Configurator - v1.3.4.twx",
                    "filepath": "/mnt/<persistent_volume>/Start_Process_Configurator - v1.3.4.twx",
                    "name": "Start Process Configurator",
                    "id": "2066.1ddaefd6-6d52-48e8-adb7-57f5b4d6c705",
                    "type": "TWX",
                    "version": "2064.0cd733ad-9c2f-4973-975d-323e989998c9",
                    "url": "https://https://<pillar_hostname>:<pillar_port>/rest/bpm/wle/v1/resourceregistry/contributions/2066.1ddaefd6-6d52-48e8-adb7-57f5b4d6c705/download"
                }
            ]
        }
    }
```
**Sample code notes**:
- ***`url`*** and ***`filepath`*** are mutually exclusive. You need to define only one of them.  
***`filename`*** is ignored and is assumed to be included as part of ***`filepath`***.
If you define ***`filepath`***, then Application Designer will try to copy the file from that path assuming that the path is mounted and accessible.
On the other hand, if ***`url`*** is defined, then Application Designer will attempt to download the file.

- A pillar lists all of its contributions in the service registry. 
- Key: /dba/appStudio/contributor/IBM_WORKFLOW/CreditWF  

|/dba/appStudio/contributor|IBM_WORKFLOW|CreditWF|  
|--------------------------|--------|--------|  
| Namespace | Type | Unique ID |

- The studio only supports one copy of a toolkit of a particular version of the same unique ID. You cannot have the same toolkit of the same type that is contributed from multiple pillars if the unique IDs are the same.

- If multiple servers attempt to contirbute the same toolkit or application, then this is a valid scenario. The Business Application Studio just won't import the other copies.

- When the studio is started, it queries the "/dba/appStudio/contributor" namespace in the registry. The studio checks the registry and retrieves your contribution information. After starting, the studio continues to watch for all updates. 

- By using the information provided in the registry, the studio reads the toolkit files from the persistent volume. The studio will also compare existing contributions to ensure that the most recent version of a toolkit is being used. 


For file download, Application Designer will look up the environment variable `TK_DOWNLOAD_CRED` and if it finds it, it will add it as a basic authorization header for the download `GET` request.
```
export TK_DOWNLOAD_CRED=dHdfYWRtaW46dHdfYWRtaW4=
```
```
Authorization: Basic dHdfYWRtaW46dHdfYWRtaW4=
```
This environment variable should be set by the continuous delivery operator task that install the Application Designer.



