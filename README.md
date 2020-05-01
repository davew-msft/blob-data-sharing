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

# create a SAS token to upload container
SAS=$(az storage container generate-sas \
    --account-name $ACCT_NAME \
    --name $UPLOAD_CONTAINER \
    --permissions acw \
    --expiry "$EXPIRY" \
    --auth-mode key )

# upload the static website assets


```


