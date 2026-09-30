# 🗄️ SAWA POS — Database Design & Schema Documentation

## وثيقة التصميم التفصيلية لقاعدة البيانات

> هذه الوثيقة تصف **Schema قاعدة البيانات المقدم للمشروع** وتربطه بالمنطق الفعلي للنظام. تم استبعاد `AuditLogs` بالكامل من هذه الوثيقة.

---

# 📌 1. نظرة عامة

قاعدة بيانات SAWA POS هي قاعدة بيانات علائقية SQL Server تدعم دورة التشغيل الأساسية للنظام:

```text
People
   ↓
Users
   ↓
Roles

Categories
   ↓
Products
   ↓
ProductVariants
   ↓
Variants

Users / Shifts
       ↓
Orders
       ↓
OrderItems
       ↓
Payments

Shifts
 ├── Orders
 ├── Expenses
 └── ShiftCorrections

Original Order
      ↕
RefundTracking
      ↕
Refund Order

SystemSettings
```

تخدم قاعدة البيانات وظائف:

- 👤 إدارة المستخدمين.
- 🛡️ الأدوار والصلاحيات.
- 🗂️ التصنيفات.
- 🍔 المنتجات.
- 📏 Product Variants / Sizes.
- 🧾 الطلبات.
- 🧩 عناصر الطلبات.
- 💳 المدفوعات.
- 🕒 الورديات.
- 💵 المصروفات.
- 🛠️ تصحيحات الوردية.
- ↩️ تتبع المرتجعات.
- ⚙️ إعدادات النظام.

---

# 🏗️ 2. نموذج البيانات العام

```text
┌────────────┐
│   People   │
└─────┬──────┘
      │
      ▼
┌────────────┐       ┌────────────┐
│   Users    │───────│   Roles    │
└─────┬──────┘       └────────────┘
      │
      ├──────────────┐
      │              │
      ▼              ▼
┌────────────┐  ┌────────────┐
│   Shifts   │  │   Orders   │
└─────┬──────┘  └─────┬──────┘
      │                │
      ├───────┐        ├──────────────┐
      │       │        │              │
      ▼       ▼        ▼              ▼
 Expenses  Corrections OrderItems   Payments
                         │
                  ┌──────┴──────┐
                  ▼             ▼
              Products       Variants
                  │
                  ▼
           ProductVariants
                  │
                  ▼
              Categories

RefundTracking
   │
   ├── OriginalOrderID
   └── RefundOrderID

SystemSettings
   └── Global Configuration
```

---

# 📋 3. الجداول

عدد الجداول في الـSchema المقدم، بعد استبعاد `AuditLogs`، هو:

**15 جدولًا**

```text
1. Categories
2. Expenses
3. OrderItems
4. Orders
5. Payments
6. People
7. Products
8. ProductVariants
9. RefundTracking
10. Roles
11. ShiftCorrections
12. Shifts
13. SystemSettings
14. Users
15. Variants
```

---

# 👤 4. People

## الغرض

يمثل البيانات الشخصية المرتبطة بحسابات المستخدمين.

## Columns

| Column | Type | Null | Key | Default |
|---|---|---:|---|---|
| `PersonID` | int | NO | PK, Identity | — |
| `FirstName` | nvarchar(50) | NO | — | — |
| `LastName` | nvarchar(50) | NO | — | — |
| `Phone` | nvarchar(20) | YES | — | — |
| `Address` | nvarchar(100) | YES | — | — |

## العلاقة

```text
People
  1
  │
  └────────── N? / logically one account per person
             Users
```

الـFK المقدم:

```text
FK_Users_People
Users.PersonID → People.PersonID
```

---

# 👤 5. Users

## الغرض

يمثل حسابات الدخول والصلاحيات.

## Columns

| Column | Type | Null | Key | Default |
|---|---|---:|---|---|
| `UserID` | int | NO | PK, Identity | — |
| `PersonID` | int | NO | FK | — |
| `RoleID` | int | NO | FK | — |
| `Username` | nvarchar(50) | NO | — | — |
| `HashedPassword` | nvarchar(200) | NO | — | — |
| `IsActive` | bit | NO | — | `1` |
| `Permissions` | int | YES | — | NULL |

## العلاقات

```text
People → Users
Roles  → Users
```

وكذلك يستخدم User كمصدر للعمليات:

```text
Users
 ├── Orders.CreatedByUserID
 ├── Expenses.CreatedByUserID
 ├── Shifts.OpenedByUser
 └── ShiftCorrections.CorrectedByUserID
```

---

# 🛡️ 6. Roles

## الغرض

تخزين الدور المرتبط بالمستخدم.

## Columns

| Column | Type | Null | Key | Default |
|---|---|---:|---|---|
| `RoleID` | int | NO | PK, Identity | — |
| `Role` | nvarchar(50) | NO | — | — |

## التطبيق الحالي

منطق التطبيق الحالي يستخدم:

```text
1 → Admin
2 → Cashier
```

> الجدول يسمح بتخزين أدوار أخرى، لكن الواجهة والمنطق الحاليين لا يشكلان نظامًا ديناميكيًا كاملًا لإدارة الأدوار.

---

# 🗂️ 7. Categories

## الغرض

تصنيف المنتجات.

## Columns

| Column | Type | Null | Key | Default |
|---|---|---:|---|---|
| `CategoryID` | int | NO | PK, Identity | — |
| `CategoryName` | nvarchar(100) | NO | — | — |
| `IsActive` | bit | NO | — | `1` |
| `Description` | nvarchar(200) | YES | — | NULL |
| `CreatedAt` | datetime | YES | — | NULL |
| `ImagePath` | nvarchar(200) | YES | — | NULL |

## العلاقة

```text
Categories 1 ───── N Products
```

FK:

```text
FK_Products_Categories
Products.CategoryID → Categories.CategoryID
```

---

# 🍔 8. Products

## الغرض

يمثل المنتج الذي يباع من خلال POS.

## Columns

| Column | Type | Null | Key | Default |
|---|---|---:|---|---|
| `ProductID` | int | NO | PK, Identity | — |
| `CategoryID` | int | NO | FK | — |
| `ProductName` | nvarchar(200) | NO | — | — |
| `Description` | nvarchar(255) | YES | — | NULL |
| `Price` | decimal(18,2) | NO | — | — |
| `IsAvailable` | bit | NO | — | `1` |
| `ImagePath` | nvarchar(255) | YES | — | NULL |

---

# 📏 9. Variants

## الغرض

تعريف أنواع/أحجام المنتجات.

## Columns

| Column | Type | Null | Key | Default |
|---|---|---:|---|---|
| `VariantID` | int | NO | PK | — |
| `Name` | varchar(50) | NO | — | — |

أمثلة:

```text
Small
Medium
Large
```

> `VariantID` ليس Identity حسب الـSchema المقدم.

---

# 🔗 10. ProductVariants

## الغرض

ربط المنتجات بالـVariants مع سعر خاص لكل ارتباط.

## Columns

| Column | Type | Null | Key | Default |
|---|---|---:|---|---|
| `ProductID` | int | NO | PK, FK | — |
| `VariantID` | int | NO | PK, FK | — |
| `Price` | decimal(10,3) | NO | — | — |

## Primary Key

المفتاح مركب:

```text
PRIMARY KEY
(
    ProductID,
    VariantID
)
```

## العلاقات

```text
Products
    │
    ▼
ProductVariants
    ▲
    │
Variants
```

FKs:

```text
FK__ProductVa__Produ__04E4BC85
ProductVariants.ProductID → Products.ProductID
ON DELETE CASCADE

FK__ProductVa__Varia__05D8E0BE
ProductVariants.VariantID → Variants.VariantID
ON DELETE NO ACTION
```

---

# 🕒 11. Shifts

## الغرض

تمثل الوردية التشغيلية والنقدية.

## Columns

| Column | Type | Null | Key | Default |
|---|---|---:|---|---|
| `ShiftID` | int | NO | PK, Identity | — |
| `OpenedByUser` | int | NO | FK | — |
| `OpenedAt` | datetime | NO | — | — |
| `ClosedAt` | datetime | YES | — | NULL |
| `OpeningCash` | decimal(18,2) | NO | — | — |
| `Status` | tinyint | NO | — | — |
| `ExpectedCash` | decimal(18,2) | YES | — | NULL |
| `ActualCash` | decimal(18,2) | YES | — | NULL |
| `Difference` | decimal(18,2) | YES | — | NULL |
| `Notes` | nvarchar(255) | YES | — | NULL |

FK:

```text
FK_Shifts_Users
Shifts.OpenedByUser → Users.UserID
```

---

# 🧾 12. Orders

## الغرض

الوحدة الرئيسية لعملية البيع.

## Columns

| Column | Type | Null | Key | Default |
|---|---|---:|---|---|
| `OrderID` | int | NO | PK, Identity | — |
| `ShiftID` | int | NO | FK | — |
| `CreatedByUserID` | int | NO | FK | — |
| `CreatedAt` | datetime | NO | — | `GETDATE()` |
| `SubTotal` | decimal(18,2) | NO | — | — |
| `DiscountPercentage` | decimal(5,2) | NO | — | `0.00` |
| `TotalPrice` | decimal(18,2) | NO | — | — |
| `Status` | tinyint | NO | — | — |
| `TotalAmount` | decimal(18,3) | NO | — | `0.000` |
| `TaxNumber` | nvarchar(50) | YES | — | NULL |
| `VATPercentage` | decimal(5,2) | YES | — | NULL |
| `IsTaxInclusive` | bit | YES | — | NULL |

## العلاقات

```text
Shifts → Orders
Users  → Orders
```

FKs:

```text
FK_Orders_Shifts
Orders.ShiftID → Shifts.ShiftID

FK_Orders_Users
Orders.CreatedByUserID → Users.UserID
```

---

# 🧩 13. OrderItems

## الغرض

تفاصيل المنتجات داخل الطلب.

## Columns

| Column | Type | Null | Key | Default |
|---|---|---:|---|---|
| `ItemID` | int | NO | PK, Identity | — |
| `OrderID` | int | NO | FK | — |
| `ProductID` | int | NO | FK | — |
| `UnitPrice` | decimal(18,2) | NO | — | — |
| `DiscountPercentage` | decimal(5,2) | YES | — | NULL |
| `TotalPrice` | decimal(18,2) | NO | — | — |
| `Quantity` | int | NO | — | — |
| `Notes` | nvarchar(150) | YES | — | NULL |
| `VariantID` | int | YES | — | NULL |

## العلاقات

```text
Orders
   │
   └── OrderItems
          │
          ├── Products
          └── Variant (logical relationship)
```

FKs الموجودة في الـSchema المقدم:

```text
FK_OrderItems_Orders
OrderItems.OrderID → Orders.OrderID

FK_OrderItems_Products
OrderItems.ProductID → Products.ProductID
```

### ملاحظة مهمة حول VariantID

`OrderItems.VariantID` موجود في الجدول ويستخدمه النظام، لكن **لا يوجد FK له في قائمة القيود المرسلة**.

والتصميم الأقوى منطقيًا هو ضمان أن زوج:

```text
(ProductID, VariantID)
```

موجود في `ProductVariants`.

---

# 💳 14. Payments

## الغرض

تسجيل الأموال المدفوعة مقابل الطلبات.

## Columns

| Column | Type | Null | Key | Default |
|---|---|---:|---|---|
| `PaymentID` | int | NO | PK, Identity | — |
| `OrderID` | int | NO | — | — |
| `PaymentMethod` | tinyint | NO | — | — |
| `Amount` | decimal(18,2) | NO | — | — |
| `PaidAmount` | decimal(18,2) | NO | — | — |
| `ChangeAmount` | decimal(18,2) | NO | — | — |
| `PaidAt` | datetime | NO | — | `GETDATE()` |
| `ShiftID` | int | NO | — | — |

## Payment Methods

حسب التطبيق:

```text
1 = Cash
2 = Visa/Card
```

## Split Payment

يمكن أن توجد عدة سجلات Payment للطلب نفسه:

```text
Order 1001
│
├── Payment: Cash = 4.000
└── Payment: Card = 6.000
```

### ملاحظة Schema

الـFK list المقدم لا يحتوي FK صريحًا لـ:

```text
Payments.OrderID → Orders.OrderID
Payments.ShiftID → Shifts.ShiftID
```

مع أن التطبيق يستخدمهما منطقيًا.

يوصى بإضافة القيود إذا لم تكن موجودة فعليًا في قاعدة البيانات.

---

# 💵 15. Expenses

## الغرض

تسجيل المصروفات التشغيلية.

## Columns

| Column | Type | Null | Key | Default |
|---|---|---:|---|---|
| `ExpenseID` | int | NO | PK, Identity | — |
| `CreatedByUserID` | int | NO | FK | — |
| `ShiftID` | int | NO | FK | — |
| `Amount` | decimal(18,2) | NO | — | — |
| `Reason` | nvarchar(200) | NO | — | — |
| `CreatedAt` | datetime | NO | — | `GETDATE()` |
| `PaymentSource` | tinyint | NO | — | — |
| `Description` | nvarchar(255) | YES | — | NULL |
| `StatusExpense` | bit | NO | — | — |

FKs:

```text
FK_Expenses_Shifts
Expenses.ShiftID → Shifts.ShiftID

FK_Expenses_Users
Expenses.CreatedByUserID → Users.UserID
```

---

# ↩️ 16. RefundTracking

## الغرض

ربط عملية المرتجع بالطلب الأصلي وطلب المرتجع.

## Columns

| Column | Type | Null | Key | Default |
|---|---|---:|---|---|
| `TrackingID` | int | NO | PK, Identity | — |
| `RefundOrderID` | int | NO | — | — |
| `OriginalOrderID` | int | NO | — | — |
| `RefundReason` | nvarchar(255) | YES | — | NULL |
| `RefundDate` | datetime | YES | — | `GETDATE()` |

العلاقة المنطقية:

```text
Original Order
      │
      ▼
RefundTracking
      │
      ▼
Refund Order
```

### ملاحظة مهمة

قائمة الـFKs المرسلة لا تحتوي قيودًا على:

```text
RefundTracking.RefundOrderID
RefundTracking.OriginalOrderID
```

يوصى بإنشاء FKين إلى `Orders.OrderID` إذا لم يكونا موجودين في قاعدة البيانات الفعلية.

---

# 🛠️ 17. ShiftCorrections

## الغرض

تسجيل تصحيحات القيم المالية للوردية.

## Columns

| Column | Type | Null | Key | Default |
|---|---|---:|---|---|
| `CorrectionID` | int | NO | PK, Identity | — |
| `ShiftID` | int | NO | FK | — |
| `CorrectionType` | tinyint | NO | — | — |
| `OldValue` | decimal(18,2) | NO | — | — |
| `NewValue` | decimal(18,2) | NO | — | — |
| `Reason` | nvarchar(255) | YES | — | NULL |
| `CorrectedByUserID` | int | NO | FK | — |
| `CorrectedAt` | datetime | NO | — | `GETDATE()` |

FKs:

```text
FK_ShiftCorrections_Shifts
ShiftCorrections.ShiftID → Shifts.ShiftID

FK_ShiftCorrections_Users
ShiftCorrections.CorrectedByUserID → Users.UserID
```

---

# ⚙️ 18. SystemSettings

## الغرض

تخزين إعدادات المنشأة والنظام.

## Columns

| Column | Type | Null | Key | Default |
|---|---|---:|---|---|
| `SettingID` | int | NO | PK, Identity | — |
| `Language` | nvarchar(50) | NO | — | — |
| `LogoPath` | nvarchar(250) | NO | — | — |
| `RestaurantName` | nvarchar(250) | NO | — | — |
| `Currency` | nvarchar(50) | NO | — | — |
| `VAT` | decimal(5,2) | YES | — | NULL |
| `VatEnable` | bit | NO | — | `0` |
| `crNumber` | nvarchar(50) | YES | — | NULL |
| `TaxNumber` | nvarchar(50) | YES | — | NULL |
| `Phone` | nvarchar(30) | YES | — | NULL |
| `Address` | nvarchar(255) | YES | — | NULL |
| `DecimalPlaces` | int | NO | — | `3` |
| `TaxMethod` | nvarchar(100) | YES | — | `Inclusive` |
| `PrinterName` | nvarchar(100) | YES | — | NULL |
| `AutoPrintReceipt` | bit | YES | — | `1` |
| `ShowQRCode` | bit | YES | — | `1` |
| `MaxDiscountPercentage` | decimal(5,2) | YES | — | `0.00` |
| `EnableBlindClose` | bit | YES | — | `0` |
| `EnableDiscount` | bit | YES | — | NULL |
| `Email` | nvarchar(50) | YES | — | NULL |
| `EncryptedBrevoKey` | nvarchar(500) | YES | — | NULL |
| `SenderEmail` | nvarchar(50) | YES | — | NULL |

هذه البيانات تمثل Configuration وليس معاملات تشغيلية.

---

# 🔗 19. جميع Foreign Keys المرسلة

| FK | From | To | Delete | Update |
|---|---|---|---|---|
| `FK_Expenses_Shifts` | Expenses.ShiftID | Shifts.ShiftID | NO ACTION | NO ACTION |
| `FK_Expenses_Users` | Expenses.CreatedByUserID | Users.UserID | NO ACTION | NO ACTION |
| `FK_OrderItems_Orders` | OrderItems.OrderID | Orders.OrderID | NO ACTION | NO ACTION |
| `FK_OrderItems_Products` | OrderItems.ProductID | Products.ProductID | NO ACTION | NO ACTION |
| `FK_Orders_Shifts` | Orders.ShiftID | Shifts.ShiftID | NO ACTION | NO ACTION |
| `FK_Orders_Users` | Orders.CreatedByUserID | Users.UserID | NO ACTION | NO ACTION |
| `FK_Products_Categories` | Products.CategoryID | Categories.CategoryID | NO ACTION | NO ACTION |
| `FK__ProductVa__Produ__04E4BC85` | ProductVariants.ProductID | Products.ProductID | CASCADE | NO ACTION |
| `FK__ProductVa__Varia__05D8E0BE` | ProductVariants.VariantID | Variants.VariantID | NO ACTION | NO ACTION |
| `FK_ShiftCorrections_Shifts` | ShiftCorrections.ShiftID | Shifts.ShiftID | NO ACTION | NO ACTION |
| `FK_ShiftCorrections_Users` | ShiftCorrections.CorrectedByUserID | Users.UserID | NO ACTION | NO ACTION |
| `FK_Shifts_Users` | Shifts.OpenedByUser | Users.UserID | NO ACTION | NO ACTION |
| `FK_Users_People` | Users.PersonID | People.PersonID | NO ACTION | NO ACTION |
| `FK_Users_Roles` | Users.RoleID | Roles.RoleID | NO ACTION | NO ACTION |

---

# 🧬 20. ERD المنطقي

```text
                         ┌──────────────┐
                         │    Roles     │
                         │──────────────│
                         │ RoleID  PK   │
                         │ Role         │
                         └──────┬───────┘
                                │
                                │
┌──────────────┐         ┌──────▼───────┐
│    People    │         │    Users     │
│──────────────│         │──────────────│
│ PersonID PK  │◄────────│ PersonID FK  │
│ FirstName    │         │ UserID PK    │
│ LastName     │         │ RoleID FK    │
│ Phone        │         │ Username     │
│ Address      │         │ PasswordHash │
└──────────────┘         │ Permissions  │
                         └──────┬───────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
          ▼                     ▼                     ▼
    ┌───────────┐         ┌───────────┐       ┌───────────────┐
    │  Shifts   │         │  Orders   │       │ ShiftCorrect. │
    └─────┬─────┘         └─────┬─────┘       └───────────────┘
          │                     │
     ┌────┼────┐           ┌────┴───────┐
     │    │    │           │            │
     ▼    ▼    ▼           ▼            ▼
 Expenses Orders Payments OrderItems   Payments
             │
             ▼
        ┌───────────┐
        │OrderItems │
        └─────┬─────┘
              │
        ┌─────┴─────┐
        ▼           ▼
    Products     Variants
        │           ▲
        │           │
        └─────┬─────┘
              ▼
      ProductVariants
              │
              ▼
         Categories
```

---

# 🧾 21. Order / Payment / Refund Model

## البيع

```text
Order
 │
 ├── OrderItems
 │       ├── Product
 │       └── Variant
 │
 └── Payments
```

## الدفع المجزأ

```text
Order
 │
 ├── Payment #1 → Cash
 └── Payment #2 → Card
```

## المرتجع

```text
Original Order
      │
      ▼
RefundTracking
      │
      ▼
Refund Order
      │
      ├── Negative OrderItems
      └── Negative Payment
```

---

# 🕒 22. Shift Financial Model

الوردية تجمع العمليات المالية التي حدثت خلالها:

```text
Shift
 │
 ├── OpeningCash
 │
 ├── Orders
 │      └── Payments
 │
 ├── Expenses
 │
 └── Corrections
```

النموذج التنفيذي للنقد المتوقع:

```text
Expected Cash
=
Opening Cash
+
Cash Payments
-
Cash Expenses
```

وبما أن المرتجعات النقدية تسجل بقيم سالبة، فإنها تؤثر تلقائيًا في صافي Cash Payments.

ثم:

```text
Difference
=
Actual Cash
-
Expected Cash
```

---

# 🔐 23. Data Integrity

## Primary Keys

كل كيان تشغيلي رئيسي لديه Primary Key.

أمثلة:

```text
Users.UserID
Products.ProductID
Orders.OrderID
Payments.PaymentID
Shifts.ShiftID
Expenses.ExpenseID
```

## Composite Primary Key

جدول:

```text
ProductVariants
```

يستخدم:

```text
(ProductID, VariantID)
```

كمفتاح مركب.

---

# 🔗 24. Referential Integrity

الـFKs الحالية تحمي عددًا من العلاقات المهمة:

```text
Users → People
Users → Roles

Products → Categories

Orders → Users
Orders → Shifts

OrderItems → Orders
OrderItems → Products

Expenses → Users
Expenses → Shifts

Shifts → Users

ShiftCorrections → Users
ShiftCorrections → Shifts

ProductVariants → Products
ProductVariants → Variants
```

---

# ⚠️ 25. فجوات Referential Integrity في الـSchema المقدم

بحسب قائمة الـFKs التي تم تقديمها، هناك علاقات يستخدمها النظام منطقيًا ولكنها غير موجودة كقيود في القائمة.

## Payments

يفترض:

```text
Payments.OrderID → Orders.OrderID
Payments.ShiftID → Shifts.ShiftID
```

لكن هذه القيود غير ظاهرة في القائمة المرسلة.

## RefundTracking

يفترض:

```text
RefundTracking.RefundOrderID → Orders.OrderID
RefundTracking.OriginalOrderID → Orders.OrderID
```

لكنها غير ظاهرة في القائمة.

## OrderItems.VariantID

يفترض ارتباطًا بالـVariants، والأفضل تصميميًا أن يتم التحقق من:

```text
(OrderItems.ProductID, OrderItems.VariantID)
        ↓
(ProductVariants.ProductID, ProductVariants.VariantID)
```

وذلك لمنع اختيار Variant لا ينتمي إلى Product المحدد.

---

# 🧠 26. لماذا Historical Pricing مهم؟

`Products.Price` يمثل السعر الحالي.

لكن:

```text
OrderItems.UnitPrice
```

يمثل السعر الذي تم البيع به.

مثال:

```text
Product.Price = 2.500

Order created:
OrderItem.UnitPrice = 2.000

Product.Price later:
2.500

Historical Order:
2.000
```

وهذا يحافظ على دقة التقارير والفواتير التاريخية.

---

# 💰 27. Monetary Data Types

يستخدم التصميم `decimal` للأموال بدل `float`.

أمثلة:

```text
decimal(18,2)
decimal(18,3)
decimal(10,3)
decimal(5,2)
```

الاستخدامات:

- Prices.
- Totals.
- Discounts.
- VAT.
- Payments.
- Expenses.
- Shift Cash.

وهذا مناسب للبيانات المالية لأنه يتجنب مشاكل الدقة الشائعة مع floating-point arithmetic.

---

# 🧮 28. VAT Data Model

يوجد إعداد عام:

```text
SystemSettings.VAT
SystemSettings.VatEnable
SystemSettings.TaxMethod
```

ويحتفظ الطلب أيضًا بالقيم المرتبطة به:

```text
Orders.VATPercentage
Orders.TaxNumber
Orders.IsTaxInclusive
```

وهذا يسمح بالحفاظ على معلومات الضريبة المرتبطة بالطلب بدل الاعتماد فقط على الإعداد الحالي.

---

# ⚙️ 29. System Configuration Model

```text
SystemSettings
│
├── Business Identity
│   ├── RestaurantName
│   ├── LogoPath
│   ├── Phone
│   ├── Address
│   ├── crNumber
│   └── TaxNumber
│
├── Financial
│   ├── Currency
│   ├── DecimalPlaces
│   ├── VAT
│   ├── VatEnable
│   └── TaxMethod
│
├── Printing
│   ├── PrinterName
│   ├── AutoPrintReceipt
│   └── ShowQRCode
│
├── Discount / Shift
│   ├── MaxDiscountPercentage
│   ├── EnableBlindClose
│   └── EnableDiscount
│
└── Email
    ├── Email
    ├── SenderEmail
    └── EncryptedBrevoKey
```

---

# 📊 30. Views المستخدمة في التطبيق

من تحليل طبقة البيانات يظهر استخدام أسماء Views مثل:

```text
Orders_View
AllExpenses_View
AllShiftsView
```

هذه ليست ضمن قائمة الـTables التي تم إرسالها، وبالتالي يجب التعامل معها على أنها **Database Views منفصلة إذا كانت موجودة فعليًا في SQL Server**.

لا يمكن تحديد أعمدتها بدقة من الـSchema المرسل فقط، ولذلك لا ينبغي اختراع تعريفات لها في وثيقة الـDDL دون استخراجها من قاعدة البيانات نفسها.

---

# 🚫 31. كيانات غير موجودة

رغم أن بعض أوصاف المشروع القديمة تشير إلى Inventory/Purchases، فإن الـSchema الحالي لا يحتوي على:

```text
Inventory
Stock
StockMovements
Purchases
PurchaseItems
Suppliers
```

كما لا يحتوي على:

```text
AddOns
ProductAddOns
OrderItemAddOns
```

لذلك:

> قاعدة البيانات الحالية هي POS + Operations Database، وليست Inventory/Purchasing Database.

---

# 🔄 32. Transaction Boundaries

العمليات التي تمس عدة كيانات ينبغي تنفيذها كـAtomic Transaction.

## Order Creation

```text
BEGIN
   Insert Order
   Insert OrderItems
COMMIT
```

## Payment

```text
BEGIN
   Insert Payment(s)
   Update Order Status
COMMIT
```

## Refund

```text
BEGIN
   Validate Refund
   Create Refund Order
   Create Refund Items
   Create Refund Payment
   Create RefundTracking
COMMIT
```

وفي حالة الفشل:

```text
ROLLBACK
```

---

# 🛡️ 33. Database Security Recommendations

هذه ليست ادعاءات عن الـSchema الحالي، وإنما توصيات للحماية:

## Connection Credentials

لا ينبغي تخزين:

```text
SQL Username
SQL Password
```

داخل Repository عام.

يفضل:

```text
Environment Variables
User Secrets
Secret Manager
Encrypted Configuration
```

## API Keys

لا ينبغي تخزين مفاتيح Brevo الحقيقية داخل Git.

## Password Hashing

يستخدم التطبيق Hashing لكلمات المرور، لكن في الأنظمة الإنتاجية الحديثة يفضل استخدام:

```text
Argon2
bcrypt
PBKDF2
scrypt
```

مع Salt مناسب بدل الاعتماد على SHA-256 وحده.

---

# 🧱 34. Normalization

التصميم يفصل كيانات متكررة بشكل واضح:

```text
People
Users
Roles
```

بدل تخزين بيانات الشخص داخل Users فقط.

كما يفصل:

```text
Categories
Products
```

ويفصل:

```text
Products
Variants
ProductVariants
```

ويفصل:

```text
Orders
OrderItems
Payments
```

وهذا يقلل التكرار ويحافظ على اتساق البيانات.

---

# 📐 35. Cardinality Summary

| Relationship | Cardinality |
|---|---|
| People → Users | One-to-Many logically possible by schema; application expects one account per person |
| Roles → Users | One-to-Many |
| Categories → Products | One-to-Many |
| Products → ProductVariants | One-to-Many |
| Variants → ProductVariants | One-to-Many |
| Shifts → Orders | One-to-Many |
| Shifts → Expenses | One-to-Many |
| Shifts → ShiftCorrections | One-to-Many |
| Users → Orders | One-to-Many |
| Users → Expenses | One-to-Many |
| Users → Shifts | One-to-Many |
| Users → ShiftCorrections | One-to-Many |
| Orders → OrderItems | One-to-Many logically |
| Orders → Payments | One-to-Many logically |
| Orders ↔ Refund Orders | Via RefundTracking |

---

# 🧭 36. Complete Data Flow

```text
User Login
    ↓
Users / Roles
    ↓
Permission Check
    ↓
Open Shift
    ↓
Products / Categories / Variants
    ↓
Create Order
    ↓
OrderItems
    ↓
Discount + VAT
    ↓
Payment
    ↓
Completed Order
    ↓
Reports / Dashboard
    ↓
Possible Refund
    ↓
Shift Closing
    ↓
Expected Cash
    ↓
Actual Cash
    ↓
Difference
```

---

# 📌 37. التصميم النهائي المختصر

```text
                    ┌───────────────┐
                    │ SystemSettings│
                    └───────────────┘

┌─────────┐       ┌─────────┐       ┌─────────┐
│ People  │──────▶│  Users  │──────▶│  Roles  │
└─────────┘       └────┬────┘       └─────────┘
                       │
            ┌──────────┼───────────┐
            │          │           │
            ▼          ▼           ▼
         Shifts      Orders     Expenses
            │          │
            │          ├──────────────┐
            │          │              │
            │          ▼              ▼
            │      OrderItems      Payments
            │          │
            │      ┌───┴────┐
            │      ▼        ▼
            │  Products   Variants
            │      │        ▲
            │      └───┬────┘
            │          ▼
            │   ProductVariants
            │
            └── ShiftCorrections

              RefundTracking
                    │
           ┌────────┴────────┐
           ▼                 ▼
     Original Order      Refund Order
```

---

# 🎯 38. الخلاصة

قاعدة بيانات SAWA POS مصممة لدعم نظام POS تشغيلي يعتمد على:

- المستخدم.
- الدور والصلاحيات.
- المنتج والتصنيف.
- الـVariants.
- الطلب.
- تفاصيل الطلب.
- الدفع.
- الوردية.
- المصروف.
- تصحيحات الوردية.
- المرتجعات.
- إعدادات النظام.

أهم مبدأ في التصميم هو ربط المعاملات التشغيلية بالوردية والمستخدم، مع الاحتفاظ بالسعر التاريخي داخل `OrderItems`، وفصل الطلب عن تفاصيله والمدفوعات المرتبطة به.

كما أن `ProductVariants` يستخدم مفتاحًا مركبًا، و`ShiftCorrections` يوفر سجلًا متخصصًا لتعديلات القيم النقدية للوردية.

## ⚠️ أهم تحسينات Schema المقترحة

قبل اعتماد قاعدة البيانات كنسخة Production نهائية، يوصى بمراجعة وإضافة القيود المرجعية التالية إذا لم تكن موجودة فعليًا:

```text
Payments.OrderID → Orders.OrderID
Payments.ShiftID → Shifts.ShiftID

RefundTracking.RefundOrderID → Orders.OrderID
RefundTracking.OriginalOrderID → Orders.OrderID

(OrderItems.ProductID, OrderItems.VariantID)
    →
(ProductVariants.ProductID, ProductVariants.VariantID)
```

كما ينبغي توثيق الـViews:

```text
Orders_View
AllExpenses_View
AllShiftsView
```

إذا كانت موجودة فعليًا في قاعدة البيانات.

> **هذه الوثيقة تصف الـSchema المقدم للمشروع بدقة، ولا تضيف كيانات غير موجودة مثل Inventory أو Purchases أو Add-ons.**
