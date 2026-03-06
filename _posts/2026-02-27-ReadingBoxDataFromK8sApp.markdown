---
layout: post
title:  "Reading Box Data from a Kubernetes App"
date:   2026-02-27 15:55:00 +0600
categories: kubernetes
author: "BenGalewsky"
---
We often need to deploy custom web apps in Kubernetes that depend on datasets that don't fit
in GitHub or inside the container image. This post describes how to use the Box SDK to read data
from a box folder inside a Kubernetes app.

## Create a Box App and Get Your Credentials
First you need to create a Box app and get your credentials. Start by visiting the [Box Developer Console](https://uofi.app.box.com/developers/console) and creating a new app. Choose "Custom App" and then "Server Authentication (with JWT)". This will create a new app with a public/private key pair that you can use to authenticate with the Box API.

You can leave most the fields as they are, but *NOTE:* due to the way Box handles permissions, you will need to add the "Read and Write All Files and Folders Stored in Box" scope to your app. This is a very broad scope, but it's necessary to read data from a folder that you don't own.

Click on the "Authorization" tab and click on "Review and Submit". This will generate a request
for your app to be approved by your Box admin. Once your app is approved, you can click on the "Configuration" tab and download your private key. You will need this key to authenticate with the Box API.

The admins are not always notified of the request, so you may need to follow up with them to get your app approved.
You can submit a [help ticket](https://help.uillinois.edu/TDClient/42/UIUC/Requests/TicketRequests/NewForm?ID=189&RequestorType=Service) to ask them politely to check the list of apps to be approved for yours.

Once the request is approved you can get your _Service Account ID_. This will look like long 
email address. Give this address to your collaborator and ask them to share the folder with this email address. This will give your app access to the folder and its contents.

Finally, click on the _Configuration_ tab and scroll to _Add and Manage Public Keys
_. Generate a "Public/Private Key Pair" and download the generated JSON file. This file
contains keys to the kingdom.

## Create a Kubernetes Secret with Your Credentials
Next, you need to create a Kubernetes secret that contains your Box credentials. You can do this
by creating the following manifest:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sorghum-rna-box-secret
  namespace: igb

type: Opaque
stringData:
  config.json: |
    {
      "boxAppSettings": {
        "clientID": "XXXXXXXXXXXXXXX",
        "clientSecret": "XXXXXXXXXXXXXXX",
        "appAuth": {
          "publicKeyID": "XXXXXX",
          "privateKey": "-----BEGIN ENCRYPTED PRIVATE KEY-----
        }
      },
      "enterpriseID": "xxxxx"
    }
```

Just paste the contents of the JSON file you downloaded from the Box Developer Console into the `config.json` field of the secret. This will make your Box credentials available to your Kubernetes app. Add this file to the
`kubernetes` folder of the cluster repo and allow ArgoCD to deploy it to the cluster.

## Use the Box SDK to Read Data from Box
In your kubernetes app, you can use the Box SDK to read data from Box. Configure your pod
with a volume that mounts the secret you just created. Then, in your app code, you can use the Box SDK to authenticate with the Box API and read data from the folder that your app has access to.


```yaml
  volumes:
    db-secret:
      secret:
        secretName: sorghum-rna-box-secret

  volumeMounts:
    db-secret:
        mountPath: /opt/box
```


```python
import sys
import os
from pathlib import Path
from boxsdk import Client
from boxsdk.auth.jwt_auth import JWTAuth

# If using JWT authentication (recommended for Box Applications)
# You'll need a config file from Box Developer Console
try:
    config = JWTAuth.from_settings_file('/opt/box/config.json')
    client = Client(config)
except Exception as e:
    print(f"Error authenticating with Box: {e}")
    sys.exit(1)

# Read directories from Box folder with id=352402921983
folder_id = '352402921983'
try:
    folder = client.folder(folder_id).get()
except Exception as e:
    print(f"Error accessing Box folder {folder_id}: {e}")
    sys.exit(1)

# Get all items in the folder 
items = folder.get_items()
for item in items:
    if item.type == 'file':
        file_path = os.path.join(local_dir, item.name)
        print(f"Downloading: {item.name}")
        
        # Download file content
        with open(file_path, 'wb') as f:
            item.download_to(f)
```

Or if you want to use R
```R
library(boxr)

box_auth_service(token_file = "/opt/box/config.json")

folder_contents <- as.data.frame(box_ls(dir_id = "368819567266"))

csv_files <- folder_contents[grepl("\\.csv$", folder_contents$name, ignore.case = TRUE), ]

dir.create("/app/data", recursive = TRUE, showWarnings = FALSE)

for (i in seq_len(nrow(csv_files))) {
  box_dl(csv_files$id[i], local_dir = "/app/data", overwrite = TRUE)
}
```

I usually attach a volume to the pod and have a script in the `entrypoint` that runs the code to download the files from Box before starting the main app. This way the files are available in a local directory that your app can read from.
