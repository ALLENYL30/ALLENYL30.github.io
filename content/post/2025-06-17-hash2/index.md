+++
author = "yuhao"
title = "A Tour of Hash Functions: From MD5 to Argon2 and Beyond"
date = "2025-06-16"
description = "In a previous article, we explored the topic of salted password hashing. This time, we'll expand on that by taking a broader look at the world of hash functions."
tags = [
    "C#",
    ".NET",
    "Security",
]
categories = [
    "Programming/Development",
]
+++
In a previous article, we explored the topic of salted password hashing. This time, we'll expand on that by taking a broader look at the world of hash functions.

Hash functions are crucial tools in computer science and cryptography, essential for tasks like data integrity verification, digital signatures, and secure password storage. There is a wide variety of hash functions, some common and others less so. This article will introduce several of them, highlighting their characteristics and primary use cases.

### Early Hash Functions

When we talk about early hash functions, **MD5** and **SHA-1** are the most widely known. (Of course, even earlier ones like MD4 and SHA-0 exist, but they have long fallen out of use). MD5 and SHA-1 were extremely popular in the 1990s and early 2000s, widely used for file integrity checks, digital signatures, and more.

Although they were once ubiquitous, they are no longer recommended for security-sensitive applications due to known vulnerabilities. Despite this, they remain popular. For instance, many software download sites still provide MD5 or SHA-1 checksums for users to verify that their downloaded files are complete and untampered with. This highlights a common phenomenon: even when a technology has known flaws—or even severe vulnerabilities—its popularity can ensure its continued use. It’s similar to the JPG image format, which, despite its shortcomings (no transparency, lossy compression, prone to artifacts), remains one of the most used image formats today.

So, what are their security issues? In short, both MD5 and SHA-1 are vulnerable to **collision attacks**. A collision occurs when two different inputs produce the same hash value. This means an attacker could potentially create a malicious file with the same hash as a legitimate one, thereby bypassing integrity checks.

Consider the file download verification scenario mentioned earlier. A website tells you a file's hash is `d41d` (using just the first four characters for simplicity). You download the file, compute its hash, and it matches. Normally, you would feel confident that the file is authentic. However, an attacker could craft a malicious file that also produces the hash `d41d` and trick you into downloading it instead. After verifying the hash, you would mistakenly trust the file and run a virus. This could also be used to forge certificate files, leading to severe consequences.

Even so, executing a collision attack is still computationally expensive. This, combined with MD5's widespread adoption and fast computation speed, means it is still used in many non-security-critical scenarios to quickly verify file integrity.

### The SHA-2 Family

Due to the security flaws in MD5 and SHA-1, the **SHA-2** family (including **SHA-256**, **SHA-384**, **SHA-512**, etc.) emerged as the new standard. SHA-2 functions are more complex in design and offer a much higher level of security. They are widely used in digital signatures, certificate authorities (CAs), and other security-critical areas.

When using SHA-2 to store user passwords in a database, it is standard practice to add a random **salt**. This prevents **rainbow table attacks**, where an attacker pre-computes the hashes of common passwords and stores them in a lookup table. By adding a unique salt to each password before hashing, the resulting hashes will be different even if two users have the same password, making such attacks infeasible.

Here is a C# example of hashing a password with SHA-256 and a salt:

```csharp
using System;
using System.Text;
using System.Security.Cryptography;

static byte[] GenerateSalt()
{
    // The method below will show an obsoletion warning in modern .NET
    using (var rng = new RNGCryptoServiceProvider())
    {
        byte[] salt = new byte[16];
        rng.GetBytes(salt);
        return salt;
    }

    // You should use the following instead:
    // return RandomNumberGenerator.GetBytes(16);
}

static byte[] HashPassword(string password, byte[] salt)
{
    using (var sha256 = SHA256.Create())
    {
        byte[] passwordBytes = Encoding.UTF8.GetBytes(password);
        byte[] saltedPassword = new byte[passwordBytes.Length + salt.Length];
        Buffer.BlockCopy(passwordBytes, 0, saltedPassword, 0, passwordBytes.Length);
        Buffer.BlockCopy(salt, 0, saltedPassword, passwordBytes.Length, salt.Length);
        return sha256.ComputeHash(saltedPassword);
    }
    
    // Newer .NET versions also provide more convenient static methods, like SHA256.HashData().
}
```

You would then store both the salted hash and the salt in the database. To verify a password, retrieve the salt from the database, apply it to the user's input, hash it using the same function, and compare it to the stored hash.

Don't worry about the salt being exposed. Its purpose is not to be secret but to ensure the uniqueness of each hash, rendering pre-computed tables useless.

### The SHA-3 Family

While SHA-2 resolved the security issues of its predecessors, it is still based on the same Merkle-Damgård construction. Concerned that cryptanalytic techniques could eventually be extended to break the SHA-2 family and with the rise of quantum computing, the U.S. National Institute of Standards and Technology (NIST) released the **SHA-3** standard in 2015.

SHA-3 is based on a completely different design philosophy, the **Keccak** algorithm. It not only provides higher security but also offers flexible output lengths (e.g., SHA3-224, SHA3-256, SHA3-384, SHA3-512).

However, SHA-3 is not yet widely adopted. It currently serves more as a backup for SHA-2, ready to be deployed if a critical vulnerability is ever found in SHA-2. Furthermore, while it offers excellent security and flexibility, other specialized algorithms are often preferred for specific use cases like password hashing.

### PBKDF2

To further increase the difficulty of cracking passwords beyond just adding a salt, we can introduce **iterations**. **PBKDF2** (Password-Based Key Derivation Function 2) is a popular function that does exactly this. It increases the computational work required for a brute-force attack by repeatedly applying a hash function.

PBKDF2 takes a password and a salt and runs them through a hash function thousands of times. The more iterations, the more difficult it is to crack. It is commonly used for password storage and key derivation.

In C#, you can use the `Rfc2898DeriveBytes` class to implement it:

```csharp
using System;
using System.Security.Cryptography;
using System.Text;

static byte[] HashPasswordWithPBKDF2(string password, byte[] salt, int iterations = 10000)
{
    using var pbkdf2 = new Rfc2898DeriveBytes(password, salt, iterations, HashAlgorithmName.SHA256);
    return pbkdf2.GetBytes(32); // Generate a 32-byte hash
}
```

### bcrypt and Argon2

Unfortunately, even PBKDF2 can be vulnerable to attacks using modern hardware, especially the massive parallel processing power of GPUs. To address this, more advanced password hashing functions like **bcrypt** and **Argon2** were developed.

**bcrypt** is a password hashing function based on the Blowfish cipher. It is designed to be slow and computationally intensive. A key feature of bcrypt is its adjustable **cost factor**, which controls the hashing time. Unlike simple iterations, the cost factor is exponential—incrementing it by one doubles the computation time. In C#, you can use libraries like `BCrypt.Net-Next` to implement bcrypt.

**Argon2** was the winner of the Password Hashing Competition in 2015 and is considered one of the most secure password hashing functions available today. It is highly configurable, allowing you to tune its memory usage, iteration count, and degree of parallelism to provide robust security. Argon2 comes in three variants: **Argon2d**, **Argon2i**, and **Argon2id**, each designed to protect against different types of attacks. In C#, you can use libraries like `Konscious.Security.Cryptography.Argon2`.

### BLAKE2

**BLAKE2** is a relatively new hash function (published in 2013) that strikes an excellent balance between speed and security. It was designed to be faster than MD5 and SHA-1 while providing higher security than SHA-2. It is well-suited for file integrity verification and even password hashing.

Its high performance comes from leveraging modern CPU SIMD instruction sets (like SSE2/AVX), making it especially efficient on multi-core processors. It comes in two main versions: **BLAKE2b** (for 64-bit platforms, with a variable output up to 64 bytes) and **BLAKE2s** (for 8- to 32-bit platforms, with a variable output up to 32 bytes). It also includes features like a built-in keying mechanism, optional salt, and personalization strings, making it very versatile.

In C#, you can use libraries like BouncyCastle to implement BLAKE2:

```csharp
using Org.BouncyCastle.Crypto;
using Org.BouncyCastle.Crypto.Digests;

var digest = new Blake2bDigest(512); // Output is 512 bits by default
digest.BlockUpdate(data, 0, data.Length);
var hash = new byte[digest.GetDigestSize()];
digest.DoFinal(hash, 0);
```

### SM3

Finally, let's introduce a Chinese national standard hash function: **SM3**. Released by the China State Cryptography Administration in 2010, SM3 is an independently designed hash algorithm that does not rely on foreign standards, which is significant for national and information security.

The SM3 algorithm produces a 256-bit hash value and is comparable to SHA-256 in terms of security and efficiency. It is designed to be resistant to collision and preimage attacks and is seeing increasing adoption in China, especially in the finance, government, and defense sectors.

### Summary

The world of hash functions is diverse, with each function having its own strengths. From the early MD5 and SHA-1 to the modern SHA-2, SHA-3, PBKDF2, bcrypt, Argon2, BLAKE2, and SM3, every algorithm was designed with a specific purpose in mind.

Here’s a quick guide for common needs:

* **Data Integrity Verification:** MD5, SHA-1 (for non-critical uses), SHA-2, BLAKE2
* **Password Storage:** PBKDF2, bcrypt, Argon2
* **Digital Signatures:** SHA-2, SHA-3

