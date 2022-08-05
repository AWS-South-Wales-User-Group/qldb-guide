---
title: "Security"
date: 2021-07-30T21:17:45+01:00
weight: 85
draft: false
---

## Encryption at rest

All data stored in Amazon QLDB is fully encrypted at rest by default. QLDB encryption at rest provides enhanced security by encrypting all ledger data at rest using encryption keys in AWS Key Management Service (AWS KMS).

When QLDB was first launched, it only supported AWS owned keys. Amazon QLDB launched support for [customer managed AWS KMS keys](https://aws.amazon.com/about-aws/whats-new/2021/07/amazon-qldb-supports-customer-managed-kms-keys/) on July 22, 2021.

In QLDB, you can specify the type of AWS KMS key for each ledger resource. When you create a new ledger or update an existing ledger, you can choose one of the following types of KMS keys to protect your ledger data:

* AWS owned key – The default encryption type. The key is owned by QLDB (no additional charge).
* Customer managed key – The key is stored in your AWS account and is created, owned, and managed by you. You have full control over the key (AWS KMS charges apply).

A fascinating insight into the design behind this was provided in this [Twitter thread](https://twitter.com/marcbowes/status/1418640809155461121) by Marc Bowes which states:

> The feature itself is really simple. We *always* encrypt ledgers. Usually, with keys we control. 
> Now, customers can bring their own keys! Importantly, if they remove our access, then there is no 
> way we (Amazon) can access the data. This is an important regulatory requirement.
>
> However, the feature goes a little bit beyond that. You can also *change* the key used to encrypt 
> the data. This is where things get interesting, because having to decrypt-encrypt huge amounts of 
> data would be a showstopper.
>
> To solve this problem, we create a hierarchy of data keys. The structure is actually pretty 
> complicated, and designing it took some care. What it allows us to do is decrypt-encrypt the *keys* 
> and not *the data*. Weirdly, this design also improves several security considerations!
>
> We can perform a full key change in under 30 minutes regardless of the amount of data. Even better, 
> it costs customers ZERO dollars to change their key, either from Amazon-owned to customer-owned or
> from one key to another.


## Encryption in transit

Amazon QLDB only accepts secure connections that use the HTTPS protocol, which protects network traffic by using Secure Sockets Layer (SSL)/Transport Layer Security (TLS). Encryption in transit provides an additional layer of data protection by encrypting your data as it travels to and from QLDB.

Amazon QLDB only accepts secure connections that use the HTTPS protocol, which protects network traffic by using Secure Sockets Layer (SSL)/Transport Layer Security (TLS). Encryption in transit provides an additional layer of data protection by encrypting your data as it travels to and from QLDB.