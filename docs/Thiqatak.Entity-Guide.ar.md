# دليل كيانات منصة ثقة تك

هذا الملف يشرح وظيفة كل كيان وعلاقاته الأساسية.

الجمل مختصرة.

## قواعد مهمة

- شركة التأمين سجل مرجعي.
- بياناتها المعروضة هي الاسم والصورة فقط.
- كود الربط والمعرّف حقول تقنية.
- بيانات العروض تأتي من API خارجي.
- العرض المحلي نسخة ثابتة من رد الـAPI.
- الأسعار والتغطيات لا تُعدّل محليًا.

## 1. الهوية والأطراف

| الكيان | الاستخدام | العلاقات |
|---|---|---|
| `PARTIES` | يمثل شخصًا أو منشأة بشكل موحد. ويحفظ بيانات الشخص عندما يكون النوع `Person`. | أصل لـ`ORGANIZATIONS`. ويرتبط بالعملاء والسائقين والأعضاء وبيانات الاتصال والعناوين والحسابات. |
| `CLIENTS` | يمثل العميل الفرد أو المنشأة. | يرتبط بـ`PARTIES`. وقد يرتبط بحساب مستخدم ABP. |
| `ORGANIZATIONS` | يحفظ بيانات المنشأة القانونية. | يتخصص من`PARTIES`. ويرتبط بجهات التمويل والعضويات والأساطيل. |
| `FUNDERS` | يمثل جهة التمويل المستأجرة للمنصة. | يرتبط بـ`ORGANIZATIONS` و`FUNDER_SETTINGS` والعقود والتجديدات. |
| `PARTY_CONTACTS` | يحفظ الجوال والبريد وواتساب. | يتبع `PARTIES`. |
| `PARTY_ADDRESSES` | يحفظ العنوان الوطني والمدينة. | يتبع `PARTIES`. |
| `PARTY_BANK_ACCOUNTS` | يحفظ الآيبان وحالة التحقق. | يتبع `PARTIES`. ويُستخدم للاسترداد. |
| `PARTY_CONSENTS` | يسجل موافقات العميل القانونية. | يتبع `PARTIES`. وتستخدمه `QUOTE_CONSENTS`. |
| `IDENTITY_VERIFICATIONS` | يسجل نتيجة التحقق من الهوية. | يتبع `PARTIES`. ويرتبط بمزود تحقق خارجي. |

## 2. المنتجات وطلبات التسعير

| الكيان | الاستخدام | العلاقات |
|---|---|---|
| `INSURERS` | مرجع لشركة التأمين. يعرض الاسم والصورة فقط. | يرتبط بالعروض والوثائق والشبكات. |
| `INSURANCE_PRODUCTS` | يعرف أنواع منتجات التأمين. | يرتبط بالتغطيات والإضافات وطلبات التسعير. |
| `INSURER_PRODUCTS` | يربط كود منتج الـAPI بالشركة والمنتج. | يربط `INSURERS` مع`INSURANCE_PRODUCTS`. |
| `COVERAGE_DEFINITIONS` | يعرف أسماء التغطيات للعرض. | يتبع `INSURANCE_PRODUCTS`. ويصف `QUOTE_OFFER_COVERAGES`. |
| `ADDON_DEFINITIONS` | يعرف أنواع الخدمات الإضافية. | يتبع `INSURANCE_PRODUCTS`. ويصف `QUOTE_OFFER_ADDONS`. |
| `QUOTE_REQUESTS` | يمثل طلب تسعير واحد. | يرتبط بالعميل والمنتج والمخاطر والعروض والاختيار. |
| `QUOTE_RISK_ITEMS` | يحفظ عناصر الخطر المرسلة للتسعير. | يتبع `QUOTE_REQUESTS`. وقد يشير إلى مركبة أو شخص أو مجموعة. |
| `QUOTE_CONSENTS` | يثبت موافقات العميل داخل الطلب. | يربط `QUOTE_REQUESTS` مع`PARTY_CONSENTS`. |
| `QUOTE_OFFERS` | يحفظ نسخة ثابتة من عرض الـAPI. | يتبع `QUOTE_REQUESTS` و`INSURER_PRODUCTS`. وله تفاصيل وتغطيات وإضافات. |
| `QUOTE_OFFER_LINES` | يحفظ بنود السعر والضريبة والرسوم. | يتبع `QUOTE_OFFERS`. |
| `QUOTE_OFFER_COVERAGES` | يحفظ التغطيات التي أعادها الـAPI. | يربط `QUOTE_OFFERS` مع`COVERAGE_DEFINITIONS`. |
| `QUOTE_OFFER_ADDONS` | يحفظ الإضافات التي أعادها الـAPI. | يربط `QUOTE_OFFERS` مع`ADDON_DEFINITIONS`. |
| `QUOTE_SELECTIONS` | يسجل العرض الذي اختاره المستخدم. | يربط `QUOTE_REQUESTS` مع`QUOTE_OFFERS`. وينتج التوقيع والوثيقة. |
| `QUOTE_SIGNATURES` | يحفظ توقيع أو موافقة العميل. | يتبع `QUOTE_SELECTIONS`. ويرتبط بالطرف الموقّع. |
| `QUOTE_STATUS_HISTORY` | يسجل تغيرات حالة طلب التسعير. | يتبع `QUOTE_REQUESTS`. |
| `PRICING_SNAPSHOTS` | يجمد مدخلات ونتائج التسعير. | يتبع `QUOTE_REQUESTS`. ويستخدم للتدقيق. |

## 3. الوثائق والفوترة والمطالبات

| الكيان | الاستخدام | العلاقات |
|---|---|---|
| `POLICIES` | يمثل وثيقة تأمين صادرة. | ينتج من`QUOTE_SELECTIONS`. ويرتبط بالشركة والأطراف والأصول والفواتير والمطالبات. |
| `POLICY_PARTIES` | يحدد دور كل طرف في الوثيقة. | يربط `POLICIES` مع`PARTIES`. |
| `POLICY_ASSETS` | يسجل الأصول المشمولة بالوثيقة. | يتبع `POLICIES`. وقد يشير إلى مركبة أو أصل آخر. |
| `POLICY_COVERAGES` | يحفظ التغطيات النهائية للوثيقة. | يتبع `POLICIES`. |
| `POLICY_ADDONS` | يحفظ الخدمات الإضافية المشتراة. | يتبع `POLICIES`. وقد ينشئ شراء خدمة للمستأجر. |
| `POLICY_DOCUMENTS` | يحفظ بيانات ملفات الوثيقة والشهادات. | يتبع `POLICIES`. |
| `POLICY_STATUS_HISTORY` | يسجل تغير حالة الوثيقة. | يتبع `POLICIES`. |
| `POLICY_ENDORSEMENTS` | يمثل ملحق تعديل على وثيقة. | يتبع `POLICIES`. وله بنود وأعضاء طبيون متأثرون. |
| `ENDORSEMENT_LINES` | يحفظ تفاصيل الإضافة أو الحذف أو التعديل. | يتبع `POLICY_ENDORSEMENTS`. |
| `INVOICES` | يمثل فاتورة مستحقة. | يتبع `POLICIES`. ويرتبط بالبنود والدفعات والأقساط والإشعارات الدائنة. |
| `INVOICE_LINES` | يفصل عناصر الفاتورة. | يتبع `INVOICES`. |
| `PAYMENTS` | يسجل عملية دفع منطقية. | يسدد فاتورة أو أكثر. وله محاولات واستردادات واستثناءات. |
| `PAYMENT_ATTEMPTS` | يسجل كل محاولة مع بوابة الدفع. | يتبع `PAYMENTS`. ولا يخزن بيانات البطاقة. |
| `INSTALLMENTS` | يحدد جدول أقساط الفاتورة. | يتبع `INVOICES`. |
| `CREDIT_NOTES` | يسجل تخفيضًا أو رصيدًا دائنًا. | يتبع `INVOICES`. وقد ينتج عن إلغاء أو ملحق. |
| `REFUNDS` | يسجل استرداد مبلغ للعميل. | يتبع `PAYMENTS`. ويستخدم حساب الطرف البنكي. |
| `CLAIMS` | يمثل مطالبة على وثيقة. | يتبع `POLICIES`. وله مستندات وسجل حالات. |
| `CLAIM_DOCUMENTS` | يحفظ مرفقات المطالبة. | يتبع `CLAIMS`. |
| `CLAIM_STATUS_HISTORY` | يسجل مراحل معالجة المطالبة. | يتبع `CLAIMS`. |

## 4. تأمين المركبات والأساطيل

| الكيان | الاستخدام | العلاقات |
|---|---|---|
| `VEHICLE_MAKES` | مرجع لماركات المركبات. | يحتوي على`VEHICLE_MODELS`. |
| `VEHICLE_MODELS` | مرجع لموديلات المركبات. | يتبع `VEHICLE_MAKES`. ويرتبط بالمركبات وخرائط الأكواد. |
| `VEHICLE_CODE_MAPPINGS` | يربط أكواد المركبة بين الأنظمة الخارجية. | يتبع `VEHICLE_MODELS`. ويستخدم لحل اختلاف الأكواد. |
| `VEHICLES` | يمثل مركبة واحدة. | يرتبط بالموديل والتسجيلات والأطراف والعقود والوثائق. |
| `VEHICLE_REGISTRATIONS` | يحفظ بيانات اللوحة والتسجيل عبر الزمن. | يتبع `VEHICLES`. |
| `VEHICLE_PARTY_ROLES` | يحدد المالك والمستخدم والمستأجر. | يربط `VEHICLES` مع`PARTIES`. |
| `MOTOR_QUOTE_DETAILS` | يحفظ تفاصيل طلب تأمين مركبة فردية. | يتبع `QUOTE_REQUESTS`. ويرتبط بـ`VEHICLES` والسائقين. |
| `MOTOR_QUOTE_DRIVERS` | يحفظ السائقين ونسب القيادة داخل الطلب. | يتبع `MOTOR_QUOTE_DETAILS`. ويرتبط بطرف نوعه `Person`. |
| `FLEETS` | يمثل أسطول مركبات لمنشأة. | يتبع `ORGANIZATIONS`. ويحتوي مركبات الأسطول. |
| `FLEET_VEHICLES` | يحدد عضوية المركبة في الأسطول. | يربط `FLEETS` مع`VEHICLES`. |
| `FLEET_QUOTE_DETAILS` | يحفظ بيانات طلب تسعير الأسطول. | يتبع `QUOTE_REQUESTS`. ويحتوي عناصر التسعير. |
| `FLEET_QUOTE_ITEMS` | يحفظ تسعير كل مركبة في الأسطول. | يتبع `FLEET_QUOTE_DETAILS`. ويرتبط بـ`VEHICLES`. |
| `POLICY_VEHICLES` | يربط المركبات بالوثائق الصادرة. | يربط `POLICIES` مع`VEHICLES`. |

## 5. التأمين الطبي للمنشآت

| الكيان | الاستخدام | العلاقات |
|---|---|---|
| `ORGANIZATION_MEMBERS` | يمثل موظفًا أو تابعًا داخل المنشأة. | يربط `ORGANIZATIONS` مع طرف نوعه `Person`. وقد يشير إلى عضو تابع. |
| `MEDICAL_PLAN_CLASSES` | يعرف فئات التغطية الطبية. | يرتبط بالمزايا والأعضاء والشبكات. |
| `MEDICAL_CLASS_BENEFITS` | يحدد مزايا كل فئة طبية. | يتبع `MEDICAL_PLAN_CLASSES`. |
| `MEDICAL_QUOTE_DETAILS` | يحفظ بيانات طلب التأمين الطبي. | يتبع `QUOTE_REQUESTS`. ويحتوي الأعضاء والإفصاح. |
| `MEDICAL_QUOTE_MEMBERS` | يحفظ كل عضو مطلوب تسعيره وفئته. | يربط `MEDICAL_QUOTE_DETAILS` مع`ORGANIZATION_MEMBERS` و`MEDICAL_PLAN_CLASSES`. |
| `DISCLOSURE_QUESTIONS` | يعرف أسئلة الإفصاح الطبي. | ترتبط بها`DISCLOSURE_ANSWERS`. |
| `MEDICAL_DISCLOSURES` | يمثل نموذج إفصاح لطلب طبي. | يتبع `MEDICAL_QUOTE_DETAILS`. وله إجابات وقرار اكتتاب. |
| `DISCLOSURE_ANSWERS` | يحفظ إجابة سؤال طبي. | يربط `MEDICAL_DISCLOSURES` مع`DISCLOSURE_QUESTIONS`. |
| `DISCLOSURE_PERSONS` | يحدد الأشخاص المتأثرين بالإجابة. | يربط `DISCLOSURE_ANSWERS` مع طرف نوعه `Person`. |
| `UNDERWRITING_DECISIONS` | يحفظ قبول أو رفض أو تحميل طبي. | يتبع `MEDICAL_DISCLOSURES`. |
| `MEDICAL_POLICY_MEMBERS` | يسجل الأعضاء المؤمن عليهم فعليًا. | يربط `POLICIES` مع طرف نوعه `Person` ومع `MEDICAL_PLAN_CLASSES`. |
| `MEDICAL_PROVIDERS` | نسخة مرجعية لمقدم خدمة طبية خارجي. | يدخل في عضوية الشبكات الطبية. |
| `MEDICAL_NETWORKS` | نسخة شبكة طبية منشورة من الشركة. | تتبع `INSURERS`. وترتبط بالفئات ومقدمي الخدمة. |
| `NETWORK_CLASS_ACCESS` | يحدد الفئات المسموح لها باستخدام الشبكة. | يربط `MEDICAL_NETWORKS` مع`MEDICAL_PLAN_CLASSES`. |
| `NETWORK_PROVIDER_MEMBERSHIPS` | يحدد مقدمي الخدمة داخل الشبكة. | يربط `MEDICAL_NETWORKS` مع`MEDICAL_PROVIDERS`. |
| `MEDICAL_ENDORSEMENT_MEMBERS` | يسجل أعضاء الملحق الطبي. | يربط `POLICY_ENDORSEMENTS` مع طرف نوعه `Person`. |

## 6. التمويل والتأجير

| الكيان | الاستخدام | العلاقات |
|---|---|---|
| `FUNDER_SETTINGS` | يحفظ إعدادات بوابة جهة التمويل. | له علاقة واحد لواحد مع`FUNDERS`. |
| `FUNDER_INSURERS` | يحدد الشركات المتاحة لجهة التمويل. | يربط `FUNDERS` مع`INSURERS`. |
| `FINANCING_CONTRACTS` | يمثل عقد تمويل لمركبة ومستأجر. | يتبع `FUNDERS`. ويرتبط بـ`PARTIES` و`VEHICLES`. |
| `LEASE_QUOTE_DETAILS` | يحفظ تفاصيل تسعير عقد التأجير. | يتبع `QUOTE_REQUESTS` و`FINANCING_CONTRACTS`. |
| `LEASE_YEAR_PROJECTIONS` | يحفظ القيمة والتغطية المتوقعة لكل سنة. | يتبع `LEASE_QUOTE_DETAILS`. |
| `LEASE_OFFER_YEARS` | يحفظ سعر كل سنة داخل عرض التأجير. | يتبع `QUOTE_OFFERS`. |
| `CONTRACT_POLICY_YEARS` | يربط سنة العقد بوثيقتها. | يربط `FINANCING_CONTRACTS` مع`POLICIES`. |
| `INSURANCE_COLLECTIONS` | يسجل مبالغ التأمين المحصلة مع العقد. | يتبع `FINANCING_CONTRACTS`. |
| `RENEWAL_BATCHES` | يمثل دفعة تجديد جماعية. | يتبع `FUNDERS`. ويحتوي عناصر التجديد. |
| `RENEWAL_ITEMS` | يمثل عقدًا أو مركبة داخل دفعة التجديد. | يتبع `RENEWAL_BATCHES`. ويرتبط بـ`FINANCING_CONTRACTS`. |
| `RENEWAL_ITEM_OFFERS` | يحفظ عروض التجديد القادمة من الـAPI. | يتبع `RENEWAL_ITEMS`. ويرتبط بـ`INSURERS`. |
| `RENEWAL_APPROVALS` | يسجل موافقات جهة التمويل على التجديد. | يتبع `RENEWAL_ITEMS`. |
| `LESSEE_SERVICE_PURCHASES` | يسجل شراء المستأجر لخدمة إضافية. | يتبع `FINANCING_CONTRACTS`. وقد ينتج من`POLICY_ADDONS`. |

## 7. العمليات والتكامل والدعم

| الكيان | الاستخدام | العلاقات |
|---|---|---|
| `OPERATIONAL_EXCEPTIONS` | يسجل مشكلة تحتاج تدخلًا يدويًا. | قد يرتبط بطلب أو وثيقة أو دفعة. وله إجراءات حل. |
| `EXCEPTION_ACTIONS` | يسجل خطوات معالجة الاستثناء. | يتبع `OPERATIONAL_EXCEPTIONS`. |
| `SUPPORT_TICKETS` | يمثل تذكرة دعم لعميل أو منشأة. | يتبع `PARTIES`. وله أنشطة وروابط. |
| `TICKET_ACTIVITIES` | يسجل التعليقات والتعيين وتغير الحالة. | يتبع `SUPPORT_TICKETS`. |
| `TICKET_LINKS` | يربط التذكرة بسجل أعمال. | يتبع `SUPPORT_TICKETS`. وقد يشير إلى طلب أو وثيقة أو دفعة. |
| `INTEGRATION_PROVIDERS` | يعرف مزود تكامل خارجي. | يحتوي نقاط الاتصال والحوادث والاشتراكات. |
| `INTEGRATION_ENDPOINTS` | يعرف عملية API عند المزود. | يتبع `INTEGRATION_PROVIDERS`. وله سجلات استدعاء. |
| `INTEGRATION_CALLS` | يسجل كل استدعاء API ومدة الاستجابة. | يتبع `INTEGRATION_ENDPOINTS`. ويرتبط بالسجل التجاري المتأثر. |
| `INTEGRATION_INCIDENTS` | يسجل عطلًا عند مزود خارجي. | يتبع `INTEGRATION_PROVIDERS`. وله إشعارات حادث. |
| `INCIDENT_NOTIFICATIONS` | يسجل من تم إشعاره بالعطل. | يتبع `INTEGRATION_INCIDENTS`. |
| `INTEGRATION_SUBSCRIPTIONS` | يحفظ العقد أو الاشتراك مع مزود التكامل. | يتبع `INTEGRATION_PROVIDERS`. |
| `NOTIFICATIONS` | يمثل رسالة موجهة لطرف. | يتبع `PARTIES`. وله محاولات توصيل. |
| `NOTIFICATION_DELIVERIES` | يسجل إرسال الرسالة عبر قناة محددة. | يتبع `NOTIFICATIONS`. |
| `PROMOTIONS` | يعرف حملة محلية للمنصة فقط. | يتبع `INSURANCE_PRODUCTS`. ولا يكرر عروض شركة التأمين. |
| `PROMOTION_REDEMPTIONS` | يسجل استخدام الحملة المحلية. | يربط `PROMOTIONS` مع`QUOTE_REQUESTS`. |
| `COMMISSION_RULES` | يحدد عمولة الوسيط حسب المنتج. | يتبع `INSURANCE_PRODUCTS`. وقد يقيد بـ`INSURERS`. |

## ملاحظات العلاقات

- العلاقة واحد إلى متعدد تعني أن السجل الأب يملك عدة سجلات تابعة.
- العلاقة الاختيارية تعني أن السجل قد لا يوجد بعد.
- العرض لا ينشئ وثيقة قبل الاختيار والدفع والإصدار الخارجي.
- حذف السجلات المالية أو التأمينية غير مسموح.
- تغير الحالة يسجل في جداول التاريخ.
- ردود التكامل تحفظ كنسخ تدقيق غير قابلة للتعديل.
