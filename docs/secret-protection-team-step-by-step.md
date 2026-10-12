# تطبيق تشفير أسرار الإنتاج خطوة بخطوة

هذا دليل لتنفيذ نفس طريقة التشفير الموجودة في حل TheqaTech على مشروع جديد. الأوامر تستخدم أسماء ومسارات TheqaTech كمثال عملي؛ عند استخدامها لمشروع آخر غيّر اسم المشروع، رابط الموقع، ومسار السيرفر إلى قيم مشروعك.

**الفكرة ببساطة:** مسؤول السيرفر ينشئ شهادة ويعطي المطوّر الجزء العام منها (`.cer`). المطوّر يستخدمه لتشفير ملف الأسرار. التطبيق على السيرفر يستخدم الجزء الخاص من نفس الشهادة لفك التشفير عند التشغيل. إعدادات التطوير المحلية تظل كما هي. **لا ترسل ملف الأسرار غير المشفّر إلى السيرفر.**

نفّذ الأوامر في **PowerShell**، إلا إذا كتبنا **Command Prompt**. ابدأ أوامر المطوّر من مجلد الحل `D:\workGit\TheqaTech`.

## 1. اربط التطبيق بمشروع التشفير — المطوّر

أضف مرجع مشروع التشفير للتطبيق وأداة ترحيل قاعدة البيانات:

```powershell
dotnet add '.\src\Al7.TheqaTech.HttpApi.Host\Al7.TheqaTech.HttpApi.Host.csproj' reference '.\src\Al7.TheqaTech.SecretProtection\Al7.TheqaTech.SecretProtection.csproj'
dotnet add '.\src\Al7.TheqaTech.DbMigrator\Al7.TheqaTech.DbMigrator.csproj' reference '.\src\Al7.TheqaTech.SecretProtection\Al7.TheqaTech.SecretProtection.csproj'
```

في `Program.cs` لكل تطبيق، أضف `using Al7.TheqaTech.SecretProtection;` ثم استدعِ `config.AddTheqaTechProtectedSecrets()` داخل `ConfigureAppConfiguration`. افتح المثالين الموجودين في الحل:

```powershell
notepad '.\src\Al7.TheqaTech.HttpApi.Host\Program.cs'
notepad '.\src\Al7.TheqaTech.DbMigrator\Program.cs'
```

في مشروع آخر، سجّل اسم التطبيق والمفاتيح المسموحة والمطلوبة في [ProtectedSecretCatalog.cs](../src/Al7.TheqaTech.SecretProtection/ProtectedSecretCatalog.cs). اسم تطبيق هذا المثال هو `TheqaTech.HttpApi.Host`. أداة التشفير ترفض اسم تطبيق أو مفتاح غير مسجل.

## 2. جهّز إعدادات النشر — المطوّر

اترك أسرار التطوير في `appsettings.json` المحلي. أنشئ ملف إعدادات للنشر بدون قيم الأسرار، مثل `appsettings.publish.json`. اضبط ملف `.csproj` لكي ينشر هذا الملف باسم `appsettings.json` **ولا ينشر الملف المحلي**. افتح أمثلة TheqaTech:

```powershell
notepad '.\src\Al7.TheqaTech.HttpApi.Host\appsettings.publish.json'
notepad '.\src\Al7.TheqaTech.HttpApi.Host\Al7.TheqaTech.HttpApi.Host.csproj'
notepad '.\src\Al7.TheqaTech.DbMigrator\Al7.TheqaTech.DbMigrator.csproj'
```

مهم: ملف نشر الـ API لا يحتوي كلمات مرور أو tokens. ملف نشر DbMigrator لا يحتوي `appsettings.json` المحلي أيضًا. القيم العادية، مثل روابط المواقع والمفاتيح العامة، تظل في الإعدادات.

## 3. أنشئ شهادة التشفير — مسؤول السيرفر IT

افتح **PowerShell كمسؤول** على Windows Server، ونفّذ الأوامر **بالترتيب**:

```powershell
New-Item -ItemType Directory -Path 'C:\SecureWork' -Force | Out-Null
$cert = New-SelfSignedCertificate -Type Custom -Subject 'CN=TheqaTech-Host-Secrets' -KeyAlgorithm RSA -KeyLength 3072 -KeyUsage KeyEncipherment -KeyExportPolicy NonExportable -CertStoreLocation 'Cert:\LocalMachine\My'
if ($null -eq $cert) { throw 'Certificate creation failed' }
Export-Certificate -Cert $cert -FilePath 'C:\SecureWork\host-public.cer'
$cert.Thumbprint
```

أرسل للمطوّر **الملف `host-public.cer` والبصمة المطبوعة فقط**. لا ترسل المفتاح الخاص ولا ملف `.pfx`. البصمة ستختلف في كل مرة تُنشأ فيها شهادة جديدة؛ استخدم البصمة الناتجة من هذا الأمر في الخطوة 7.

## 4. امنح التطبيق حق قراءة المفتاح الخاص — مسؤول السيرفر IT

افتح مدير الشهادات:

```powershell
certlm.msc
```

اذهب إلى **Personal → Certificates**، وابحث عن الشهادة بالبصمة من الخطوة 3. اضغط باليمين **All Tasks → Manage Private Keys**، ثم أضف حساب تشغيل تطبيق الـ API بصلاحية **Read**. اعرف اسم الحساب الحقيقي من إعدادات Application Pool في IIS/Plesk. إذا ستشغّل DbMigrator على السيرفر، أضف حساب تشغيله بصلاحية **Read** أيضًا. لا تضف `Everyone`.

## 5. جهّز ملف الأسرار على جهاز المطوّر

انسخ المثال خارج مجلد Git، ثم افتحه واملأ قيم الإنتاج:

```powershell
New-Item -ItemType Directory -Path 'C:\SecureWork' -Force | Out-Null
Copy-Item -LiteralPath '.\docs\secret-protection-payload.example.json' -Destination 'C:\SecureWork\production-secrets.json'
notepad 'C:\SecureWork\production-secrets.json'
```

ضع مثلًا اتصال قاعدة البيانات، كلمة مرور شهادة OpenIddict، `StringEncryption:DefaultPassPhrase`، كلمة مرور Swagger، و`HyperPay:AccessToken` و`SiteAccess:Turnstile:SecretKey` عند استخدامهما. **لا تغيّر `StringEncryption:DefaultPassPhrase` لقاعدة بيانات فيها بيانات مشفّرة بالفعل.** التنفيذ الحالي يشترط أيضًا وجود قيمة غير فارغة لـ `Firebase:CredentialsJson` حتى لو لم تستخدم Firebase الآن.

تأكد أن صيغة JSON صحيحة بدون عرض القيم:

```powershell
Get-Content 'C:\SecureWork\production-secrets.json' -Raw | ConvertFrom-Json | Out-Null
```

## 6. شفّر الملف على جهاز المطوّر

احفظ `host-public.cer` الذي أرسله IT في `C:\SecureWork\host-public.cer`. ابنِ أداة التشفير:

```powershell
([System.Security.Cryptography.X509Certificates.X509Certificate2]::new('C:\SecureWork\host-public.cer')).Thumbprint
dotnet publish '.\src\Al7.TheqaTech.SecretProtection.Cli\Al7.TheqaTech.SecretProtection.Cli.csproj' -c Release -o 'C:\SecureWork\secret-cli'
```

تأكد أن البصمة المطبوعة هنا هي نفس البصمة التي أرسلها IT في الخطوة 3. لا تستخدم ملف `.cer` قديمًا من تجربة أخرى.

افتح **Command Prompt (`cmd.exe`)** ونفّذ هذا الأمر؛ علامة `<` تقرأ الأسرار من الملف، فلا تظهر كلمات المرور داخل الأمر:

```cmd
dotnet "C:\SecureWork\secret-cli\Al7.TheqaTech.SecretProtection.Cli.dll" encrypt --certificate "C:\SecureWork\host-public.cer" --application "TheqaTech.HttpApi.Host" --environment "Production" --output "C:\SecureWork\host.secrets.production.enc" < "C:\SecureWork\production-secrets.json"
```

إذا نجح، ستظهر `Encrypted secrets file written.` افحص الملف وسجّل البصمة `SHA256` لمقارنتها على السيرفر:

```powershell
Test-Path -LiteralPath 'C:\SecureWork\host.secrets.production.enc' -PathType Leaf
(Get-FileHash -LiteralPath 'C:\SecureWork\host.secrets.production.enc' -Algorithm SHA256).Hash
```

الملف الذي سترفعه هو `.enc` فقط، وليس `production-secrets.json`.

## 7. اضبط `appsettings.Production.json` — المطوّر

افتح ملف إنتاج الـ API وملف إنتاج DbMigrator:

```powershell
notepad '.\src\Al7.TheqaTech.HttpApi.Host\appsettings.Production.json'
notepad '.\src\Al7.TheqaTech.DbMigrator\appsettings.Production.json'
```

ضع نفس قسم `SecretProtection` في الملفين. هذا مثال بمسار TheqaTech؛ **استبدل البصمة الظاهرة ببصمة الشهادة الجديدة من الخطوة 3**:

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

**مهم:** هذا القسم يُضاف إلى بقية إعدادات الملف، ولا تستبدل محتوى الملف كله بهذا المثال. في JSON نكتب `\\` لكل `\` في مسار Windows. يجب أن يتطابق اسم التطبيق و`Production` ومسار الملف وبصمة الشهادة مع الملف المشفّر.

بعد كتابة البصمة في الملفين، تأكد أنها تطابق الشهادة التي ستشفّر بها:

```powershell
$publicCert = [System.Security.Cryptography.X509Certificates.X509Certificate2]::new('C:\SecureWork\host-public.cer')
$apiThumbprint = (Get-Content '.\src\Al7.TheqaTech.HttpApi.Host\appsettings.Production.json' -Raw | ConvertFrom-Json).SecretProtection.CertificateThumbprint
$migratorThumbprint = (Get-Content '.\src\Al7.TheqaTech.DbMigrator\appsettings.Production.json' -Raw | ConvertFrom-Json).SecretProtection.CertificateThumbprint
if ($apiThumbprint -ne $publicCert.Thumbprint -or $migratorThumbprint -ne $publicCert.Thumbprint) { throw 'بصمة الشهادة لا تطابق إعدادات الإنتاج' }
```

## 8. انشر الـ API وDbMigrator — المطوّر

انشر كل تطبيق في **مجلد جديد فارغ**:

```powershell
dotnet publish '.\src\Al7.TheqaTech.HttpApi.Host\Al7.TheqaTech.HttpApi.Host.csproj' -c Release -o 'C:\SecureWork\publish-api'
dotnet publish '.\src\Al7.TheqaTech.DbMigrator\Al7.TheqaTech.DbMigrator.csproj' -c Release -o 'C:\SecureWork\publish-migrator'
```

افحص ناتج نشر الـ API:

```powershell
Test-Path 'C:\SecureWork\publish-api\appsettings.json'
Test-Path 'C:\SecureWork\publish-api\appsettings.secrets.json'
Test-Path 'C:\SecureWork\publish-api\openiddict.pfx'
Test-Path 'C:\SecureWork\publish-api\host.secrets.production.enc'
Select-String -Path 'C:\SecureWork\publish-api\web.config' -Pattern 'fileExtension=".pfx" allowed="false"', 'fileExtension=".enc" allowed="false"'
```

أول `Test-Path` فقط يجب أن يكون `True`، والباقي `False`. يجب أن يظهر سطران من `web.config` لمنع تنزيل `.pfx` و`.enc`. افتح `publish-api\appsettings.json` وتأكد أن قيم الأسرار فارغة أو غير موجودة. ارفع **محتويات** `publish-api` إلى موقع الـ API في Plesk. احتفظ بـ DbMigrator خارج مجلد الموقع.

## 9. ضع الملف المشفّر على السيرفر — مسؤول السيرفر IT

أنشئ مجلدًا مؤقتًا على السيرفر:

```powershell
New-Item -ItemType Directory -Path 'C:\SecureWork' -Force | Out-Null
```

انقل `host.secrets.production.enc` من جهاز المطوّر إلى `C:\SecureWork` على السيرفر بوسيلة نقل الملفات المعتمدة، ثم نفّذ:

```powershell
$file = 'C:\inetpub\vhosts\thiqatech.com.sa\api.thiqatech.com.sa\host.secrets.production.enc'
Copy-Item -LiteralPath 'C:\SecureWork\host.secrets.production.enc' -Destination $file -Force
Test-Path -LiteralPath $file -PathType Leaf
(Get-FileHash -LiteralPath $file -Algorithm SHA256).Hash
```

يجب أن تظهر `True`، وأن يطابق `SHA256` القيمة من الخطوة 6. امنح حساب تشغيل الـ API حق قراءة الملف. الأمر يسألك عن **اسم الحساب الحقيقي** بدل افتراض اسم Application Pool:

```powershell
$apiAccount = Read-Host 'اكتب حساب تشغيل TheqaTech API كما يظهر في IIS/Plesk'
icacls 'C:\inetpub\vhosts\thiqatech.com.sa\api.thiqatech.com.sa\host.secrets.production.enc' /grant "${apiAccount}:R"
```

إذا شغّلت DbMigrator على السيرفر بحساب مختلف، امنح ذلك الحساب قراءة الملف أيضًا. ضَع `openiddict.pfx` في مجلد الـ API الرئيسي بصلاحيات محدودة؛ هذه شهادة مختلفة عن شهادة تشفير الأسرار. لا تضع أي ملف أسرار غير مشفّر داخل مجلد الموقع.

## 10. شغّل التطبيق وافحصه — مسؤول السيرفر IT

اضبط بيئة الـ API في Plesk/IIS على `ASPNETCORE_ENVIRONMENT=Production`. إذا توجد تغييرات قاعدة بيانات، ضع ملفات `publish-migrator` خارج مجلد الموقع، وخذ نسخة احتياطية من قاعدة البيانات، ثم شغّله بالحساب الذي حصل على صلاحيات الخطوتين 4 و9:

```powershell
Set-Location 'C:\SecureWork\publish-migrator'
$env:DOTNET_ENVIRONMENT = 'Production'
dotnet '.\Al7.TheqaTech.DbMigrator.dll'
```

بعد ذلك شغّل أو أعد تشغيل موقع الـ API من Plesk/IIS. تأكد من المسار والبصمة اللذين يقرأهما التطبيق:

```powershell
(Get-Content 'C:\inetpub\vhosts\thiqatech.com.sa\api.thiqatech.com.sa\appsettings.Production.json' -Raw | ConvertFrom-Json).SecretProtection | Format-List
```

جرّب فتح الـ API وعملية تصل إلى قاعدة البيانات. التطبيق يفك تشفير `.enc` **عند بدء التشغيل** باستخدام المفتاح الخاص على السيرفر؛ المطوّر لا يحتاج أمرًا لفك التشفير.

## 11. عند تغيير كلمة مرور أو token لاحقًا

عدّل `C:\SecureWork\production-secrets.json` على جهاز المطوّر، وأعد تنفيذ أمر التشفير في الخطوة 6 باستخدام نفس `.cer`. ارفع ملف `.enc` الجديد، قارن `SHA256`، ثم أعد تشغيل الـ API. مثال أمر فحص الملف بعد التحديث:

```powershell
(Get-FileHash -LiteralPath 'C:\SecureWork\host.secrets.production.enc' -Algorithm SHA256).Hash
```

إذا ظهر خطأ `Protected secrets file is missing`، افحص المسار و`Test-Path`. إذا ظهر `Protected secrets envelope was rejected`، افحص البصمة، واسم التطبيق، و`Production`، وتطابق الملف المنقول.
