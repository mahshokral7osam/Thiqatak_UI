# Secret protection: team setup and production deployment

Use this guide for the encryption implementation in this solution. The developer prepares an encrypted file with the server's **public** certificate. The production API uses the matching **private** key on the server to read it. Secret values stay in local `appsettings.json` for local development; the published API gets them from the encrypted file.

Run commands in **PowerShell** unless a step says **Command Prompt**. Every example below uses the real TheqaTech project names, server path, application ID, and certificate thumbprint. When another team uses this code, they change those values to theirs. Do not put real passwords or tokens in this guide, Git, a publish ZIP, or a command argument.

**Current TheqaTech production is already set up:** its certificate thumbprint is `5ACECE1EBA79CEF5FA7118165CB7B93AD319138E` and its encrypted file is at `C:\inetpub\vhosts\thiqatech.com.sa\api.thiqatech.com.sa\host.secrets.production.enc`. Do not create another certificate for that working site. Steps 2 and 3 are for a new server or project; the later steps also explain future updates to this site.

## 1. Connect a new host project to the shared code (developer, once per project)

The existing API and DbMigrator already do this. For another project in the solution:

1. Add a project reference to `src/Al7.TheqaTech.SecretProtection/Al7.TheqaTech.SecretProtection.csproj` in each executable that needs the secrets.
2. In startup, add `using Al7.TheqaTech.SecretProtection;` and call `config.AddTheqaTechProtectedSecrets()` in `ConfigureAppConfiguration`, as shown in [API Program.cs](../src/Al7.TheqaTech.HttpApi.Host/Program.cs) and [DbMigrator Program.cs](../src/Al7.TheqaTech.DbMigrator/Program.cs).
3. Add the new application ID and its **allowed** and **required** configuration keys in [ProtectedSecretCatalog.cs](../src/Al7.TheqaTech.SecretProtection/ProtectedSecretCatalog.cs). The CLI rejects unknown IDs and keys. This is a code change, not a configuration setting.
4. Keep local secrets in the new project's local `appsettings.json`. Publish a separate base `appsettings.json` without secrets, following [the API project file](../src/Al7.TheqaTech.HttpApi.Host/Al7.TheqaTech.HttpApi.Host.csproj) and [appsettings.publish.json](../src/Al7.TheqaTech.HttpApi.Host/appsettings.publish.json). Add a production configuration with `SecretProtection` enabled (step 6).

The command for item 1 with **TheqaTech's actual project paths** is below. The reference is already present in this branch, so the TheqaTech team does not need to run it again. Another team changes the first path to its own host project:

```powershell
dotnet add '.\src\Al7.TheqaTech.HttpApi.Host\Al7.TheqaTech.HttpApi.Host.csproj' reference '.\src\Al7.TheqaTech.SecretProtection\Al7.TheqaTech.SecretProtection.csproj'
```

For **the existing TheqaTech API**, use application ID `TheqaTech.HttpApi.Host` and the commands below without changing the catalog. Its protected keys are `ConnectionStrings:Default`, `AuthServer:CertificatePassPhrase`, `StringEncryption:DefaultPassPhrase`, `Swagger:Password`, `Firebase:CredentialsJson`, `HyperPay:AccessToken`, `HyperSplit:Password`, `HyperSplit:NotificationKey`, and `SiteAccess:Turnstile:SecretKey`. The first five are currently required to be nonempty. Firebase setup is outside this guide, but **do not remove its required key** from the payload unless you change the catalog and the API code; an unused placeholder only allows startup and will not make Firebase calls work. Optional keys may be omitted.

## 2. IT creates the certificate on the production Windows server (once)

Open **PowerShell as Administrator** on the server. Create the folder *before* exporting. Run these lines in order:

```powershell
New-Item -ItemType Directory -Path 'C:\SecureWork' -Force | Out-Null
$cert = New-SelfSignedCertificate -Type Custom -Subject 'CN=TheqaTech-Host-Secrets' -KeyAlgorithm RSA -KeyLength 3072 -KeyUsage KeyEncipherment -KeyExportPolicy NonExportable -CertStoreLocation 'Cert:\LocalMachine\My'
if ($null -eq $cert) { throw 'Certificate creation failed' }
Export-Certificate -Cert $cert -FilePath 'C:\SecureWork\host-public.cer'
$cert.Thumbprint
```

IT sends the resulting `host-public.cer` file and the printed thumbprint to the developer through the approved channel. **Only the public `.cer` leaves the server.** The private key stays in `LocalMachine\My`. The thumbprint is an identifier, not a password. Do not reuse the API's `openiddict.pfx`.

IT can check the certificate later without creating another one:

```powershell
$thumbprint = '5ACECE1EBA79CEF5FA7118165CB7B93AD319138E'
Get-Item "Cert:\LocalMachine\My\$thumbprint" | Select-Object Thumbprint, HasPrivateKey
```

## 3. IT grants the application access to the private key (once)

Run `certlm.msc` on the server. Open **Personal > Certificates**, find the TheqaTech certificate by thumbprint, right click it, choose **All Tasks > Manage Private Keys**, then grant **Read** to the actual Windows account running the TheqaTech API. IT gets this account from the site's IIS/Plesk application pool settings; its exact name is not recorded in this repository. If DbMigrator will run on the server, also grant **Read** to the Windows account that runs it. Do not grant `Everyone`.

The API and DbMigrator also need **Read** on the encrypted file. After the file exists (step 8), IT can grant it with:

```powershell
$apiAccount = Read-Host 'Enter the exact TheqaTech API Windows account shown in IIS/Plesk'
icacls 'C:\inetpub\vhosts\thiqatech.com.sa\api.thiqatech.com.sa\host.secrets.production.enc' /grant "${apiAccount}:R"
```

For an IIS `ApplicationPoolIdentity`, enter `IIS AppPool\` followed by the actual pool name. For a custom identity, enter that Windows account. Grant file Read to the DbMigrator account too if it runs on the server. The application account does not need Write permission.

## 4. Developer prepares the plaintext payload locally

From the **solution root**, create a private working folder and copy the example:

```powershell
New-Item -ItemType Directory -Path 'C:\SecureWork' -Force | Out-Null
Copy-Item -LiteralPath '.\docs\secret-protection-payload.example.json' -Destination 'C:\SecureWork\production-secrets.json'
notepad 'C:\SecureWork\production-secrets.json'
```

For the existing API, replace the example values with the actual production secrets. For another project, make a JSON file with only that project's keys from `ProtectedSecretCatalog.cs`; the TheqaTech example will not match it. Extra keys are rejected. Keep the current `StringEncryption:DefaultPassPhrase` if the database already contains ABP encrypted settings. Keep this plaintext file outside the repository and the website. Do not send it to IT. Check JSON syntax without displaying the values:

```powershell
Get-Content 'C:\SecureWork\production-secrets.json' -Raw | ConvertFrom-Json | Out-Null
```

## 5. Developer builds the encryption CLI and creates the encrypted file

Put IT's `host-public.cer` at `C:\SecureWork\host-public.cer`. From the **solution root**, publish the CLI:

```powershell
dotnet publish '.\src\Al7.TheqaTech.SecretProtection.Cli\Al7.TheqaTech.SecretProtection.Cli.csproj' -c Release -o 'C:\SecureWork\secret-cli'
```

Open **Command Prompt (`cmd.exe`)** for the next command. CMD's `<` passes the JSON file to the CLI as bytes. Change `--application` if step 1 registered a different project ID:

```cmd
dotnet "C:\SecureWork\secret-cli\Al7.TheqaTech.SecretProtection.Cli.dll" encrypt --certificate "C:\SecureWork\host-public.cer" --application "TheqaTech.HttpApi.Host" --environment "Production" --output "C:\SecureWork\host.secrets.production.enc" < "C:\SecureWork\production-secrets.json"
```

Success prints `Encrypted secrets file written.` The `.enc` file is the file to upload. The public `.cer` encrypts; the developer does not need the server's private key. Each encryption creates a new encrypted file, even when the plaintext is unchanged.

Check the file and record its hash in **PowerShell**:

```powershell
Test-Path -LiteralPath 'C:\SecureWork\host.secrets.production.enc' -PathType Leaf
(Get-FileHash -LiteralPath 'C:\SecureWork\host.secrets.production.enc' -Algorithm SHA256).Hash
```

## 6. Set production configuration (developer)

Set these values in the **production** `appsettings.Production.json` for every process reading the file. Use the thumbprint from step 2 and the **exact destination path** from step 8. In JSON, write each Windows `\` as `\\`:

```json
{
  "SecretProtection": {
    "Enabled": true,
    "ApplicationId": "TheqaTech.HttpApi.Host",
    "EnvironmentName": "Production",
    "FilePath": "C:\\inetpub\\vhosts\\thiqatech.com.sa\\api.thiqatech.com.sa\\host.secrets.production.enc",
    "CertificateThumbprint": "5ACECE1EBA79CEF5FA7118165CB7B93AD319138E",
    "CertificateStoreLocation": "LocalMachine",
    "CertificateStoreName": "My"
  }
}
```

The existing API and DbMigrator production files are [here](../src/Al7.TheqaTech.HttpApi.Host/appsettings.Production.json) and [here](../src/Al7.TheqaTech.DbMigrator/appsettings.Production.json). Both use the same application ID and encrypted file. For another project, all four values must match its own encrypted file: application ID, environment, thumbprint, and file path. Keep the published base `appsettings.json` free of plaintext protected secrets. Ordinary values such as URLs and public site keys stay in configuration. Set production payment URLs and entity IDs there if the project is taking live payments; the access token stays in the encrypted payload.

## 7. Publish the applications (developer)

From the solution root, use **new empty output folders** for each release. Replace the project paths for another host:

```powershell
dotnet publish '.\src\Al7.TheqaTech.HttpApi.Host\Al7.TheqaTech.HttpApi.Host.csproj' -c Release -o 'C:\SecureWork\publish-api-new'
dotnet publish '.\src\Al7.TheqaTech.DbMigrator\Al7.TheqaTech.DbMigrator.csproj' -c Release -o 'C:\SecureWork\publish-migrator-new'
```

Check the API publish folder before sending it to IT:

```powershell
Test-Path 'C:\SecureWork\publish-api-new\appsettings.json'
Test-Path 'C:\SecureWork\publish-api-new\appsettings.secrets.json'
Test-Path 'C:\SecureWork\publish-api-new\openiddict.pfx'
Test-Path 'C:\SecureWork\publish-api-new\host.secrets.production.enc'
```

Only the first check should be `True`. Open the published `appsettings.json` and verify protected values are blank or absent. Check that the published `web.config` blocks `.pfx` and `.enc`:

```powershell
Select-String -Path 'C:\SecureWork\publish-api-new\web.config' -Pattern 'fileExtension=".pfx" allowed="false"', 'fileExtension=".enc" allowed="false"'
```

Both lines must appear. Upload the **contents of the API publish folder** to the Plesk/IIS API site. Keep DbMigrator and the CLI outside the web root. `openiddict.pfx` and the `.enc` file are supplied separately.

## 8. IT puts the encrypted file on the server

IT first makes a staging folder on the server:

```powershell
New-Item -ItemType Directory -Path 'C:\SecureWork' -Force | Out-Null
```

Transfer **only** `C:\SecureWork\host.secrets.production.enc` to `C:\SecureWork\host.secrets.production.enc` on the server using your team's approved file transfer. IT then copies it to the `FilePath` in step 6. For the current TheqaTech site, that path is `C:\inetpub\vhosts\thiqatech.com.sa\api.thiqatech.com.sa\host.secrets.production.enc`.

On the server, IT checks the destination and compares its hash to the developer's hash from step 5:

```powershell
$file = 'C:\inetpub\vhosts\thiqatech.com.sa\api.thiqatech.com.sa\host.secrets.production.enc'
Copy-Item -LiteralPath 'C:\SecureWork\host.secrets.production.enc' -Destination $file -Force
Test-Path -LiteralPath $file -PathType Leaf
(Get-FileHash -LiteralPath $file -Algorithm SHA256).Hash
```

The first result must be `True` and the hashes must match. Then apply the file permission from step 3. If `Test-Path` is `False`, fix the upload destination or the `FilePath` value before starting the API.

## 9. IT starts the production API and runs migrations when needed

In Plesk/IIS, set the API environment to `Production` (`ASPNETCORE_ENVIRONMENT=Production`) and recycle/start the site. Keep the existing `openiddict.pfx` in the API content root with restricted access; it is a different certificate. Delete plaintext secret files left by any old deployment, including `appsettings.secrets.json` and old files under `wwwroot`. Copying a new publish folder does not delete old files.

If this release needs database changes, place the **DbMigrator publish folder outside the website**. Run it on the server under the Windows account granted certificate and file Read access in step 3:

```powershell
Set-Location 'C:\SecureWork\publish-migrator-new'
$env:DOTNET_ENVIRONMENT = 'Production'
dotnet '.\Al7.TheqaTech.DbMigrator.dll'
```

Back up the target database and confirm it is the intended database before running migrations. The existing developer workflow can still run DbMigrator locally from Visual Studio with its local settings; that is separate from this production command.

## 10. Verify and update later

On the server, confirm the production configuration points to the file actually uploaded:

```powershell
(Get-Content 'C:\inetpub\vhosts\thiqatech.com.sa\api.thiqatech.com.sa\appsettings.Production.json' -Raw | ConvertFrom-Json).SecretProtection | Format-List
```

Then open the API, check the startup log for success, and test one operation that uses a protected secret (for example, database access). For later password or token changes: edit the local plaintext payload, rerun **step 5** with the same public `.cer`, upload the new `.enc`, compare hashes, and recycle the API. There is **no decryption command for the developer**; the application decrypts at startup using the server private key.

| Error | Check first |
| --- | --- |
| `Protected secrets file is missing` | `FilePath` and `Test-Path` in step 8. |
| `Protected secrets envelope was rejected` | Correct `.enc` hash, application ID, `Production` environment, certificate thumbprint, and required payload keys. |
| Certificate/private key unavailable | Certificate in `LocalMachine\My` and private-key **Read** permission for the process account. |

The official Windows/IIS references for IT are [New-SelfSignedCertificate](https://learn.microsoft.com/en-us/powershell/module/pki/new-selfsignedcertificate), [application pool identities](https://learn.microsoft.com/en-us/iis/manage/configuring-security/application-pool-identities), and [publishing ASP.NET Core to IIS](https://learn.microsoft.com/en-us/aspnet/core/tutorials/publish-to-iis).
