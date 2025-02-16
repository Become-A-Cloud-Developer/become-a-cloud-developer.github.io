# Provision and deploy to VM

Provision and deploy the webapp to an Azure VM running Ubuntu

1. Provision VM

    - Create a resource group
    - Provision an Ubuntu VM with SSH keys
    - Open port 5000 for the webapp

2. Configure VM for .Net

    - Install .Net run-time
    - Prepare a directory to store the app artifacts (/opt/<App>)
    - Create a systemd service file
    - Keep environment variables in a separate file (/etc/<App>/.env)

3. Deploy App to VM

    - Copy the published app artifacts to the /opt/<App>
    - Remember to stop and start the service before and after redeploy