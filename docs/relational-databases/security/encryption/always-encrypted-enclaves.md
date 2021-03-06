---
title: "Always Encrypted with Secure Enclaves (Database Engine) | Microsoft Docs"
ms.custom: ""
ms.date: 06/26/2019
ms.prod: sql
ms.prod_service: "database-engine, sql-database"
ms.reviewer: ""
ms.technology: security
ms.topic: conceptual
author: jaszymas
ms.author: jaszymas
monikerRange: ">= sql-server-ver15 || = sqlallproducts-allversions"
---
# Always Encrypted with Secure Enclaves
[!INCLUDE[tsql-appliesto-ssver15-xxxx-xxxx-xxx](../../../includes/tsql-appliesto-ssver15-xxxx-xxxx-xxx.md)]

 
Always Encrypted with secure enclaves provides additional functionality to the [Always Encrypted](always-encrypted-database-engine.md) feature.

Introduced in SQL Server 2016, Always Encrypted protects the confidentiality of sensitive data from malware and high-privileged *unauthorized* users of SQL Server. High-privileged unauthorized users are DBAs, computer admins, cloud admins, or anyone else who has legitimate access to server instances, hardware, etc., but who should not have access to some or all of the actual data. 

Until now, Always Encrypted protected the data by encrypting it on the client side and never allowing the data or the corresponding cryptographic keys to appear in plaintext inside the SQL Server Engine. As a result, the functionality on encrypted columns inside the database was severely restricted. The only operations SQL Server could perform on encrypted data were equality comparisons (and equality comparisons were only available with deterministic encryption). All other operations, including cryptographic operations (initial data encryption or key rotation), or rich computations (for example, pattern matching) were not supported inside the database. Users needed to move the data outside of the database to perform these operations on the client-side.

Always Encrypted *with secure enclaves* addresses these limitations by allowing computations on plaintext data inside a secure enclave on the server side. A secure enclave is a protected region of memory within the SQL Server process, and acts as a trusted execution environment for processing sensitive data inside the SQL Server engine. A secure enclave appears as a black box to the rest of the SQL Server and other processes on the hosting machine. There is no way to view any data or code inside the enclave from the outside, even with a debugger.  


Always Encrypted uses secure enclaves as illustrated in the following diagram:

![data flow](./media/always-encrypted-enclaves/ae-data-flow.png)



When parsing an application's query, the SQL Server Engine determines if the query contains any operations on encrypted data that require the use of the secure enclave. For queries where the secure enclave needs to be accessed:

- The client driver sends the column encryption keys required for the operations to the secure enclave (over a secure channel). 
- Then, the client driver submits the query for execution along with the encrypted query parameters.

During query processing, the data or the column encryption keys are not exposed in plaintext in the SQL Server Engine outside of the secure enclave. The SQL Server Engine delegates cryptographic operations and computations on encrypted columns to the secure enclave. If needed, the secure enclave decrypts the query parameters and/or the data stored in encrypted columns and performs the requested operations.

## Why use Always Encrypted with secure enclaves?

With secure enclaves, Always Encrypted protects the confidentiality of sensitive data while providing the following benefits:

- **In-place encryption** - cryptographic operations on sensitive data, for example: initial data encryption or rotating a column encryption key, are performed inside the secure enclave and do not require moving the data outside of the database. You can issue in-place encryption using the ALTER TABLE Transact-SQL statement, and you do not need to use tools, such as the Always Encrypted wizard in SSMS or the Set-SqlColumnEncryption PowerShell cmdlet.

- **Rich computations (preview)** - operations on encrypted columns, including pattern matching (the LIKE predicate) and range comparisons, are supported inside the secure enclave, which unlocks Always Encrypted to a broad range of applications and scenarios that require such computations to be performed inside the database system.

> [!IMPORTANT]
> In [!INCLUDE[sql-server-2019](../../../includes/sssqlv15-md.md)], rich computations are pending several performance optimizations and error-handling enhancements, and are currently disabled by default. To enable rich computations, see  [Enable rich computations](configure-always-encrypted-enclaves.md#configure-a-secure-enclave).

In [!INCLUDE[sql-server-2019](../../../includes/sssqlv15-md.md)], Always Encrypted with secure enclaves uses [Virtualization-based Security (VBS)](https://www.microsoft.com/security/blog/2018/06/05/virtualization-based-security-vbs-memory-enclaves-data-protection-through-isolation/) secure memory enclaves (also known as Virtual Secure Mode, or VSM enclaves) in Windows.

## Secure Enclave Attestation

The secure enclave inside the SQL Server Engine can access sensitive data stored in encrypted database columns and the corresponding column encryption keys in plaintext. Before submitting a query that involves enclave computations to SQL Server, the client driver inside the application must verify the secure enclave is a genuine enclave based on a given technology (for example, VBS) and the code running inside the enclave has been signed for running inside the enclave. 

The process of verifying the enclave is called **enclave attestation**, and it usually involves both a client driver within the application and SQL Server contacting an external attestation service. The specifics of the attestation process depend on the enclave technology and the attestation service.

The attestation process SQL Server supports for VBS secure enclaves in [!INCLUDE[sql-server-2019](../../../includes/sssqlv15-md.md)] is Windows Defender System Guard runtime attestation, which uses Host Guardian Service (HGS) as an attestation service. You need to configure HGS in your environment and register the machine hosting your SQL Server instance in HGS. You also must configure you client applications or tools (for example, SQL Server Management Studio) with an HGS attestation.

## Secure Enclave Providers

To use Always Encrypted with secure enclaves, an application must use a client driver that supports the feature. In [!INCLUDE[sql-server-2019](../../../includes/sssqlv15-md.md)], your applications must use .NET Framework 4.7.2 and .NET Framework Data Provider for SQL Server. In addition, .NET applications must be configured with a **secure enclave provider** specific to the enclave type (for example, VBS) and the attestation service (for example, HGS), you are using. The supported enclave providers are shipped separately in a NuGet package, which you need to integrate with your application. An enclave provider implements the client-side logic for the attestation protocol and for establishing a secure channel with a secure enclave of a given type.

## Enclave-enabled Keys

Always Encrypted with secure enclaves introduces the concept of enclave-enabled keys:

- **Enclave-enabled column master key** - a column master key that has the ENCLAVE_COMPUTATIONS property specified in the column master key metadata object inside the database. The column master key metadata object must also contain a valid signature of the metadata properties.
- **Enclave-enabled column encryption key** - a column encryption key that is encrypted with an enclave-enabled column master key.

When the SQL Server Engine determines operations, specified in a query, need to be performed inside the secure enclave, the SQL Server Engine requests the client driver shares the column encryption keys that are needed for the computations with the secure enclave. The client driver shares the column encryption keys only if the keys are enclave-enabled (that is, encrypted with enclave-enabled column master keys) and they're properly signed. Otherwise, the query fails.

## Enclave-enabled Columns

An enclave-enabled column is a database column encrypted with an enclave-enabled column encryption key. The functionality available for an enclave-enabled column depends on the encryption type the column is using.

- **Deterministic encryption** - Enclave-enabled columns using deterministic encryption support in-place encryption, but no other operations inside the secure enclave. Equality comparison is supported, but it is performed by comparing the ciphertext outside of the enclave.  
- **Randomized encryption** - Enclave-enabled columns using randomized encryption support in-place encryption as well as rich computations inside the secure enclave. The supported rich computations are pattern matching and [comparison operators](../../../t-sql/language-elements/comparison-operators-transact-sql.md), including equality comparison.

For more information about encryption types, see [Always Encrypted Cryptography](always-encrypted-cryptography.md).

The following table summarizes the functionality available for encrypted columns, depending on whether the columns use enclave-enabled column encryption keys and an encryption
type.

| **Operation**| **Column is NOT enclave-enabled** |**Column is NOT enclave-enabled**| **Column is enclave-enabled**  |**Column is enclave-enabled** |
|:---|:---|:---|:---|:---|
| | **Randomized encryption**  | **Deterministic encryption**     | **Randomized encryption**      | **Deterministic encryption**     |
| **In-place encryption** | Not Supported  | Not Supported   | Supported         | Supported    |
| **Equality comparison**   | Not Supported | Supported outside of the enclave | Supported (inside the enclave) | Supported outside of the enclave |
| **Comparison operators beyond equality** | Not Supported  | Not Supported   | Supported      | Not Supported     |
| **LIKE**    | Not Supported      | Not Supported    | Supported     | Not Supported    |

In-place encryption includes support for the following operations inside the enclave:

- Initial encryption of data stored in an existing column.
- Re-encrypting existing data in a column, for example:
  
  - Rotating the column encryption key (re-encrypting the column with a new key).
  - Changing the encryption type.  

- Decrypting data stored in an encrypted column (converting the column into a plaintext column).

For in-place encryption to be possible, the column encryption key (or keys), involved in the cryptographic operations, must be enclave-enabled:

- Initial encryption: the column encryption key for the column being encrypted must be enclave-enabled.
- Re-encryption: both the current and the target column encryption key (if different than the current key) must be enclave-enabled.
- Decryption: the current column encryption key of the column must be enclave-enabled.

### Indexes on Enclave-enabled Columns using Randomized Encryption
You can create nonclustered indexes on enclave-enabled columns using randomized encryption to make rich queries run faster. To ensure an index on a column encrypted using randomized encryption doesn't leak sensitive data and, at the same time, it's useful for processing queries inside the enclave, the key values in the index data structure (B-tree) are encrypted and sorted based on their plaintext values. When the query executor in the SQL Server Engine uses an index on an encrypted column for computations inside the enclave, it searches the index to look up specific values stored in the column. Each search may involve multiple comparisons. The query executor delegates each comparison to the enclave, which decrypts a value stored in the column and the encrypted index key value to be compared, it performs the comparison on plaintext and it returns the result of the comparison to the executor. For general information, not specific to Always Encrypted, on how indexing in SQL Server works, see [Clustered and Nonclustered Indexes Described](../../indexes/clustered-and-nonclustered-indexes-described.md).

Because an index on an enclave-enabled column using randomized encryption stores the encrypted index key values while the values are sorted based on plaintext, SQL Server Engine must use the enclave for any operation that involves creating or updating an index, including:
- Creating or rebuilding an index.
- Inserting, updating, or deleting a row in the table (containing an indexed/encrypted column), which triggers inserting or/and removing an index key to/from the index.
- Running DBCC commands that involve checking the integrity of indexes, for example [DBCC CHECKDB (Transact-SQL)](../../../t-sql/database-console-commands/dbcc-checkdb-transact-sql.md) or [DBCC CHECKTABLE (Transact-SQL)](../../../t-sql/database-console-commands/dbcc-checktable-transact-sql.md).
- Database recovery (for example, after SQL Server fails and restarts), if SQL Server needs to undo any changes to the index (see more details below).

All the above operations require the enclave has the column encryption key for the indexed column, so that it can decrypt the index keys. In general, the enclave can obtain a column encryption key in one of two ways:

- **Directly from the client application** that invoked the operation on the index, as described in the introduction above. This method requires the application or the user to have access to the column master key and the column encryption key, protecting the indexed column. The application must connect to the database with Always Encrypted enabled for the connection.
- **From the cache of column encryption keys.** The enclave stores the keys, used in previous queries, in the cache that is located inside the enclave, and is therefore inaccessible from the outside. If an application triggers an operation on an index without providing the required column encryption key directly, the enclave looks up the key in the cache. If the enclave finds the key in the cache, it uses it for the operation. This method allows DBAs to manage indexes and perform certain data cleansing operations (for example, removing a row from a table containing an index on an encrypted column), without having access to the cryptographic keys or the data in plaintext. This method requires the application connects to the database without Always Encrypted enabled for the connection.

Creating indexes on columns that use randomized encryption and are not enclave-enabled remains unsupported.

#### Database Recovery

If an instance of SQL Server fails, its databases may be left in a state where the data files may contain some modifications from incomplete transactions. When the instance is started, it runs a process called [database recovery](../../logs/the-transaction-log-sql-server.md#recovery-of-all-incomplete-transactions-when--is-started), which involves rolling back every incomplete transaction found in the transaction log to make sure the integrity of the database is preserved. If an incomplete transaction made any changes to an index, those changes also need to be undone. For example, some key values in the index may need to be removed or reinserted.

> [!IMPORTANT]
> Microsoft strongly recommends enabling [Accelerated database recovery (ADR)](../../backup-restore/restore-and-recovery-overview-sql-server.md#adr) for your database, **before** creating the first index on an enclave-enabled column encrypted with randomized encryption.

With the [traditional database recovery process](https://docs.microsoft.com/azure/sql-database/sql-database-accelerated-database-recovery#the-current-database-recovery-process) (that follows the [ARIES](https://people.eecs.berkeley.edu/~brewer/cs262/Aries.pdf) recovery model), to undo a change to an index, SQL Server needs to wait until an application provides the column encryption key for the column to the enclave, which can take a long time. ADR dramatically reduces the number of undo operations that must be deferred because a column encryption key is not available in the cache inside the enclave. Consequently, it substantially increases the database availability by minimizing a chance for a new transaction to get blocked. With ADR enabled, SQL Server still may need a column encryption key to complete cleaning up old data versions but it does that as a background task that does not impact the availability of the database or user transactions. You may, however, see error messages in the error log, indicating failed cleanup operations due to a missing column encryption key.

### Indexes on Enclave-enabled Columns using Deterministic Encryption

An index on a column using deterministic encryption are sorted based on ciphertext (not plaintext), regardless if the column is enclave-enabled or not.

## Security Considerations

The following security considerations apply to Always Encrypted with secure enclaves.

- The security of your data inside the enclave depends on an attestation protocol and an attestation service. Therefore, you need to ensure the attestation service and attestation policies, the attestation service enforces, are managed by a trusted administrator. Also, attestation services typically support different policies and attestation protocols, some of which perform minimal verification of the enclave and its environment, and are designed for testing and development. Closely follow the guidelines specific to your attestation service to ensure you are using the recommended configurations and policies for your production deployments. 
- Encrypting a column using randomized encryption with an enclave-enabled CEK may result in leaking the order of data stored in the column, as such columns support range comparisons. For example, if an encrypted column, containing employee salaries, has an index, a malicious DBA could scan the index to find the maximum encrypted salary value and identify a person with the maximum salary (assuming the name of the person is not encrypted). 
- If you use Always Encrypted to protect sensitive data from unauthorized access by DBAs, do not share the column master keys or column encryption keys with the DBAs. A DBA can manage indexes on encrypted columns without having direct access to the keys, by leveraging the cache of column encryption keys inside the enclave.

## Considerations for AlwaysOn and Database Migration

When configuring an AlwaysOn availability group that is required to support queries using enclaves, you need to ensure that all SQL Server instances hosting the databases in the availability group support Always Encrypted with secure enclaves and have an enclave configured. If the primary database supports enclaves, but a secondary replica does not, any query that attempts to use the functionality of Always Encrypted with secure enclaves will fail.

When you restore a backup file of a database that uses the functionality of Always Encrypted with secure enclaves on a SQL Server instance that doesn't have the enclave configured, the restore operation will succeed and all the functionality that doesn't rely on the enclave will be available. However, any subsequent queries using the enclave functionality will fail, and indexes on enclave-enabled columns using randomized encryption will become invalid.  The same applies when you attach a database using Always Encrypted with secure enclaves on the instance that doesn't have the enclave configured.

If your database contains indexes on enclave-enabled columns using randomized encryption, make sure you enable  [Accelerated database recovery (ADR)](../../backup-restore/restore-and-recovery-overview-sql-server.md#adr) in the database before creating a database backup. ADR will ensure the database, including the indexes, is available immediately after you restore the database. For more information, see [Database Recovery](#database-recovery).

When you migrate your database using a bacpac file, you need to make sure you drop all indexes enclave-enabled columns using randomized encryption before creating the bacpac file.

## Known Limitations

Secure enclaves enhance the functionality of Always Encrypted. The following capabilities **are now supported for enclave-enabled columns**:

- In-place cryptographic operations.
- Pattern matching (LIKE) and comparison operators on column encrypted using randomized encryption.
    > [!NOTE]
    > The above operations are supported for character string columns that use collations with a binary2 sort order (BIN2 collations). Character string columns using non-BIN2 collations can be encrypted using randomized encryption and enclave-enabled column encryption keys. However, the only new functionality that is enabled for such columns is in-place encryption.
- Creating nonclustered indexes on columns using randomized encryption.
- Computed columns using expressions containing the LIKE predicate and comparison operators on columns using randomized encryption.

All other limitations (not addressed by the above enhancements) that are listed for Always Encrypted (without secure enclaves) at [Feature Details](always-encrypted-database-engine.md#feature-details) also apply to Always Encrypted with secure enclaves.

The following limitations are specific to Always Encrypted with secure enclaves:

- Clustered indexes can't be created on enclave-enabled columns using randomized encryption.
- Enclave-enabled columns using randomized encryption can't be primary key columns and cannot be referenced by foreign key constraints or unique key constraints.
- Hash joins and merged joins on enclave-enabled columns using randomized encryption are not supported. Only nested loop joins (using indexes, if available) are supported.
- Queries with the LIKE operator or a comparison operator that has a query parameter using one of the following data types (that become large objects after encryption) ignore indexes and perform table scans.
    - nchar[n] and nvarchar[n], if n is greater than 3967.
    - char[n], varchar[n], binary[n], varbinary[n], if n is greater than 7935.
- In-place cryptographic operations cannot be combined with any other changes of column metadata, except changing a collation within the same code page and nullability. For example, you cannot encrypt, re-encrypt, or decrypt a column AND change a data type of the column in a single ALTER TABLE or ALTER COLUMN Transact-SQL statement. Use two separate statements.
- Using enclave-enabled keys for columns in in-memory tables isn't supported.
- The only supported key stores for storing enclave-enabled column master keys are Windows Certificate Store and Azure Key Vault.

The following limitations apply to [!INCLUDE[sql-server-2019](../../../includes/sssqlv15-md.md)], but are on the roadmap to be addressed:

- Creating statistics for enclave-enabled columns using randomized encryption is not supported.
- The only client driver supporting Always Encrypted with secure enclaves is .NET Framework Data Provider for SQL Server (ADO.NET) in .NET Framework 4.7.2. ODBC/JDBC isn't supported.
- Tooling support for Always Encrypted with secure enclaves is currently incomplete. In particular:
  - Import/exporting databases containing enclave-enabled keys is not supported.
  - To trigger an in-place cryptographic operation via an ALTER TABLE Transact-SQL statement, you need to issue the statement using a query window in SSMS, or you can write your own program that issues the statement. The Set-SqlColumnEncryption cmdlet in the SqlServer PowerShell module and the Always Encrypted wizard in SQL Server Management Studio do not support in-place encryption yet - both tools currently move the data out of the database for cryptographic operations, even if the column encryption keys used for the operations are enclave-enabled.
- DBCC commands checking the integrity of indexes or updating indexes are not supported.
- Creating indexes on encrypted columns at the time of creating the table (via CREATE TABLE). You need to create an index on an encrypted column separately via CREATE INDEX.

## Next steps

- Set up your test environment and try the functionality of Always Encrypted with secure enclaves in SSMS - see [Tutorial: Getting started with Always Encrypted with secure enclaves using SSMS](../tutorial-getting-started-with-always-encrypted-enclaves.md).
- For details on how to use Always Encrypted with secure enclaves, see [Configure Always Encrypted with secure enclaves](configure-always-encrypted-enclaves.md).
