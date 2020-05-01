# Simple Data Sharing with Blob Storage Accounts -- "The Blob Uploader"

> If you need an easy way for your customers to share files with you, check this out:   

## What is this?  

Customers often need to share large data files with me.  How can we do this in the most friction-free way?  

* Azure storage accounts work well, but not every customer has Storage Explorer available to upload files.  
* Everyone understands FTP but who wants to spin up an FTP server every time data sharing is needed?
  * We want a "no-infrastructure" solution
* Everyone has access to a browser, so a drag-and-drop interface would be best.  
* Security is important but if the sharing mechanism is short-lived we can make some trade-offs.  I never ask for confidential data, or data that is business-critical using this approach.  

## Solution

We rely on some features of Azure storage accounts to meet the above requirements:

* spin up a storage account and secure it with a SAS token
* storage accounts support static websites, so we build a little website that will allow a customer to upload files via drag/drop

> Important:  Since this solution doesn't have a server-side component, we do need to **hardcode the SAS key in some javascript**.  This is a short-lived solution and I'm relying on obfuscation to ensure no one hacks the storage account.  **The container does NOT have anonymous  write access**.  We only allow customers to WRITE to the storage account, which helps a little with security.  We also limit access to 5 days.

## Technical Details

* there are no practical file size limits.  There is no server to enforce them.  
* the Azure Storage SDK for Javascript is used to communicate to the storage account
* we set the SAS key to expire in 10 days.  If your customer can't upload their data in that amount of time, change the var below.  
* you can use an URL shortener to make this even more intuitive for your customers.  

## Deployment

Open `Cloud Shell` from the Portal, choose `bash`, and follow along:

```bash
# vars to change
SUBSCRIPTION="davew demo"
RES_GROUP="rgBlobUploader"
LOCATION="eastus"
# this must be unique across Azure, best to obfuscate it
ACCT_NAME="dkfjekfcurt"
ERR_DOC="error.html"
INDEX_DOC="upload.html"
UPLOAD_CONTAINER="incoming"
# date the SAS expires (5 days as coded)
EXPIRY=$(date +%Y-%m-%d -d "$(date) + 5 day")
EXPIRY=$EXPIRY"T00:00:00Z"
GIT_ROOT="git"
REPO="https://github.com/davew-msft/blob-data-sharing"
FOLDER="blob-data-sharing"
AMPR="&"

az account set -s "$SUBSCRIPTION"
az group create -n $RES_GROUP -l $LOCATION

az storage account create \
    --name $ACCT_NAME \
    --location $LOCATION \
    -g $RES_GROUP \
    --https-only

# set up the static website option on the storage account
az storage blob \
    service-properties update \
    --account-name $ACCT_NAME \
    --static-website \
    --404-document "$ERR_DOC" \
    --index-document "$INDEX_DOC"

# get the website URL
URL=$(az storage account show \
    -n $ACCT_NAME \
    -g $RES_GROUP \
    --query "primaryEndpoints.web" --output tsv)

# enable CORS so the javascript works
az storage cors add \
    --methods PUT \
    --origins $URL \
    --services b \
    --account-name $ACCT_NAME \
    --allowed-headers * \
    --subscription "$SUBSCRIPTION" \
    --exposed-headers * \
    --max-age 86400

# build the upload container, this is where customer data gets uploaded
az storage container create \
    -n $UPLOAD_CONTAINER \
    --account-name $ACCT_NAME \
    --subscription "$SUBSCRIPTION"

# create a SAS token to upload container
SAS=$(az storage container generate-sas \
    --account-name $ACCT_NAME \
    --name $UPLOAD_CONTAINER \
    --permissions acwl \
    --expiry "$EXPIRY" \
    --auth-mode key | tr -d \") 

# upload the static website assets
# to do this we are going to create a root git folder in cloudshell
# then cd into it
# then clone this repo which has the static website assets
# then we can push the static assets to our storage account
mkdir -p $GIT_ROOT
cd $GIT_ROOT
git clone $REPO $FOLDER

# this syntax assumes running from cloud shell.  You may not need the backslash in \$web otherwise
az storage blob upload-batch \
    -s $FOLDER/web \
    -d \$web \
    --account-name $ACCT_NAME \
    --content-type 'text/html; charset=utf-8'

echo "This is the link to give to customers, try it out first:"
# https://<url>/index.html<SASKey>&accountName=<accountname>&containerName=<containername>
echo $URL$INDEX_DOC?$SAS$AMPRaccountName=$ACCT_NAME$AMPRcontainerName=$UPLOAD_CONTAINER
# https://dkfjekfcurt.z13.web.core.windows.net/upload.html?se=2020-05-06T00%3A00%3A00Z&sp=acwl&sv=2018-11-09&sr=c&sig=CHjXCuY20aYUSyzD%2BRB6Pr1s89R8a7x2yCeuDx2RRvA%3D&accountName=dkfjekfcurt&containerName=incoming


https://dkfjekfcurt.blob.core.windows.net/incoming?se=2020-05-06T00%3A00%3A00Z
&sp=acwl
&sv=2018-11-09
&sr=c
&sig=CHjXCuY20aYUSyzD%2BRB6Pr1s89R8a7x2yCeuDx2RRvA%3D
```


