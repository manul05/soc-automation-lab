[Back to README](../README.md)

# 02. Wazuh Server (Manager, Indexer, Dashboard)
**Server**: Ubuntu 24.04 LTS (Recommended)  
**Host IP**: `<WAZUH_PUBLIC_IP>`  
**Installation Type**: All-in-one (Wazuh indexer, server, and dashboard on a single host)  

> [!IMPORTANT]
> **System Requirements**
> Ensure your cloud instance (DigitalOcean, Vultr, etc.) has **at least 8GB of RAM**. The Wazuh 4.14.x series indexer is more resource-heavy, and older kernels or lower memory footprints can result in system instability or failure during installation.

---

## Step 1 — Download and Run the Installation Assistant

Wazuh provides a streamlined installation assistant script. We will use the `4.14` branch alias to ensure we pull the latest stable version of the script.

To download the script and execute the **All-in-one** installation (using the `-a` flag), run:

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```

> [!NOTE]
> Keep an eye on the terminal output as the script runs. Once it finishes successfully, it will output a summary to your terminal.

---

## Step 2 — Retrieve the Dashboard Credentials

After the script finishes, it generates a `wazuh-install-files.tar` file in the directory where the script was run. This archive stores your unique SSL certificates and the `wazuh-passwords.txt` file.

To cleanly view the generated passwords without cluttering your directory with extracted certificate files, run the following command to print the passwords directly to the terminal:

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

*(Optional: If you ever specifically need the certificate files, you can extract the entire archive using `sudo tar -xvf wazuh-install-files.tar`)*

---

> [!TIP]
> Save the generated passwords in a password manager and **never commit them**. The `wazuh-install-files.tar` archive also contains private keys, so delete it from the server once you have copied what you need.

## Step 3 — Access the Dashboard

Once you have retrieved your passwords, you can access the Wazuh Dashboard interface:

1. Open your web browser and navigate to: `https://<WAZUH_PUBLIC_IP>`
2. **Username**: `admin`
3. **Password**: *(paste the admin password obtained in Step 2)*

> [!WARNING]
> Because the installation assistant generates self-signed SSL certificates by default, your web browser will likely show a "Your connection is not private" or security warning. You can safely proceed past this warning (e.g., clicking "Advanced" -> "Proceed to `<WAZUH_PUBLIC_IP>`") to reach the login screen.
