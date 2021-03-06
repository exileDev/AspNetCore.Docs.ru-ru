---
title: Пользовательские поставщики хранилища для удостоверения ASP.NET Core
author: ardalis
description: Узнайте, как настроить пользовательские поставщики хранилища для удостоверения ASP.NET Core.
ms.author: riande
ms.custom: mvc
ms.date: 07/23/2019
uid: security/authentication/identity-custom-storage-providers
ms.openlocfilehash: 70951085474d88fd57f1b1496a41adcda520b91f
ms.sourcegitcommit: 7dfe6cc8408ac6a4549c29ca57b0c67ec4baa8de
ms.translationtype: MT
ms.contentlocale: ru-RU
ms.lasthandoff: 01/09/2020
ms.locfileid: "75829158"
---
# <a name="custom-storage-providers-for-aspnet-core-identity"></a>Пользовательские поставщики хранилища для удостоверения ASP.NET Core

Автор: [Стив Смит](https://ardalis.com/) (Steve Smith)

ASP.NET Core Identity — это расширяемая система, которая позволяет создать пользовательский поставщик хранилища и подключить его к приложению. В этом разделе описывается создание настраиваемого поставщика хранилища для удостоверения ASP.NET Core. В нем рассматриваются важные понятия создания собственного поставщика хранилища, но не пошаговое пошаговое руководство.

[Просмотреть или скачать образец с GitHub](https://github.com/aspnet/AspNetCore.Docs/tree/master/aspnetcore/security/authentication/identity/sample).

## <a name="introduction"></a>Введение

По умолчанию система удостоверений ASP.NET Core сохраняет сведения о пользователях в базе данных SQL Server с помощью Entity Framework Core. Для многих приложений этот подход хорошо работает. Однако можно использовать другой механизм сохраняемости или схему данных. Например:

* Вы используете [хранилище таблиц Azure](/azure/storage/) или другое хранилище данных.
* Таблицы базы данных имеют различную структуру. 
* Может потребоваться использовать другой подход к доступу к данным, например [Dapper](https://github.com/StackExchange/Dapper). 

В каждом из этих случаев можно написать настраиваемый поставщик для механизма хранения и подключить этот поставщик к приложению.

ASP.NET Core удостоверение включается в шаблоны проектов Visual Studio с параметром "индивидуальные учетные записи пользователей".

При использовании .NET Core CLI добавьте `-au Individual`:

```dotnetcli
dotnet new mvc -au Individual
```

## <a name="the-aspnet-core-identity-architecture"></a>Архитектура идентификации ASP.NET Core

ASP.NET Core удостоверение состоит из классов, именуемых диспетчерами и хранилищами. *Руководители* — это высокоуровневые классы, которые разработчик приложений использует для выполнения таких операций, как создание пользователя удостоверения. *Магазины* — это классы более низкого уровня, указывающие, как сохраняются сущности, например пользователи и роли. Хранилища соответствуют шаблону репозитория и тесно связаны с механизмом сохраняемости. Менеджеры отделяются от магазинов, что означает возможность замены механизма сохраняемости без изменения кода приложения (кроме конфигурации).

На следующей схеме показано, как веб-приложение взаимодействует с менеджерами, в то время как сохраняет взаимодействие с уровнем доступа к данным.

![ASP.NET Core приложения работают с менеджерами (например, "UserManager", "RoleManager"). Диспетчеры работают с хранилищами (например, "UserStore"), которые взаимодействуют с источником данных с помощью библиотеки, например Entity Framework Core.](identity-custom-storage-providers/_static/identity-architecture-diagram.png)

Чтобы создать пользовательский поставщик хранилища, создайте источник данных, уровень доступа к данным и классы хранилища, взаимодействующие с этим уровнем доступа к данным (зеленые и серые поля на схеме выше). Вам не нужно настраивать руководителей или код приложения, который взаимодействует с ними (синие ячейки выше).

При создании нового экземпляра `UserManager` или `RoleManager` указать тип класса пользователя и передать экземпляр класса Store в качестве аргумента. Такой подход позволяет подключать пользовательские классы к ASP.NET Core. 

[Перенастроить приложение для использования нового поставщика хранилища](#reconfigure-app-to-use-a-new-storage-provider) показывает, как создать экземпляр `UserManager` и `RoleManager` с настроенным хранилищем.

## <a name="aspnet-core-identity-stores-data-types"></a>Типы данных хранилища ASP.NET Core

[ASP.NET Core типы данных удостоверений](https://github.com/aspnet/identity) подробно описаны в следующих разделах:

### <a name="users"></a>Пользователи

Зарегистрированные пользователи вашего веб-сайта. Тип [идентитюсер](/dotnet/api/microsoft.aspnet.identity.corecompat.identityuser) можно расширить или использовать в качестве примера для собственного пользовательского типа. Для реализации пользовательского решения для хранения удостоверений не нужно наследовать от определенного типа.

### <a name="user-claims"></a>Утверждения пользователей

Набор инструкций (или [утверждений](/dotnet/api/system.security.claims.claim)) о пользователе, который представляет удостоверение пользователя. Можно включить большее выражение удостоверения пользователя, чем можно достичь с помощью ролей.

### <a name="user-logins"></a>Имена входа пользователей

Сведения о внешнем поставщике проверки подлинности (например, Facebook или учетная запись Майкрософт), который будет использоваться для входа пользователя. [Пример](/dotnet/api/microsoft.aspnet.identity.corecompat.identityuserlogin)

### <a name="roles"></a>Роли

Группы авторизации для сайта. Содержит идентификатор роли и имя роли (например, "admin" или "Employee"). [Пример](/dotnet/api/microsoft.aspnet.identity.corecompat.identityrole)

## <a name="the-data-access-layer"></a>Уровень доступа к данным

В этом разделе предполагается, что вы знакомы с механизмом сохраняемости, который вы собираетесь использовать, и как создавать сущности для этого механизма. Этот раздел не содержит сведений о создании репозиториев или классов доступа к данным. в нем приводятся некоторые рекомендации по проектированию при работе с удостоверениями ASP.NET Core.

При проектировании уровня доступа к данным для настроенного поставщика хранилища у вас есть масса свободы. Для функций, которые предполагается использовать в приложении, необходимо создать только механизмы сохраняемости. Например, если вы не используете роли в приложении, вам не нужно создавать хранилище для ролей или ассоциаций ролей пользователей. Для вашей технологии и существующей инфраструктуры может потребоваться структура, которая сильно отличается от стандартной реализации удостоверения ASP.NET Core. На уровне доступа к данным вы предоставляете логику для работы со структурой реализации хранилища.

Уровень доступа к данным предоставляет логику для сохранения данных из ASP.NET Core удостоверения в источнике данных. Уровень доступа к данным для настроенного поставщика хранилища может включать следующие классы для хранения сведений о пользователях и ролях.

### <a name="context-class"></a>Context - класс

Инкапсулирует сведения для подключения к механизму сохранения и выполнения запросов. Для нескольких классов данных требуется экземпляр этого класса, который обычно предоставляется посредством внедрения зависимостей. [Пример](/dotnet/api/microsoft.aspnet.identity.corecompat.identitydbcontext-1).

### <a name="user-storage"></a>Хранилище пользователя

Сохраняет и извлекает сведения о пользователе (например, имя пользователя и хэш пароля). [Пример](/dotnet/api/microsoft.aspnet.identity.corecompat.userstore-1)

### <a name="role-storage"></a>Хранилище ролей

Сохраняет и извлекает сведения о ролях (например, имя роли). [Пример](/dotnet/api/microsoft.aspnetcore.identity.entityframeworkcore.rolestore-1)

### <a name="userclaims-storage"></a>Хранилище Усерклаимс

Сохраняет и извлекает сведения о пользовательской заявке (например, тип и значение утверждения). [Пример](/dotnet/api/microsoft.aspnet.identity.corecompat.userstore-1)

### <a name="userlogins-storage"></a>Хранилище Усерлогинс

Хранит и получает сведения для входа пользователя (например, внешний поставщик проверки подлинности). [Пример](/dotnet/api/microsoft.aspnet.identity.corecompat.userstore-1)

### <a name="userrole-storage"></a>Хранилище UserRole

Сохраняет и извлекает роли, назначенные пользователям. [Пример](/dotnet/api/microsoft.aspnet.identity.corecompat.userstore-1)

**Совет.** Реализуйте только те классы, которые планируется использовать в приложении.

В классах доступа к данным укажите код для выполнения операций с данными в механизме сохраняемости. Например, в настраиваемом поставщике может быть создан следующий код для создания пользователя в классе *Store* :

[!code-csharp[](identity-custom-storage-providers/sample/CustomIdentityProviderSample/CustomProvider/CustomUserStore.cs?name=createuser&highlight=7)]

Логика реализации для создания пользователя находится в методе `_usersTable.CreateAsync`, показанном ниже.

## <a name="customize-the-user-class"></a>Настройка класса User

При реализации поставщика хранилища создайте класс пользователя, эквивалентный [классу идентитюсер](/dotnet/api/microsoft.aspnet.identity.corecompat.identityuser).

Класс пользователя должен содержать как минимум `Id` и свойство `UserName`.

Класс `IdentityUser` определяет свойства, которые `UserManager` вызывает при выполнении запрошенных операций. По умолчанию типом свойства `Id` является строка, но можно наследовать от `IdentityUser<TKey, TUserClaim, TUserRole, TUserLogin, TUserToken>` и указать другой тип. Платформа должна обеспечивать реализацию хранилища для преобразования типов данных.

## <a name="customize-the-user-store"></a>Настройка хранилища пользователей

Создайте класс `UserStore`, предоставляющий методы для всех операций с данными пользователя. Этот класс эквивалентен классу [UserStore&lt;тусер&gt;](/dotnet/api/microsoft.aspnetcore.identity.entityframeworkcore.userstore-1) . В классе `UserStore` реализуйте `IUserStore<TUser>` и необходимые дополнительные интерфейсы. Выбор дополнительных интерфейсов для реализации зависит от функциональности, предоставляемой в приложении.

### <a name="optional-interfaces"></a>Необязательные интерфейсы

* [IUserRoleStore](/dotnet/api/microsoft.aspnetcore.identity.iuserrolestore-1)
* [IUserClaimStore](/dotnet/api/microsoft.aspnetcore.identity.iuserclaimstore-1)
* [IUserPasswordStore](/dotnet/api/microsoft.aspnetcore.identity.iuserpasswordstore-1)
* [иусерсекуритистампсторе](/dotnet/api/microsoft.aspnetcore.identity.iusersecuritystampstore-1)
* [иусеремаилсторе](/dotnet/api/microsoft.aspnetcore.identity.iuseremailstore-1)
* [иусерфоненумберсторе](/dotnet/api/microsoft.aspnetcore.identity.iuserphonenumberstore-1)
* [IQueryableUserStore](/dotnet/api/microsoft.aspnetcore.identity.iqueryableuserstore-1)
* [иусерлогинсторе](/dotnet/api/microsoft.aspnetcore.identity.iuserloginstore-1)
* [IUserTwoFactorStore](/dotnet/api/microsoft.aspnetcore.identity.iusertwofactorstore-1)
* [иусерлоккаутсторе](/dotnet/api/microsoft.aspnetcore.identity.iuserlockoutstore-1)

Необязательные интерфейсы наследуют от `IUserStore<TUser>`. В [примере приложения](https://github.com/aspnet/AspNetCore.Docs/blob/master/aspnetcore/security/authentication/identity-custom-storage-providers/sample/CustomIdentityProviderSample/CustomProvider/CustomUserStore.cs)можно увидеть частично реализованное хранилище пользователей с примером.

В классе `UserStore` используются классы доступа к данным, созданные для выполнения операций. Они передаются при помощи внедрения зависимостей. Например, в SQL Server с реализацией Dapper класс `UserStore` имеет метод `CreateAsync`, который использует экземпляр `DapperUsersTable` для вставки новой записи:

[!code-csharp[](identity-custom-storage-providers/sample/CustomIdentityProviderSample/CustomProvider/DapperUsersTable.cs?name=createuser&highlight=7)]

### <a name="interfaces-to-implement-when-customizing-user-store"></a>Интерфейсы, реализуемые при настройке хранилища пользователей

* **IUserStore**  
 Интерфейс [IUserStore&lt;тусер&gt;](/dotnet/api/microsoft.aspnetcore.identity.iuserstore-1) является единственным интерфейсом, который необходимо реализовать в хранилище пользователя. Он определяет методы для создания, обновления, удаления и получения пользователей.
* **IUserClaimStore**  
 Интерфейс [&gt;IUserClaimStore&lt;Тусер](/dotnet/api/microsoft.aspnetcore.identity.iuserclaimstore-1) определяет методы, которые реализуются для включения заявок пользователя. Он содержит методы для добавления, удаления и получения утверждений пользователей.
* **иусерлогинсторе**  
 [&gt;иусерлогинсторе&lt;Тусер](/dotnet/api/microsoft.aspnetcore.identity.iuserloginstore-1) определяет методы, которые реализуются для включения внешних поставщиков проверки подлинности. Он содержит методы для добавления, удаления и получения имен входа пользователей, а также метод для получения пользователя на основе сведений об имени входа.
* **IUserRoleStore**  
 Интерфейс [&gt;IUserRoleStore&lt;Тусер](/dotnet/api/microsoft.aspnetcore.identity.iuserrolestore-1) определяет методы, которые реализуются при сопоставлении пользователя с ролью. Он содержит методы для добавления, удаления и получения ролей пользователя, а также метод для проверки того, назначен ли пользователю роль.
* **IUserPasswordStore**  
 Интерфейс [&gt;IUserPasswordStore&lt;Тусер](/dotnet/api/microsoft.aspnetcore.identity.iuserpasswordstore-1) определяет методы, которые реализуются для сохранения хэшированных паролей. Он содержит методы для получения и установки хэшированного пароля, а также метод, который указывает, установил ли пользователь пароль.
* **иусерсекуритистампсторе**  
 Интерфейс [&gt;иусерсекуритистампсторе&lt;Тусер](/dotnet/api/microsoft.aspnetcore.identity.iusersecuritystampstore-1) определяет методы, которые вы реализуете для использования метки безопасности для указания, изменились ли сведения об учетной записи пользователя. Эта метка обновляется, когда пользователь изменяет пароль или добавляет или удаляет имена входа. Он содержит методы для получения и установки метки безопасности.
* **IUserTwoFactorStore**  
 Интерфейс [&gt;IUserTwoFactorStore&lt;Тусер](/dotnet/api/microsoft.aspnetcore.identity.iusertwofactorstore-1) определяет методы, которые реализуются для поддержки двухфакторной проверки подлинности. Он содержит методы для получения и настройки проверки подлинности, включенной для пользователя.
* **иусерфоненумберсторе**  
 Интерфейс [&gt;иусерфоненумберсторе&lt;Тусер](/dotnet/api/microsoft.aspnetcore.identity.iuserphonenumberstore-1) определяет методы, которые вы реализуете для хранения телефонных номеров пользователей. Он содержит методы для получения и установки номера телефона, а также сведения о подтверждении номера телефона.
* **иусеремаилсторе**  
 Интерфейс [&gt;иусеремаилсторе&lt;Тусер](/dotnet/api/microsoft.aspnetcore.identity.iuseremailstore-1) определяет методы, которые вы реализуете для хранения адресов электронной почты пользователей. Он содержит методы для получения и установки адреса электронной почты, а также сведения о том, подтверждено ли сообщение электронной почты.
* **иусерлоккаутсторе**  
 Интерфейс [&gt;иусерлоккаутсторе&lt;Тусер](/dotnet/api/microsoft.aspnetcore.identity.iuserlockoutstore-1) определяет методы, которые вы реализуете для хранения сведений о блокировке учетной записи. Он содержит методы для отслеживания неудачных попыток доступа и блокировки.
* **IQueryableUserStore**  
 Интерфейс [&gt;IQueryableUserStore&lt;Тусер](/dotnet/api/microsoft.aspnetcore.identity.iqueryableuserstore-1) определяет члены, которые вы реализуете для предоставления запрашиваемого хранилища пользователя.

Вы реализуете только те интерфейсы, которые необходимы в приложении. Например:

```csharp
public class UserStore : IUserStore<IdentityUser>,
                         IUserClaimStore<IdentityUser>,
                         IUserLoginStore<IdentityUser>,
                         IUserRoleStore<IdentityUser>,
                         IUserPasswordStore<IdentityUser>,
                         IUserSecurityStampStore<IdentityUser>
{
    // interface implementations not shown
}
```

### <a name="identityuserclaim-identityuserlogin-and-identityuserrole"></a>Идентитюсерклаим, Идентитюсерлогин и Идентитюсерроле

Пространство имен `Microsoft.AspNet.Identity.EntityFramework` содержит реализации классов [идентитюсерклаим](/dotnet/api/microsoft.aspnetcore.identity.entityframeworkcore.identityuserclaim-1), [идентитюсерлогин](/dotnet/api/microsoft.aspnet.identity.corecompat.identityuserlogin)и [идентитюсерроле](/dotnet/api/microsoft.aspnetcore.identity.entityframeworkcore.identityuserrole-1) . При использовании этих функций может потребоваться создать собственные версии этих классов и определить свойства приложения. Однако иногда эффективнее не загружать эти сущности в память при выполнении основных операций (например, при добавлении или удалении утверждения пользователя). Вместо этого классы хранилища серверной части могут выполнять эти операции непосредственно в источнике данных. Например, метод `UserStore.GetClaimsAsync` может вызвать метод `userClaimTable.FindByUserId(user.Id)`, чтобы выполнить запрос к этой таблице напрямую и вернуть список заявок.

## <a name="customize-the-role-class"></a>Настройка класса Role

При реализации поставщика хранилища ролей можно создать пользовательский тип роли. Не требуется реализовывать определенный интерфейс, но он должен иметь `Id` и, как правило, имеет свойство `Name`.

Ниже приведен пример класса role:

[!code-csharp[](identity-custom-storage-providers/sample/CustomIdentityProviderSample/CustomProvider/ApplicationRole.cs)]

## <a name="customize-the-role-store"></a>Настройка хранилища ролей

Можно создать класс `RoleStore`, предоставляющий методы для всех операций с данными в ролях. Этот класс эквивалентен классу [ролесторе&lt;троле&gt;](/dotnet/api/microsoft.aspnetcore.identity.entityframeworkcore.rolestore-1) . В классе `RoleStore` реализуется `IRoleStore<TRole>` и, при необходимости, интерфейс `IQueryableRoleStore<TRole>`.

* **Иролесторе&lt;троле&gt;**  
 Интерфейс [&gt;иролесторе&lt;троле](/dotnet/api/microsoft.aspnetcore.identity.irolestore-1) определяет методы для реализации в классе хранилища ролей. Он содержит методы для создания, обновления, удаления и получения ролей.
* **Ролесторе&lt;троле&gt;**  
 Чтобы настроить `RoleStore`, создайте класс, реализующий интерфейс `IRoleStore<TRole>`. 

## <a name="reconfigure-app-to-use-a-new-storage-provider"></a>Перенастройте приложение для использования нового поставщика хранилища

После реализации поставщика хранилища вы настраиваете приложение для его использования. Если приложение использовало поставщика по умолчанию, замените его пользовательским поставщиком.

1. Удалите `Microsoft.AspNetCore.EntityFramework.Identity` пакет NuGet.
1. Если поставщик хранилища находится в отдельном проекте или пакете, добавьте ссылку на него.
1. Замените все ссылки на `Microsoft.AspNetCore.EntityFramework.Identity` с помощью оператора using для пространства имен поставщика хранилища.
1. В методе `ConfigureServices` измените метод `AddIdentity`, чтобы использовать пользовательские типы. Для этой цели можно создать собственные методы расширения. Пример см. в разделе [идентитисервицеколлектионекстенсионс](https://github.com/aspnet/Identity/blob/rel/1.1.0/src/Microsoft.AspNetCore.Identity/IdentityServiceCollectionExtensions.cs) .
1. Если вы используете роли, обновите `RoleManager`, чтобы использовать класс `RoleStore`.
1. Обновите строку подключения и учетные данные в конфигурации приложения.

Пример:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // Add identity types
    services.AddIdentity<ApplicationUser, ApplicationRole>()
        .AddDefaultTokenProviders();

    // Identity Services
    services.AddTransient<IUserStore<ApplicationUser>, CustomUserStore>();
    services.AddTransient<IRoleStore<ApplicationRole>, CustomRoleStore>();
    string connectionString = Configuration.GetConnectionString("DefaultConnection");
    services.AddTransient<SqlConnection>(e => new SqlConnection(connectionString));
    services.AddTransient<DapperUsersTable>();

    // additional configuration
}
```

## <a name="references"></a>Ссылки

* [Пользовательские поставщики хранилища для удостоверения ASP.NET 4. x](/aspnet/identity/overview/extensibility/overview-of-custom-storage-providers-for-aspnet-identity)
* [ASP.NET Core Identity](https://github.com/dotnet/AspNetCore/tree/master/src/Identity) &ndash; этот репозиторий содержит ссылки на поставщиков хранилищ, поддерживаемых сообществом.
