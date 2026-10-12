# حماية الأسرار: إعداد الفريق والنشر على بيئة الإنتاج

استخدم هذا الدليل لتطبيق آلية تشفير الأسرار في هذا الحل. يجهّز المطور ملفًا مشفرًا باستخدام الشهادة **العامة** للخادم، ثم تستخدم واجهة الـAPI في بيئة الإنتاج المفتاح **الخاص** المطابق على الخادم لقراءة الملف. تبقى قيم الأسرار داخل ملف `appsettings.json` المحلي أثناء التطوير، بينما تحصل النسخة المنشورة من الـAPI عليها من الملف المشفر.

نفّذ الأوامر باستخدام **PowerShell** ما لم تنص الخطوة على استخدام **Command Prompt**. تستخدم جميع الأمثلة التالية أسماء مشاريع TheqaTech الحقيقية، ومسار الخادم، ومعرّف التطبيق، وبصمة الشهادة. عند استخدام فريق آخر لهذا الكود، يجب عليه استبدال هذه القيم بقيم مشروعه. لا تضع كلمات مرور أو رموز وصول حقيقية داخل هذا الدليل أو Git أو ملف النشر المضغوط أو معاملات سطر الأوامر.

**بيئة إنتاج TheqaTech الحالية مجهّزة بالفعل:** بصمة الشهادة هي `5ACECE1EBA79CEF5FA7118165CB7B93AD319138E`، والملف المشفر موجود في المسار `C:\inetpub\vhosts\thiqatech.com.sa\api.thiqatech.com.sa\host.secrets.production.enc`. لا تنشئ شهادة جديدة لهذا الموقع العامل. الخطوتان 2 و3 مخصصتان لخادم أو مشروع جديد، وتوضح الخطوات اللاحقة أيضًا كيفية تحديث الموقع الحالي مستقبلًا.

## 1. ربط مشروع Host جديد بالكود المشترك — مرة واحدة لكل مشروع

واجهة الـAPI الحالية وDbMigrator مرتبطان بالفعل بالكود المشترك. عند إضافة مشروع آخر إلى الحل:

1. أضف مرجعًا إلى المشروع `src/Al7.TheqaTech.SecretProtection/Al7.TheqaTech.SecretProtection.csproj` داخل كل مشروع تنفيذي يحتاج إلى قراءة الأسرار.
2. أضف `using Al7.TheqaTech.SecretProtection;` عند بدء التشغيل، ثم استدعِ `config.AddTheqaTechProtectedSecrets()` داخل `ConfigureAppConfiguration`، كما هو موضح في [API Program.cs](../src/Al7.TheqaTech.HttpApi.Host/Program.cs) و[DbMigrator Program.cs](../src/Al7.TheqaTech.DbMigrator/Program.cs).
3. أضف معرّف التطبيق الجديد ومفاتيح الإعدادات **المسموح بها** و**المطلوبة** في [ProtectedSecretCatalog.cs](../src/Al7.TheqaTech.SecretProtection/ProtectedSecretCatalog.cs). ترفض أداة CLI أي معرّفات أو مفاتيح غير مسجلة. هذا تعديل برمجي وليس إعدادًا خارجيًا.
4. احتفظ بالأسرار المحلية داخل `appsettings.json` الخاص بالمشروع الجديد. أنشئ ملف نشر أساسيًا باسم `appsettings.json` لا يحتوي على أسرار، باتباع [ملف مشروع الـAPI](../src/Al7.TheqaTech.HttpApi.Host/Al7.TheqaTech.HttpApi.Host.csproj) و[appsettings.publish.json](../src/Al7.TheqaTech.HttpApi.Host/appsettings.publish.json). أضف إعداد إنتاج يفعّل `SecretProtection` كما في الخطوة 6.

الأمر التالي ينفذ البند الأول باستخدام مسارات مشروع TheqaTech الفعلية. المرجع موجود بالفعل في هذا الفرع، لذلك لا يحتاج فريق TheqaTech إلى تشغيله مرة أخرى. عند استخدامه في فريق آخر، يجب تغيير المسار الأول إلى مشروع الـHost الخاص به:

```powershell
dotnet add '.\src\Al7.TheqaTech.HttpApi.Host\Al7.TheqaTech.HttpApi.Host.csproj' reference '.\src\Al7.TheqaTech.SecretProtection\Al7.TheqaTech.SecretProtection.csproj'
```

بالنسبة إلى **واجهة TheqaTech API الحالية**، استخدم معرّف التطبيق `TheqaTech.HttpApi.Host` ونفّذ الأوامر التالية دون تعديل كتالوج المفاتيح. المفاتيح المحمية هي:

- `ConnectionStrings:Default`
- `AuthServer:CertificatePassPhrase`
- `StringEncryption:DefaultPassPhrase`
- `Swagger:Password`
- `Firebase:CredentialsJson`
- `HyperPay:AccessToken`
- `HyperSplit:Password`
- `HyperSplit:NotificationKey`
- `SiteAccess:Turnstile:SecretKey`

المفاتيح الخمسة الأولى مطلوبة حاليًا ويجب ألا تكون فارغة. إعداد Firebase خارج نطاق هذا الدليل، لكن **لا تحذف مفتاحه المطلوب** من الحمولة إلا بعد تعديل الكتالوج وكود الـAPI. وضع قيمة تجريبية غير مستخدمة يسمح ببدء التطبيق فقط، ولا يجعل استدعاءات Firebase تعمل. يمكن حذف المفاتيح الاختيارية من الحمولة.

## 2. إنشاء الشهادة على خادم Windows في الإنتاج — ينفذه فريق IT مرة واحدة

افتح **PowerShell بصلاحية Administrator** على الخادم. أنشئ المجلد قبل تصدير الشهادة، ثم نفّذ الأسطر التالية بالترتيب:

```powershell
New-Item -ItemType Directory -Path 'C:\SecureWork' -Force | Out-Null
$cert = New-SelfSignedCertificate -Type Custom -Subject 'CN=TheqaTech-Host-Secrets' -KeyAlgorithm RSA -KeyLength 3072 -KeyUsage KeyEncipherment -KeyExportPolicy NonExportable -CertStoreLocation 'Cert:\LocalMachine\My'
if ($null -eq $cert) { throw 'Certificate creation failed' }
Export-Certificate -Cert $cert -FilePath 'C:\SecureWork\host-public.cer'
$cert.Thumbprint
```

يرسل فريق IT ملف `host-public.cer` الناتج وبصمة الشهادة المطبوعة إلى المطور عبر القناة المعتمدة. **الملف العام `.cer` فقط هو الذي يغادر الخادم.** يبقى المفتاح الخاص داخل `LocalMachine\My`. بصمة الشهادة مجرد معرّف وليست كلمة مرور. لا تستخدم ملف `openiddict.pfx` الخاص بالـAPI لهذا الغرض.

يمكن لفريق IT التحقق من الشهادة لاحقًا دون إنشاء شهادة جديدة:

```powershell
$thumbprint = '5ACECE1EBA79CEF5FA7118165CB7B93AD319138E'
Get-Item "Cert:\LocalMachine\My\$thumbprint" | Select-Object Thumbprint, HasPrivateKey
```

## 3. منح التطبيق صلاحية قراءة المفتاح الخاص — مرة واحدة

شغّل `certlm.msc` على الخادم. افتح **Personal > Certificates**، وابحث عن شهادة TheqaTech بواسطة البصمة، ثم اضغط عليها بزر الفأرة الأيمن واختر **All Tasks > Manage Private Keys**. امنح صلاحية **Read** لحساب Windows الفعلي الذي يشغّل TheqaTech API.

يحصل فريق IT على اسم الحساب من إعدادات Application Pool في IIS أو Plesk؛ الاسم الفعلي غير مسجل داخل المستودع. إذا كان DbMigrator سيعمل على الخادم، امنح صلاحية **Read** أيضًا للحساب الذي سيشغّله. لا تمنح الصلاحية إلى `Everyone`.

يحتاج كل من الـAPI وDbMigrator أيضًا إلى صلاحية **Read** على الملف المشفر. بعد وجود الملف كما في الخطوة 8، يمكن لفريق IT منح الصلاحية باستخدام:

```powershell
$apiAccount = Read-Host 'Enter the exact TheqaTech API Windows account shown in IIS/Plesk'
icacls 'C:\inetpub\vhosts\thiqatech.com.sa\api.thiqatech.com.sa\host.secrets.production.enc' /grant "${apiAccount}:R"
```

إذا كان IIS يستخدم `ApplicationPoolIdentity`، أدخل `IIS AppPool\` متبوعًا بالاسم الفعلي للـApplication Pool. وإذا كان يستخدم هوية مخصصة، أدخل حساب Windows المحدد. امنح حساب DbMigrator صلاحية قراءة الملف أيضًا إذا كان سيعمل على الخادم. لا يحتاج حساب التطبيق إلى صلاحية كتابة.

## 4. تجهيز حمولة الأسرار غير المشفرة محليًا — ينفذه المطور

من **المجلد الرئيسي للحل**، أنشئ مجلد عمل خاصًا وانسخ ملف المثال:

```powershell
New-Item -ItemType Directory -Path 'C:\SecureWork' -Force | Out-Null
Copy-Item -LiteralPath '.\docs\secret-protection-payload.example.json' -Destination 'C:\SecureWork\production-secrets.json'
notepad 'C:\SecureWork\production-secrets.json'
```

بالنسبة إلى الـAPI الحالية، استبدل قيم المثال بأسرار الإنتاج الفعلية. بالنسبة إلى مشروع آخر، أنشئ ملف JSON يحتوي فقط على مفاتيح المشروع المسجلة في `ProtectedSecretCatalog.cs`؛ مثال TheqaTech لن يكون مناسبًا له. تُرفض المفاتيح الإضافية.

احتفظ بالقيمة الحالية لـ`StringEncryption:DefaultPassPhrase` إذا كانت قاعدة البيانات تحتوي بالفعل على إعدادات ABP مشفرة. أبقِ ملف النص الصريح خارج المستودع وخارج موقع الويب، ولا ترسله إلى فريق IT. تحقّق من صحة JSON دون عرض القيم:

```powershell
Get-Content 'C:\SecureWork\production-secrets.json' -Raw | ConvertFrom-Json | Out-Null
```

## 5. بناء أداة التشفير وإنشاء الملف المشفر — ينفذه المطور

ضع ملف `host-public.cer` الذي استلمته من فريق IT في المسار `C:\SecureWork\host-public.cer`. من **المجلد الرئيسي للحل**، انشر أداة CLI:

```powershell
dotnet publish '.\src\Al7.TheqaTech.SecretProtection.Cli\Al7.TheqaTech.SecretProtection.Cli.csproj' -c Release -o 'C:\SecureWork\secret-cli'
```

افتح **Command Prompt (`cmd.exe`)** لتنفيذ الأمر التالي. يستخدم CMD الرمز `<` لتمرير ملف JSON إلى الأداة على هيئة bytes. غيّر قيمة `--application` إذا كانت الخطوة 1 قد سجلت معرّف تطبيق مختلفًا:

```cmd
dotnet "C:\SecureWork\secret-cli\Al7.TheqaTech.SecretProtection.Cli.dll" encrypt --certificate "C:\SecureWork\host-public.cer" --application "TheqaTech.HttpApi.Host" --environment "Production" --output "C:\SecureWork\host.secrets.production.enc" < "C:\SecureWork\production-secrets.json"
```

عند النجاح تظهر الرسالة `Encrypted secrets file written.`. ملف `.enc` الناتج هو الملف الذي يجب رفعه. تستخدم الشهادة العامة `.cer` للتشفير، ولا يحتاج المطور إلى المفتاح الخاص الموجود على الخادم. ينتج كل تشغيل للتشفير ملفًا مشفرًا جديدًا حتى إذا لم تتغير الحمولة.

تحقّق من الملف وسجّل قيمة الـhash باستخدام **PowerShell**:

```powershell
Test-Path -LiteralPath 'C:\SecureWork\host.secrets.production.enc' -PathType Leaf
(Get-FileHash -LiteralPath 'C:\SecureWork\host.secrets.production.enc' -Algorithm SHA256).Hash
```

## 6. إعداد بيئة الإنتاج — ينفذه المطور

ضع القيم التالية في ملف `appsettings.Production.json` الخاص بكل عملية ستقرأ الملف. استخدم البصمة من الخطوة 2 ومسار الوجهة الدقيق من الخطوة 8. داخل JSON يجب كتابة كل `\` في مسار Windows بالشكل `\\`:

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

ملفا إنتاج الـAPI وDbMigrator الحاليان موجودان [هنا](../src/Al7.TheqaTech.HttpApi.Host/appsettings.Production.json) و[هنا](../src/Al7.TheqaTech.DbMigrator/appsettings.Production.json). يستخدم كلاهما معرّف التطبيق والملف المشفر نفسيهما.

بالنسبة إلى مشروع آخر، يجب أن تتطابق القيم الأربع مع ملفه المشفر: معرّف التطبيق، والبيئة، وبصمة الشهادة، ومسار الملف. أبقِ ملف `appsettings.json` الأساسي المنشور خاليًا من الأسرار النصية. تبقى القيم العادية مثل عناوين URL ومفاتيح الموقع العامة داخل الإعدادات. عند تشغيل الدفع الحقيقي، ضع عناوين الدفع ومعرّفات الكيانات الخاصة بالإنتاج هناك، بينما يبقى رمز الوصول داخل الحمولة المشفرة.

## 7. نشر التطبيقات — ينفذه المطور

من المجلد الرئيسي للحل، استخدم **مجلدات إخراج جديدة وفارغة** لكل إصدار. استبدل مسارات المشاريع عند العمل على Host آخر:

```powershell
dotnet publish '.\src\Al7.TheqaTech.HttpApi.Host\Al7.TheqaTech.HttpApi.Host.csproj' -c Release -o 'C:\SecureWork\publish-api-new'
dotnet publish '.\src\Al7.TheqaTech.DbMigrator\Al7.TheqaTech.DbMigrator.csproj' -c Release -o 'C:\SecureWork\publish-migrator-new'
```

افحص مجلد نشر الـAPI قبل إرساله إلى فريق IT:

```powershell
Test-Path 'C:\SecureWork\publish-api-new\appsettings.json'
Test-Path 'C:\SecureWork\publish-api-new\appsettings.secrets.json'
Test-Path 'C:\SecureWork\publish-api-new\openiddict.pfx'
Test-Path 'C:\SecureWork\publish-api-new\host.secrets.production.enc'
```

يجب أن تكون نتيجة الفحص الأول فقط `True`. افتح ملف `appsettings.json` المنشور وتأكد أن القيم المحمية فارغة أو غير موجودة. تحقّق أيضًا من أن ملف `web.config` المنشور يمنع الوصول إلى ملفات `.pfx` و`.enc`:

```powershell
Select-String -Path 'C:\SecureWork\publish-api-new\web.config' -Pattern 'fileExtension=".pfx" allowed="false"', 'fileExtension=".enc" allowed="false"'
```

يجب أن يظهر السطران. ارفع **محتويات مجلد نشر الـAPI** إلى موقع الـAPI على Plesk/IIS. أبقِ DbMigrator وأداة CLI خارج مجلد الويب. يتم توفير `openiddict.pfx` وملف `.enc` بصورة منفصلة.

## 8. وضع الملف المشفر على الخادم — ينفذه فريق IT

ينشئ فريق IT أولًا مجلدًا مؤقتًا على الخادم:

```powershell
New-Item -ItemType Directory -Path 'C:\SecureWork' -Force | Out-Null
```

انقل **ملف `C:\SecureWork\host.secrets.production.enc` فقط** إلى المسار نفسه على الخادم باستخدام وسيلة نقل الملفات المعتمدة لدى الفريق. بعد ذلك ينسخه فريق IT إلى قيمة `FilePath` المحددة في الخطوة 6. في موقع TheqaTech الحالي، المسار هو `C:\inetpub\vhosts\thiqatech.com.sa\api.thiqatech.com.sa\host.secrets.production.enc`.

على الخادم، يتحقق فريق IT من الوجهة ويقارن قيمة الـhash بالقيمة التي سجلها المطور في الخطوة 5:

```powershell
$file = 'C:\inetpub\vhosts\thiqatech.com.sa\api.thiqatech.com.sa\host.secrets.production.enc'
Copy-Item -LiteralPath 'C:\SecureWork\host.secrets.production.enc' -Destination $file -Force
Test-Path -LiteralPath $file -PathType Leaf
(Get-FileHash -LiteralPath $file -Algorithm SHA256).Hash
```

يجب أن تكون النتيجة الأولى `True` وأن تتطابق قيمتا الـhash. بعد ذلك طبّق صلاحية الملف الموضحة في الخطوة 3. إذا كانت نتيجة `Test-Path` هي `False`، صحّح وجهة الرفع أو قيمة `FilePath` قبل تشغيل الـAPI.

## 9. تشغيل API الإنتاج وتنفيذ الترحيلات عند الحاجة — ينفذه فريق IT

في Plesk/IIS، اضبط بيئة الـAPI على `Production` باستخدام `ASPNETCORE_ENVIRONMENT=Production`، ثم أعد تدوير الموقع أو شغّله. احتفظ بملف `openiddict.pfx` الحالي داخل مجلد محتوى الـAPI مع تقييد صلاحيات الوصول؛ فهو شهادة مختلفة.

احذف أي ملفات قديمة تحتوي على أسرار نصية، ومنها `appsettings.secrets.json` والملفات القديمة داخل `wwwroot`. نسخ مجلد نشر جديد لا يحذف الملفات القديمة تلقائيًا.

إذا احتوى الإصدار على تغييرات في قاعدة البيانات، ضع **مجلد نشر DbMigrator خارج موقع الويب**. شغّله على الخادم تحت حساب Windows الذي حصل على صلاحية قراءة الشهادة والملف في الخطوة 3:

```powershell
Set-Location 'C:\SecureWork\publish-migrator-new'
$env:DOTNET_ENVIRONMENT = 'Production'
dotnet '.\Al7.TheqaTech.DbMigrator.dll'
```

أنشئ نسخة احتياطية من قاعدة البيانات المستهدفة وتأكد أنها قاعدة البيانات المطلوبة قبل تنفيذ الترحيلات. يمكن للمطور الاستمرار في تشغيل DbMigrator محليًا من Visual Studio باستخدام الإعدادات المحلية؛ وهذه عملية منفصلة عن أمر الإنتاج السابق.

## 10. التحقق والتحديثات المستقبلية

على الخادم، تأكد أن إعدادات الإنتاج تشير إلى الملف الذي تم رفعه فعليًا:

```powershell
(Get-Content 'C:\inetpub\vhosts\thiqatech.com.sa\api.thiqatech.com.sa\appsettings.Production.json' -Raw | ConvertFrom-Json).SecretProtection | Format-List
```

بعد ذلك افتح الـAPI، وافحص سجل بدء التشغيل للتأكد من النجاح، واختبر عملية تستخدم أحد الأسرار المحمية، مثل الاتصال بقاعدة البيانات.

عند تغيير كلمة مرور أو رمز وصول مستقبلًا:

1. عدّل حمولة النص الصريح المحلية.
2. أعد تنفيذ الخطوة 5 باستخدام ملف `.cer` العام نفسه.
3. ارفع ملف `.enc` الجديد.
4. قارن قيمتي الـhash.
5. أعد تدوير الـAPI.

لا يوجد أمر فك تشفير متاح للمطور؛ يفك التطبيق التشفير عند بدء التشغيل باستخدام المفتاح الخاص الموجود على الخادم.

| الخطأ | أول ما يجب التحقق منه |
| --- | --- |
| `Protected secrets file is missing` | قيمة `FilePath` ونتيجة `Test-Path` في الخطوة 8. |
| `Protected secrets envelope was rejected` | قيمة hash الصحيحة لملف `.enc`، ومعرّف التطبيق، وبيئة `Production`، وبصمة الشهادة، والمفاتيح المطلوبة في الحمولة. |
| Certificate/private key unavailable | وجود الشهادة داخل `LocalMachine\My` ومنح حساب العملية صلاحية **Read** على المفتاح الخاص. |

المراجع الرسمية الخاصة بـWindows وIIS لفريق IT:

- [New-SelfSignedCertificate](https://learn.microsoft.com/en-us/powershell/module/pki/new-selfsignedcertificate)
- [Application Pool Identities](https://learn.microsoft.com/en-us/iis/manage/configuring-security/application-pool-identities)
- [نشر ASP.NET Core على IIS](https://learn.microsoft.com/en-us/aspnet/core/tutorials/publish-to-iis)
