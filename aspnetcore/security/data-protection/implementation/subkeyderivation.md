---
title: Формирование подключа и шифрование с проверкой подлинности в ASP.NET Core
author: rick-anderson
description: Узнайте подробности реализации защиты данных в ASP.NET Core подраздела наследования и проверку подлинности шифрования.
ms.author: riande
ms.date: 10/14/2016
uid: security/data-protection/implementation/subkeyderivation
ms.openlocfilehash: bbfde378755b09cd5b1217b8cf66249b9fa1d6ad
ms.sourcegitcommit: 8516b586541e6ba402e57228e356639b85dfb2b9
ms.translationtype: MT
ms.contentlocale: ru-RU
ms.lasthandoff: 07/11/2019
ms.locfileid: "67814378"
---
# <a name="subkey-derivation-and-authenticated-encryption-in-aspnet-core"></a>Формирование подключа и шифрование с проверкой подлинности в ASP.NET Core

<a name="data-protection-implementation-subkey-derivation"></a>

Большинство ключей в связку ключей будет содержать определенные виды энтропии и будет иметь алгоритмического сведения о том, «шифрование в режиме CBC + HMAC проверки» или «шифрование GCM + проверки». В таких случаях мы называем embedded энтропии материала основного (или км) для этого ключа, и мы выполняем функцию формирования ключа для получения ключей, которые будут использоваться для фактических криптографических операций.

> [!NOTE]
> Ключи являются абстрактными, и пользовательская реализация может работать не так, как показано ниже. Если ключа обеспечивает собственную реализацию `IAuthenticatedEncryptor` вместо того чтобы использовать один из встроенных фабрик, больше не применяется механизм, описанные в этом разделе.

<a name="data-protection-implementation-subkey-derivation-aad"></a>

## <a name="additional-authenticated-data-and-subkey-derivation"></a>Дополнительные данные прошедшего проверку подлинности и формирование подключа

`IAuthenticatedEncryptor` Интерфейс служит в качестве основной интерфейс для всех операций шифрование с проверкой подлинности. Его `Encrypt` метод принимает два буфера: открытого текста и additionalAuthenticatedData (AAD). Поток открытого текста содержимое без изменений вызов `IDataProtector.Protect`, но AAD, формируемые системой и состоит из трех компонентов:

1. 32-разрядный magic заголовок 09 F0 C9 F0, определяющий эта версия система защиты данных.

2. Идентификатор ключа 128 бит.

3. Строка переменной длины, сформированным из цепочки цели, которая создала `IDataProtector` , выполняющий эту операцию.

Так как AAD является уникальным для всех трех компонентов кортежа, мы используем его для формирования новых ключей из км вместо использования км сам во всех наших криптографических операций. Для каждого вызова к `IAuthenticatedEncryptor.Encrypt`, следующий процесс создания производных ключей:

( K_E, K_H ) = SP800_108_CTR_HMACSHA512(K_M, AAD, contextHeader || keyModifier)

Здесь мы звоним по номеру NIST SP800-108 Проблемы в режиме счетчика (см. в разделе [NIST SP800-108](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-108.pdf), 5.1 в секунду) со следующими параметрами:

* Ключ производного ключа (KDK) = K_M

* PRF = HMACSHA512

* Метка = additionalAuthenticatedData

* context = contextHeader || keyModifier

Заголовок контекста имеет переменную длину и по существу выступает в качестве отпечатком алгоритмы, для которых мы ваш класс порожден K_E и K_H. Модификатор ключа представляет собой строку, 128 бит, случайным образом формируется для каждого вызова `Encrypt` и позволяет добиться с помощью перегрузки вероятность, что KE и KH уникальны для этой операции шифрования проверки подлинности, даже если все другие входные данные для Формирования является константой.

Шифрование в режиме CBC + операций проверки HMAC | K_E | Длина ключа симметричного блочного шифра, и | K_H | — Это размер хэш-кода HMAC подпрограммы. Для шифрования GCM + операций проверки | K_H | = 0.

## <a name="cbc-mode-encryption--hmac-validation"></a>Шифрование в режиме CBC + HMAC проверки

После создания K_E посредством механизма выше мы создать случайный вектор инициализации и запуска алгоритм симметричного блочного шифрования для шифрования открытого текста. Вектор инициализации и зашифрованных данных, затем запускаются подпрограмма HMAC, инициализируется с помощью ключа K_H для создания на компьютере MAC. Этот процесс и возвращаемое значение представлены графически ниже.

![Процесс в режиме CBC и вернуться](subkeyderivation/_static/cbcprocess.png)

*output:= keyModifier || iv || E_cbc (K_E,iv,data) || HMAC(K_H, iv || E_cbc (K_E,iv,data))*

> [!NOTE]
> `IDataProtector.Protect` Реализация будет [спереди magic заголовка и идентификатор ключа](xref:security/data-protection/implementation/authenticated-encryption-details) выходные данные перед его возвратом вызывающей стороне. Поскольку magic заголовка и идентификатор ключа неявно являются частью [AAD](xref:security/data-protection/implementation/subkeyderivation#data-protection-implementation-subkey-derivation-aad), и так как модификатор ключа подается в качестве входных данных для Формирования, это означает, что каждый один байт, последний возвращенный полезных данных проверку подлинности с MAC.

## <a name="galoiscounter-mode-encryption--validation"></a>Режим Galois/шифрование и проверка

После создания K_E посредством механизма выше мы сформировать произвольного nonce 96-разрядного и запуска алгоритм шифрования симметричного блочного шифрования открытого текста и создания тега 128-разрядный проверки подлинности.

![Процесс GCM режима и вернуться](subkeyderivation/_static/galoisprocess.png)

*output := keyModifier || nonce || E_gcm (K_E,nonce,data) || authTag*

> [!NOTE]
> Несмотря на то, что GCM изначально поддерживает концепцию AAD, мы по-прежнему поступает AAD только к исходной Проблемы, предусмотренной передать пустую строку в GCM для его параметра AAD. Причиной этого является двухступенчатой. Во-первых, [для поддержки гибкости](xref:security/data-protection/implementation/context-headers#data-protection-implementation-context-headers) мы никогда не будут K_M непосредственно в качестве ключа шифрования. Кроме того GCM налагает уникальность очень строгие требования на своих входных данных. Вероятность, что процедура шифрования GCM когда-либо вызванный для двух или более различных наборов входных данных с тем же (ключ, nonce) пары не должен превышать 2 ^ 32. Если мы исправить K_E мы не может выполнять больше, чем 2 ^ 32 операции шифрования, прежде чем мы запуска от 2 ^ ограничить -32. Это может показаться очень много операций, но быстрых веб-сервера можно воспользоваться 4 миллиардов запросов в простое дней, в пределах обычный время существования для этих ключей. Обеспечить соответствие требованиям 2 ^ ограничение вероятность-32, мы будем продолжать использовать модификатор ключа 128 бит и 96-битного nonce, которая радикально расширяет число операций можно использовать для любого заданного K_M. Для упрощения разработки мы Формирования кода путь к общей папке между операциями CBC и GCM, а так как AAD уже считается в Проблемы нет необходимости переслать его подпрограмма GCM.
