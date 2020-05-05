# Simple Data Sharing with Blob Storage Accounts -- "The Blob Uploader"

> If you need an easy way for your customers/partners to share large files with you, check this out.  Not everyone is familiar with Azure Storage accounts (blobs) and Storage Explorer and walking the uninitiated through that is frustrating.  And I don't want to setup a bunch of infrastructure like FTP servers when I just want to receive a couple large files.  This is a real quick way to setup a serverless html-based drag-and-drop interface to upload files to Azure.  You can even do this in a free Azure account.  _Some security caveats apply_.  Read on. 

## What is this?  

Customers often need to share large data files with me.  How can we do this in the most friction-free way?  

* Azure storage accounts work well, but not every customer has Storage Explorer available to upload files.  
* Everyone understands FTP but who wants to spin up an FTP server every time data sharing is needed?
  * We want a "no-infrastructure" solution (not even an Azure Function is needed.  Just a storage account)
* Everyone has access to a browser, so a drag-and-drop interface would be best.  
* Security is important but if the sharing mechanism is short-lived we can make some trade-offs.  I never ask for confidential data using this approach.  

## Solution

We rely on some features of Azure storage accounts to meet the above requirements:

* spin up a storage account and secure it with a SAS token
* storage accounts support static websites, so we build a little website that will allow a customer to upload files via drag-and-drop
* the static website has a little javascript that reads the SAS token from the URL string.  

> Important:  Since this solution doesn't have a server-side component, we can't do authentication there.  We pass the SAS key on the URL string that we provide to our customer.  URL strings aren't encrypted so it's possible this could be sniffed by a 3rd party.  This is a short-lived solution and I'm relying on obfuscation to ensure no one hacks the storage account.  **The container does NOT have anonymous write access**.  Those are easy to hack for the little script kiddies out there that like to cause mischief.  We only allow customers to WRITE to the storage account (via the SAS token), which helps a little with security.  We also limit access to 5 days.  Again, **we are relying on security-by-obfuscation**, mostly. I always explain this to my customers and let them decide if this approach is OK with them.   

## Technical Details

* there are no practical file size limits.  There is no server to enforce them.  I've upload multi-TB files this way, assuming a good network connection.  
* the Azure Storage SDK for Javascript is used to communicate to the storage account
* we set the SAS key to expire in 5 days.  If your customer can't upload their data in that amount of time, change the var below.  SAS keys are difficult/impossible to hack.  
* you can use an URL shortener to make this even more intuitive for your customers.  

## Deployment

Here is how we deploy quickly.  

Open `Cloud Shell` from the Portal, choose `bash`, and follow along:

```bash
# vars to change
SUBSCRIPTION="davew demo"
RES_GROUP="rgBlobUploaderTemp"
LOCATION="eastus"
# this must be unique across Azure, best to obfuscate it
ACCT_NAME="asdfdaveshare"
ERR_DOC="error.html"
INDEX_DOC="upload.html"
UPLOAD_CONTAINER="incoming"
# date the SAS expires (5 days as coded)
EXPIRY=$(date +%Y-%m-%d -d "$(date) + 5 day")
GIT_ROOT="git"
REPO="https://github.com/davew-msft/blob-data-sharing"
FOLDER="blob-data-sharing"
AMPR="&"

# other vars 
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
    --methods DELETE GET HEAD MERGE OPTIONS POST PUT \
    --origins "*" \
    --services b \
    --account-name $ACCT_NAME \
    --allowed-headers "*" \
    --subscription "$SUBSCRIPTION" \
    --exposed-headers "*" \
    --max-age 86400

# build the upload container, this is where customer data gets uploaded
az storage container create \
    -n $UPLOAD_CONTAINER \
    --account-name $ACCT_NAME \
    --subscription "$SUBSCRIPTION"

# create a SAS token to upload container
SAS=$(az storage account generate-sas \
    --account-name $ACCT_NAME \
    --resource-types sco\
    --services b \
    --permissions racwdl \
    --expiry "$EXPIRY" \
    | tr -d \") 

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

# echo the customer url
echo "This is the link to give to customers, try it out first:"
urldecode() { : "${*//+/ }"; echo -e "${_//%/\\x}"; }
cust_url_enc=$URL$INDEX_DOC?$SAS\&accountName=$ACCT_NAME\&containerName=$UPLOAD_CONTAINER
cust_url_dec=$(urldecode "$cust_url_enc")
echo "$cust_url_dec"


echo "when you are finished with the data please delete the resource group with:"
echo "az group delete --name $RES_GROUP --subscription "$SUBSCRIPTION" -y"

```
