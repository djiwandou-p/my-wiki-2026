# Auto Renew Fabric Token using Azure VM

#### Summary
I prefer schedule running powershell script in task scheduler windows and wrap it into a simple background service that exposes the token via a local http endpoint
Here’s a **step‑by‑step guide** to run your existing `az`‑based Fabric token script on an **Azure Windows VM**, schedule it via **Task Scheduler**, and expose the token via a **simple local HTTP endpoint** (PowerShell‑based HTTP listener).[^1][^2]

***

## 1. Prerequisites on the VM

Before anything else, log in to your **Azure Windows VM** (RDP or Bastion) and:

1. Install **Azure CLI** (if not already installed):
    - From PowerShell as admin:

```powershell
winget install Microsoft.AzureCLI
```

    - Or download from: https://aka.ms/installazurecliwindows[^2]

![install_azure_cli.png](https://raw.githubusercontent.com/djiwandou-p/my-wiki-2026/main/wiki/images/1776995758092-install_azure_cli.png)

2. Install **PowerShell Az module** (for `Get-AzAccessToken` fallback, optional):

```powershell
Install-Module Az -Scope CurrentUser -Force
Import-Module Az.Accounts
```

3. Login once (choose one):
    - Interactive (for user‑based apps):

```powershell
az login
```

    - Or service principal (for automation):

```powershell
az login --service-principal -u "<app-id>" -p "<client-secret>" --tenant "<tenant-id>"
``` [^1][^2]  

```


This will persist the context so `az account get-access-token` works without interactive login.

***

## 2. Create the PowerShell script to get and cache the token

Save this script as `C:\Scripts\Get-FabricToken.ps1` (adjust path as needed):

```powershell
# Get-FabricToken.ps1

$ErrorActionPreference = "Stop"

# Token file to cache
$TokenFile = "F:\Tokens\fabric.token.txt"

# Ensure token directory exists
$TokenDir = Split-Path $TokenFile -Parent
if (!(Test-Path $TokenDir)) {
    New-Item -ItemType Directory -Path $TokenDir -Force
}

# Get Fabric access token via Azure CLI
$tokenRaw = az account get-access-token --resource "https://api.fabric.microsoft.com" --query accessToken -o tsv

if ($LASTEXITCODE -ne 0 -or [string]::IsNullOrWhiteSpace($tokenRaw)) {
    Write-Error "Failed to get Fabric access token"
}

# Write to file (each call overwrites; you can also encode/encrypt if you want)
$tokenRaw | Out-File -FilePath $TokenFile -Encoding utf8 -Force

Write-Output "Fabric token refreshed at $(Get-Date). Token length: $($tokenRaw.Length)"
```

Run once manually to test:

```powershell
C:\Scripts\Get-FabricToken.ps1
```

Check that `C:\Tokens\fabric.token.txt` exists and contains a valid JWT.[^1][^2]

***

## 3. Schedule it with Task Scheduler

On the Windows VM:

1. Open **Task Scheduler** (search from Start).
2. **Create Task** (not “Basic Task” so you can set advanced options).

### General tab

- Name: `Refresh Fabric Token`
- Security options:
    - Run whether user is logged on or not
    - Run with highest privileges


### Triggers tab

- New → Daily (or whatever pattern you prefer)
- Advanced settings:
    - Repeat task every: **60 minutes**
    - Duration: **Indefinitely**

This keeps the token refreshed every ~1 hour (so you’re always within 2‑hour lifetime).[^3][^4]

### Actions tab

- New → Action:
    - Program: `PowerShell.exe`
    - Add arguments:

```text
-ExecutionPolicy Bypass -File "C:\Scripts\Get-FabricToken.ps1"
```


### Conditions \& Settings

- Uncheck “Stop if computer switches to battery power” (if relevant).
- Allow task to be run on demand for testing.

Click **OK**, then **Run** once to test.

***

## 4. Create a simple HTTP endpoint to expose the token

Save this script as `C:\Scripts\Start-FabricTokenHttp.ps1`:

```powershell
# Start-FabricTokenHttp.ps1

Add-Type -TypeDefinition @"
using System;
using System.Net;
using System.IO;
using System.Text;

public class HttpListenerWrapper
{
    public static void Start()
    {
        HttpListener listener = new HttpListener();
        listener.Prefixes.Add("http://localhost:8080/token/");
        listener.Start();
        Console.WriteLine("Listening on http://localhost:8080/token/");

        while (true)
        {
            HttpListenerContext context = listener.GetContext();
            HttpListenerRequest request = context.Request;
            HttpListenerResponse response = context.Response;

            try
            {
                string tokenPath = @"F:\Tokens\fabric.token.txt";
                if (!File.Exists(tokenPath))
                {
                    response.StatusCode = 404;
                    string body = "Token not found";
                    byte[] buffer = Encoding.UTF8.GetBytes(body);
                    response.ContentLength64 = buffer.Length;
                    response.OutputStream.Write(buffer, 0, buffer.Length);
                }
                else
                {
                    string token = File.ReadAllText(tokenPath);
                    response.StatusCode = 200;
                    response.ContentType = "text/plain";
                    byte[] buffer = Encoding.UTF8.GetBytes(token.Trim());
                    response.ContentLength64 = buffer.Length;
                    response.OutputStream.Write(buffer, 0, buffer.Length);
                }
            }
            catch (Exception ex)
            {
                response.StatusCode = 500;
                string body = "Internal error";
                byte[] buffer = Encoding.UTF8.GetBytes(body);
                response.ContentLength64 = buffer.Length;
                response.OutputStream.Write(buffer, 0, buffer.Length);
            }

            response.OutputStream.Close();
        }
    }
}
"@

# Start the HTTP listener in a background thread
$job = Start-ThreadJob -ScriptBlock { [HttpListenerWrapper]::Start() }

# Keep the script alive until user stops it
$job | Wait-Job
```

If `Start-ThreadJob` is not available, install it once:

```powershell
Install-Module ThreadJob -Scope CurrentUser -Force
```

Now you can start the HTTP service manually:

```powershell
C:\Scripts\Start-FabricTokenHttp.ps1
```

Then test from another PowerShell:

```powershell
Invoke-RestMethod http://localhost:8080/token/
```

This should return the current Fabric token.[^5][^1]

***

## 5. Run the HTTP service as a background service

Since this is a Windows VM, you have a few options:

### Option A: Simple – run as a scheduled task on login

1. Create a new Task Scheduler task:
    - General:
        - Name: `Start Fabric Token HTTP Service`
        - Run whether user is logged on or not
        - Run with highest privileges
    - Triggers:
        - At log on
    - Actions:
        - Program: `PowerShell.exe`
        - Add arguments:

```text
-ExecutionPolicy Bypass -File "C:\Scripts\Start-FabricTokenHttp.ps1"
```

    - Conditions:
        - Run task as soon as possible after a scheduled start is missed

This keeps the HTTP listener running whenever the machine is up.[^4][^3]

### Option B (more robust): Run as a Windows Service (NSSM)

If you want it to be a proper Windows service:

1. Download **NSSM** (Non‑Sucking Service Manager) from https://nssm.cc.[^6]
2. Extract and run `nssm.exe install FabricTokenService`.
3. In the dialog:
    - Application Path: `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe`
    - Arguments:

```text
-ExecutionPolicy Bypass -File "C:\Scripts\Start-FabricTokenHttp.ps1"
```

4. Set it to **Start automatically**.

Then:

```powershell
nssm start FabricTokenService
```

Now the HTTP endpoint runs as a Windows service, independently of user sessions.[^6]

***

## 6. How your apps get the token from the VM

From your **local laptop or another service**:

```powershell
$token = Invoke-RestMethod -Uri http://<vm-private-ip>:8080/token/ -UseBasicParsing
$headers = @{
    "Authorization" = "Bearer $token"
}

Invoke-RestMethod -Uri "https://api.fabric.microsoft.com/v1/workspaces" -Headers $headers
```

Replace `<vm-private-ip>` with the VM’s internal IP (or use a private DNS name if you have one).[^7][^1]

***

## 7. Security considerations

- Do **not** open the HTTP port (`8080`) to the public internet unless you put it behind a **private endpoint / NSG / firewall**.
- Prefer **private VNet and NSG rules** to restrict the HTTP listener to specific IPs or services.
- Optionally, add basic auth or a simple API key in the HTTP listener (e.g., check a header or query param) before returning the token.[^7][^1]

***

If you tell me:

- whether you want the HTTP service to run on **port 80** vs **8080**,
- and whether you prefer **NSSM‑style Windows service** vs **Task Scheduler on login**,

I can give you a final **one‑shot setup script** you can paste on the VM that:

- installs `az`, installs `ThreadJob`, creates the two scripts, and configures Task Scheduler / NSSM automatically. [^10][^8][^9]



[^1]: https://learn.microsoft.com/en-us/answers/questions/1161228/script-to-request-and-get-access-token-from-azure

[^2]: https://azurebuild.wordpress.com/2020/09/11/acquire-azure-access-token-with-powershell/

[^3]: https://learn.microsoft.com/en-us/answers/questions/2137812/how-to-schedule-on-azure-to-run-a-powershell-scrip

[^4]: https://stackoverflow.com/questions/36842359/what-is-the-best-option-for-setting-up-task-scheduler-hosted-on-windows-azure-us

[^5]: https://stackoverflow.com/questions/48070645/using-powershell-to-get-azure-ad-token-jwt

[^6]: https://stackoverflow.com/questions/47529307/running-powershell-script-in-azure-as-a-task-scheduler-without-creating-any-virt

[^7]: https://learn.microsoft.com/en-us/rest/api/azure/

[^8]: https://gist.github.com/lrakai/51d0c36a74a150dbed15f4c0e3bb82d6

[^9]: https://learn.microsoft.com/en-au/answers/questions/2137812/how-to-schedule-on-azure-to-run-a-powershell-scrip

[^10]: https://github.com/Azure/azure-powershell/issues/2494


