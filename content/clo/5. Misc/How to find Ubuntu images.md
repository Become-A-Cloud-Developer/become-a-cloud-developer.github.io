# Find Ubuntu Images

```bash
# List available publishers for Ubuntu (in northeurope)
az vm image list-publishers --location northeurope --query "[?contains(name,'Ubuntu')].name" -o tsv

# List available offers for a publisher (e.g., Canonical, in northeurope)
az vm image list-offers --location northeurope --publisher Canonical --query "[].name" -o tsv

# List available SKUs for an offer (e.g., ubuntu-24_04-lts, in northeurope)
az vm image list-skus --location northeurope --publisher Canonical --offer ubuntu-24_04-lts --query "[].name" -o tsv

# List available versions for a SKU (e.g., server, in northeurope)
az vm image list --location northeurope --publisher Canonical --offer ubuntu-24_04-lts --sku server --query "[].version" -o tsv
```

Remember to replace `northeurope` with your desired Azure region if it's different.  These commands will give you the information you need to populate the `imageReference` section of your Bicep or ARM template.

