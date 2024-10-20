### Using Service Account Method with DVC in GitHub CI/CD Pipeline

**Content**

1. [Setup Service account and Google drive folder](#setup-gcs-bucket-storage)
2. [Local Setup](#local-setup)
3. [Github Repo setup](#github-repo-setup)
4. [Github Actions](#github-actions)


### Setup Service account and Google drive folder

**Setup Service account and get json key**

In this method, we can store data in a Google Cloud Storage (GCS) bucket and fetch it using service account authentication.

To create a service account, navigate to IAM & Admin in the left sidebar, and select Service Accounts.

![service_account_icon](./assets/snap_tutorial_1.png)


Click + CREATE SERVICE ACCOUNT, enter a service account name (e.g., "My DVC Project"). If you are new and don't know what permissions to choose, it's better to give owner permissions.

![owner-permission](./assets/snap_tutorial_2.png)

Add all user accounts for which you need to grant access.

![email-access](./assets/snap_tutorial_3.png)

Then click CREATE AND CONTINUE. Click DONE, and you will be returned to the overview page.

Now you can see your service account; click on it and go to the Keys tab.

![serivce-mail-id](./assets/snap_tutorial_4.png)
 
Under Add Key, select Create New Key, choose JSON, and click CREATE.

![key-creation](./assets/snap_tutorial_5.png)
 
Download the generated projectname-xxxxxx.json key file to a safe location.

Important: Store the API key in a local folder as credentials.json, but do not commit it to GitHub. If you do so, GitHub will raise a warning, and Google will be notified, revoking the credentials. 

**Google drive folder**

Create a folder in your google drive. I have created a folder with name "dvc-storage-test"

![folder](./assets/snap_dvc_storage_test.png)

Important: Give permission to the folder `anyone with the link` with **editor** access. Also share with your service account for example this is my service account mail id "ajithvcodernew@devcmanager.iam.gserviceaccount.com" and give editor access. The folder in should be shared to specific users (or groups) so they can use it with DVC. "Anyone with a link" is not guaranteed to work.

![permission](./assets/snap_permission_gdrive.png)

Now get the id of the folder. For example this my folder url `https://drive.google.com/drive/folders/1b4v577_NGcEuUZK3WP6vZe5G9dBTGcsG`. So this is my gdrive folder id - `1b4v577_NGcEuUZK3WP6vZe5G9dBTGcsG` 

This is the configuration url i need to add to dvc config later `gdrive://1b4v577_NGcEuUZK3WP6vZe5G9dBTGcsG` i.e gdrive://<your_gdrive_folder_id>
Reference: [here](https://dvc.org/doc/user-guide/data-management/remote-storage/google-drive#url-format)


### Local Setup

Hereafter, in your local setup, you need to handle two things: the JSON file (dvcmanager-38xxxxxxx.json or any JSON with projectname-xxx.json which you downloaded as a key) and the Google Drive URL (gdrive://<your_gdrive_folder_id>).

- Create a folder named data locally.

- Copy the contents from this Kaggle dataset (https://www.kaggle.com/datasets/khushikhushikhushi/dog-breed-image-dataset) into the data folder and unzip it. Remove all files that are not needed (e.g., archive.zip is not needed after unzipping).

**Tree example**

|- data

|----dataset

|--------Beagle

|--------Boxer

|-------- etc folders

|-------- etc folders

- Install dvc and dvc-gdrive

```pip install dvc dvc-gdrive```

- Run git init (if you are not in a git folder already)

- Run dvc init


- Now run `dvc remote add -d myremote gdrive://<your_gdrive_folder_id> command`. Reference [here](https://dvc.org/doc/user-guide/data-management/remote-storage/google-drive#url-format)

eg:  ```dvc remote add -d myremote gdrive://1b4v577_NGcEuUZK3WP6vZe5G9dBTGcsG````

You will see that "myremote" has been added in the .dvc file.

- Run `dvc remote modify myremote gdrive_use_service_account true`

- Run `dvc remote modify myremote gdrive_acknowledge_abuse true`

- Run ```dvc remote modify myremote --local gdrive_service_account_json_file_path path/to/file.json```. i.e For example: ```dvc remote modify --local myremote gdrive_service_account_json_file_path devcmanager-385390fe7f4f.json```

You can see similar config in your `.dvc/config` file

![dvc config](./assets/snap_config_file.png)


- Run ```dvc add data```  i.e `dvc add <data_folder_name>

- Run ```dvc config core.autostage true``` (optional)

- Run ```dvc push -r myremote -v```

- Wait for about 10 minutes if it's around 800 MB of data for pushing; if it's in GitHub Actions, wait for 15 minutes.

Now you can check your Google Drive folder; you should see a folder named "files" like this:

![config_file](./assets/snap_files_store.png)

### Github Repo setup

- Now push this to your GitHub repository. Note that in the .dvc folder, by default, you can only push the "config" and ".gitignore" files. Don't change this; let it remain as is.

- Important: Never push the project-xxx.json file. If you do, Google will identify it and revoke the token; you'll need to set the key again.

- Add only .dvc/config, .gitignore, and data.dvc files.

- After pushing to repo, in github in your repo click on "Secrets and variables" -> "Actions" -> "Repository secret" in your GitHub repo and create a secret named "GDRIVE_CREDENTIALS_DATA" Copy the content of your project-xxx.json file (credentials.json file) into the content field.

![secrets](./assets/snap_add_secret.png)

- Now push this to github repo

Note in .dvc folder by default u can push only "config" and ".gitingore" file. Dont change it let it be like that.

- Kindly note it dont ever push "project-xxx.json" file. if you push google will identify it and revoke the token you need to set the key again.

- Add `.dvc/config`, `.gitignore`, `data.dvc` files alone.

- Click to "Secrets and variables" -> "Actions" -> "Reprository secret" in github repo and create a secret with name "GDRIVE_CREDENTIALS_DATA" and oopy the content of project-xxx.json file (credentials.json file) in the content.

### Github Actions

- Create a .github/workflows folder locally for setting up your GitHub Actions workflow.

- You can refer to the `dvc-pipeline.yml` file for complete content. 

Below is the code used to set up authentication and pull data inside GitHub CI/CD from Google Cloud Storage bucket:


```
      # Note you can also directly use "GDRIVE_CREDENTIALS_DATA" as env variable and pull it
      - name: Create credentials.json
        env:
          GDRIVE_CREDENTIALS_DATA: ${{ secrets.GDRIVE_CREDENTIALS_DATA }}
        run: |
          echo $GDRIVE_CREDENTIALS_DATA > credentials_1.json

      - name: Modify DVC Remote
        run: |
          dvc remote modify --local myremote gdrive_service_account_json_file_path credentials_1.json

      - name: DVC Pull Data
        run: |
          dvc pull -v
```

- Now you can trigger workflow by clicking "Run workflow" in github actions

![workflow-trigger](./assets/snap_workflow_trigger.png)

- You might see a error like this but its not a problem wait for sometime it is internally downloading files

![default_error](./assets/snap_error_default.png)

- After 5 minutes(Depending on the data size) you can see successfull run

![run_success](./assets/snap_run_success.png)

**Reference**

- Refered 1st point alone in "Using service account" in https://dvc.org/doc/user-guide/data-management/remote-storage/google-drive#using-service-accounts
- URL config setup - https://dvc.org/doc/user-guide/data-management/remote-storage/google-drive#url-format
- Using Service Account - GDrive - https://dvc.org/doc/user-guide/data-management/remote-storage/google-drive#using-service-accounts


