# Azure Migrate Agents Guide

## Overview
This guide provides instructions on using Azure Migrate Agents for various migration scenarios.

...

## Step 5: File Uploads
### Step 5 Instructions
1. Prepare your files for upload to Azure Blob Storage.
2. Generate a per-blob SAS URL recommended via Azure Function:
   - Deploy an Azure Function that handles the generation of SAS URLs.
   - The Azure Function should accept parameters for the desired blob storage account, container, and blob name and return a SAS URL.
3. Use the generated SAS URL to upload your files directly to the blob storage.
4. The Azure Function will also handle scenarios where multiple files need to be uploaded simultaneously.
5. Always ensure that the Azure Function returns appropriate outputs to confirm the success of the uploads and provide any necessary diagnostics or error messages.

...