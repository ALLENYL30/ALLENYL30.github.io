+++
author = "yuhao"
title = "Why You Must Salt and Hash Passwords: A Journey to Secure Storage"
date = "2025-06-05"
description = "The topic of salting and hashing passwords is a fundamental concept in security, but why is it so crucial? This article will guide you through the evolution of password storage, from insecure to..."
tags = [
    "C#",
    ".NET",
    "security",
]
categories = [
    "Programming/Development",
]
image = "cover.jpg"
+++
The topic of salting and hashing passwords is a fundamental concept in security, but why is it so crucial? This article will guide you through the evolution of password storage, from insecure to secure, to demonstrate the necessity of salted hashing and how to implement it.

We've all heard that passwords should never be stored in plaintext. Instead, they should be hashed, and more specifically, salted and hashed. What is the purpose and necessity of this process? And how can we achieve it in C#? Let's explore these questions by progressing from the least secure to the most secure methods.

### 1. Plaintext Storage

Let's start with the most insecure approach imaginable: storing the password as is.

```csharp
class User
{
    public int Id { get; set; } // Auto-incrementing primary key
    public string Username { get; set; }
    public string Password { get; set; } // Plaintext password
}
```

When we create a new user, the database stores the literal password.

```csharp
var user = new User
{
    Username = "admin",
    Password = "123456"
};
```

This is incredibly dangerous for several reasons:

* **If the database is local** (e.g., SQLite), anyone who gains access to the database file or decompiles the application to find the connection string can see every user's password.
* **If the database is remote**, attackers have numerous ways to access the data, including SQL injection, leaked SSH keys, exposed database backups, or other server vulnerabilities.

A plaintext password leak is catastrophic. It not only exposes private user information but also enables attackers to perform **credential stuffing** (using the leaked credentials to try and log into other websites). Therefore, storing passwords in plaintext must be avoided under all circumstances.

### 2. Simple Hashing (MD5 / SHA1)

Let's upgrade our code to store a hash of the password using an algorithm like MD5 or SHA1.

```csharp
class User
{
    public int Id { get; set; }
    public string Username { get; set; }
    public string PasswordHash { get; set; } // The password hash
}
```

We can create a helper class to handle hashing and verification:

```csharp
class PasswordHelper
{
    public static string HashPassword(string password)
    {
        using var md5 = MD5.Create();
        var hash = md5.ComputeHash(Encoding.UTF8.GetBytes(password));
        return Convert.ToBase64String(hash);
    }

    public static bool VerifyPassword(string password, string passwordHash)
    {
        return HashPassword(password) == passwordHash;
    }
}
```

In this method, we hash the password with MD5 and store it as a Base64 string. To verify, we simply hash the user's input and compare it to the stored hash.

> **Info:** Why Base64? Storing the hash as a Base64 string makes it human-readable in the database, unlike a binary BLOB. String-based columns are also often easier and more efficient to index. Many modern hashing algorithms produce string-based outputs by default.

A stored password hash might look like this: `ISMvKXpXpadDiUoOSoAfww==`

This looks much safer than plaintext. Unfortunately, to an attacker, it's only marginally better. This is because of **rainbow table attacks**. A rainbow table is a massive precomputed lookup table containing the hashes of common passwords. An attacker can compare the hashes in your database against their rainbow table to quickly find the original password.

For instance, the hash above corresponds to the plaintext "admin". An attacker with a rainbow table could crack this password instantly. Don't underestimate these tables; they can contain billions of entries. Unless your password is very complex, it's likely vulnerable.

Furthermore, algorithms like **MD5 and SHA1 are broken**. They are susceptible to **collisions**, where two different inputs produce the same hash value. An attacker might not find your original password, but if they find another string (e.g., "qwerty") that produces the same hash, they can use that to log in, as the server only compares the hash values.

### 3. Hashing with a Salt (SHA256)

To counter these attacks, we need to upgrade our algorithm and introduce a **salt**. We'll use SHA256 (part of the secure SHA-2 family) and a unique, random salt for each user.

```csharp
class User
{
    public int Id { get; set; }
    public string Username { get; set; }
    public string PasswordHash { get; set; }
    public string Salt { get; set; } // The salt
}
```

Now, we update our `PasswordHelper`:

```csharp
class PasswordHelper
{
    public static string HashPassword(string password, byte[] salt)
    {
        var passwordBytes = Encoding.UTF8.GetBytes(password);
        // Combine password and salt
        var combinedBytes = new byte[passwordBytes.Length + salt.Length];
        Buffer.BlockCopy(passwordBytes, 0, combinedBytes, 0, passwordBytes.Length);
        Buffer.BlockCopy(salt, 0, combinedBytes, passwordBytes.Length, salt.Length);

        var hash = SHA256.HashData(combinedBytes);
        return Convert.ToBase64String(hash);
    }

    public static bool VerifyPassword(string password, string passwordHash, byte[] salt)
    {
        return HashPassword(password, salt) == passwordHash;
    }
    
    public static byte[] GenerateSalt()
    {
        // A 16-byte (128-bit) salt is generally sufficient
        return RandomNumberGenerator.GetBytes(16);
    }
}
```

> **Tip:** Modern versions of .NET provide convenient static methods like `SHA256.HashData` and `RandomNumberGenerator.GetBytes`, removing the need to create instances. In the past, you might have seen `RNGCryptoServiceProvider`, which is now considered obsolete.

By combining the password with a unique salt before hashing, every resulting hash is unique, even if two users have the same password. This renders rainbow tables useless.

The salt must be stored in the database alongside the hash. During verification, you retrieve the user's salt and hash, apply the same salt to the entered password, and compare the results. Now, even if an attacker gets your database, they can't use precomputed tables to crack passwords.

### 4. Using a Key Derivation Function (PBKDF2)

Salting with SHA256 is quite secure, but we can do better. An attacker who has your database can still attempt a **brute-force attack** against each individual password. They can't use a rainbow table, but they can take one user's hash and salt and rapidly try millions of password combinations.

To combat this, our next step is to make the hashing process intentionally slow. We can achieve this with a **Password-Based Key Derivation Function (PBKDF2)**. C# provides an implementation in the `Rfc2898DeriveBytes` class.

```csharp
class PasswordHelper
{
    public static string HashPassword(string password, byte[] salt)
    {
        // PBKDF2 with 10,000 iterations and SHA256
        using var pbkdf2 = new Rfc2898DeriveBytes(password, salt, 10000, HashAlgorithmName.SHA256);
        // Get a 32-byte (256-bit) hash
        return Convert.ToBase64String(pbkdf2.GetBytes(32)); 
    }

    // Other methods remain the same
}
```

> **Info:** `Rfc2898` is the standard that specifies PBKDF2. You must specify a hash algorithm in the constructor, as the overload without it is now obsolete.

PBKDF2 introduces an **iteration count**. This forces the algorithm to repeat the hashing process thousands of times. The higher the count, the slower the computation, and the more difficult it is for an attacker to brute-force the password. A count of 10,000 is a reasonable starting point today, but this number should increase over time as computing power grows. This drastically increases the cost for an attacker to crack even a single password.

### 5. The Gold Standard: BCrypt and Argon2

Unfortunately, the arms race continues. Attackers can use specialized hardware like GPUs or FPGAs to accelerate the calculations for algorithms like PBKDF2. This brings us to our heavy hitters: **BCrypt** and **Argon2**. These algorithms are designed to be "memory-hard," making them resistant to GPU-based attacks.

Let's look at **BCrypt**. Since it's not in the .NET standard library, we can use a popular third-party library like `BCrypt.Net-Next`.

```csharp
class PasswordHelper
{
    public static string HashPassword(string password)
    {
        // The work factor (12) controls the complexity
        return BCrypt.Net.BCrypt.HashPassword(password, 12);
    }

    public static bool VerifyPassword(string password, string passwordHash)
    {
        return BCrypt.Net.BCrypt.Verify(password, passwordHash);
    }
}
```

Notice that our `User` class no longer needs a `Salt` property. BCrypt generates a salt internally and includes it in the final hash string.

A BCrypt hash looks like this: `$2a$12$lraBT1/lH3RiFXjQbywREutDElnBFaolPOEsDAvo1sjK2iRjwCAUi`

This string contains everything needed for verification: the algorithm identifier (`$2a$`), the work factor (`12`), the salt, and the hash. You only need to store this single string in your database.

> **Info:** The **work factor** is a power of 2 that controls the computational cost (e.g., a work factor of 12 means 2<sup>12</sup> = 4096 rounds). The higher the factor, the slower the hashing. A value between 10 and 12 is common today.

But what if an attacker is still determined to use their powerful hardware? This is where we bring out our ultimate weapon: **Argon2**.

Like BCrypt, Argon2 requires a third-party library, such as `Konscious.Security.Cryptography`. The implementation details are similar to BCrypt—it also produces a self-contained hash string. However, Argon2 is considered even more secure. It was the winner of the Password Hashing Competition (2013-2015) and is designed to be highly resistant to brute-force attacks by being tunable across three dimensions:

* **Time cost:** How many iterations to run (CPU-intensive).
* **Memory cost:** How much memory to use (memory-hard).
* **Parallelism:** How many threads to use.

Argon2 comes in three variants:

* **Argon2d:** Maximizes resistance to GPU cracking attacks.
* **Argon2i:** Optimized to resist side-channel attacks.
* **Argon2id:** A hybrid version that provides resistance to both side-channel and GPU attacks. It is the recommended choice for general-purpose password hashing.

With a robust algorithm like Argon2, you can be confident that passwords are protected against even the most determined attackers using current technology.

### Conclusion

In this article, we journeyed from the dangerously insecure practice of storing plaintext passwords to the modern, robust standards of password hashing. We saw how simple hashing is vulnerable to rainbow tables, how old algorithms are broken by collisions, and how salting defeats precomputation attacks. We then explored how key derivation functions like PBKDF2 and memory-hard functions like BCrypt and Argon2 raise the cost of brute-force attacks.

In practice, your choice of algorithm depends on your security requirements.

* For a small, low-risk project, **SHA256 with a strong, unique salt** is a decent baseline that is easy to implement.
* For any application where security is a priority, **BCrypt** or, even better, **Argon2** should be your go-to choice. They represent the current best practice for securing user passwords.

