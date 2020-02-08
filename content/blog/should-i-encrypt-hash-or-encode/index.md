---
title: Should I encrypt, hash or encode?
date: "2020-02-08T16:50:55.284Z"
description: Having a basic understanding of these terms can go a long way when writing code. 
---

I'm not a security expert, but as a software engineer I feel like it's
part of my job to do everything in my power to protect our customer's
data. Granted, the company pays security expert to get optimal result
in this regard, as they should, but I don't want to waste a highly
qualified security expert's time with very basic things. So I make
it part of my job to at least avoid what could be a common mistake
if I didn't understand the fundamental of these terms.

#### What is what?

Encoding is a process that transform data from one format into another.
It has very little to do with data security, but it might give a bad
sense of security for beginners in the field. In the context of web
development, encoding often comes up as a way of transferring data
from one page into another while respecting HTTP's limitation.
A good demonstration is redirection with parameters. HTTP dictates
that URL parameters comes after a `?` character. But what if you want
to send an actual `?` as a value of a parameter? .e.g `?name=?`.
Similarly, the ampersand character `&` is used to separate parameters
such as `?name=deleu&age=28`, but if you want to send an actual `&`
as a value of a parameter, encoding is a good strategy for it.
We can use URL Encode so that anything that comes out of the encoding
are considered valid characters that can be used in a URL without
conflicting with special characters that are interpreted for a specific
use case. Encoding such as `base64_encode` usually scramble the data
in a way that humans cannot read and might bring a **false** sense of
security. There is absolutely nothing secure about base64_encode
as anybody can decode it using online tools or any programming language.

Encryption, on the other hand, is a process designed to protect and
secure information. It dates back to ancient history and wars where
armies wanted to communicate with each other secretly. An important
difference from encoding, encryption will use a password or a secret
and scramble the data in a way that it can only return to it's original
state if we can figure out said password or a contra-password.
A key component about encryption is the ability to decipher it.
While writing software, I always keep in mind as a rule of thumb
that it is possible to "undo" the encryption and have the original
data again.

If you don't need to know the original data ever again, hashing is
the right choice. Extremely popular while talking about login on
various services, a password is stored in a hash format because
nobody needs to read it in it's original form. Not even the developer
or the database administrator should be able to see what is a person's
password. That is a confidential piece of data that was provided to
the system for protecting one's account and only that person has
to know it. I still remember when first learning about hashing
the natural thought occurring to me: But how am I going to check
if the user's password is correct if I cannot reverse it? The
answer is very simple, hash it again and if the same hash was
produced, then that piece of data must be the right password.

#### When to use which?

If you're new to all of this, knowing when to use which might not be
crystal clear yet and with experience it gets easier. My reasoning
process usually involve asking what data we're talking about and what
is the context that it's being used.

A very easy rule of thumb is to assume that there is **never** a reason
for you to need to know your user's passwords. If you write a login
process in your SaaS, hash their password. Nobody needs to know
their password except the people that create their own accounts.
Another common use case for hashing is to verify the integrity
of a piece of information. You may see a lot of software that show
their sha256 or other hash to provide data integrity. If you download
their software hash it yourself, you should get the exact same value
they provided. If not, it might mean that you didn't actually downloaded
the right information. This is commonly used in integration systems
for distribution. For instance, the Alpine Linux project checks for
the php binary hashing to make sure that they're not downloading
a malware and redistributing it to millions of people. You can hash
large data set reasonably fast.

A better demonstration of context presents itself when we talk about
encryption. Imagine, for the sake of argument, you have a SaaS product
and you hash the password of your users as you should. Now, one feature
of your SaaS is to connect to other people's database. They log into
your SaaS and store the username and password for a MySQL instance
that you have to connect and show all databases. The tricky part
to focus on is that this feature is not an authentication mechanism
that you are designing so you don't have to default to hashing
the database's password. In fact, if you hash it, you'll never
be able to recover the password and you'll never be able to actually
connect to it. In this case, you are the holder of someone else's
credentials. It's not your authentication mechanism therefore it's
not your job to hash this password. But you also don't want to store
it as plain text in your own database, so you choose to encrypt it.
If someone is looking directly at the database, they cannot know
the username and password for all the MySQL instances you stored.
You actually need the encryption algorithm and the key to decipher
it. When your software deciphers these values, you have the original
data back and now you can send it to the MySQL server that will
validate and authenticate them for you. This is to show you that
not because something is a password it means it should be hashed.

This is one of the reasons why the internet is full of integrations
via OAuth2 or something similar. Imagine if in order to integrate
with GitHub, you had to ask for people's GitHub password. It would
be a lot less safe and would have less adoption. Instead, GitHub
(and thousand of similar services) offer an integration mechanismm
where you create a Personal Access Token or an App. When your SaaS
wants to access a private repository from somebody else, you don't
need to know their password, you can use the Client Id & Client Secret
that their App has and GitHub will authenticate you through that.
You would encrypt **at least** the client secret when storing it
on your own database, but again you wouldn't be able to hash it and
still make it work.

#### MD5 ALL the passwords

NO, GOD! NO! PLEASE, NO!

We covered that when designing an authentication mechanism you should
hash your user's passwords. But MD5 is a hashing algorithm, right?
Yes it is, but it has been cracked a long time ago. Does this mean
you can reverse back things that have been hashed by MD5? Also, no.
I wasn't lying when I said that hashing algorithms are designed from
the ground up to be one-way. Once you hash a piece of information,
it never returns to it's original state. But an important property
of hashing algorithms is that they produce a fixed size hash.
MD5 outputs a hash of 32 characters. The characters used by md5 are
a to f and 0 to 9. That means 16 different possibilities for each
character. There exists "only" 32^16  (or 1,208,925,819,614,629,174,706,176).
This is an incredibly big number, but still finite. The fact that there
is absolutely no limit of what you can use as input of a hash algorithm,
but there is a limit of possible outputs, we're bound by math to have
**collisions**. After enough attempts, you will find 2 different
input values that result in the same hash. Remember how do we check
if a password is valid? That's right, hashing what was given to us
and checking if it matches the original hash we stored. If two distinct
values can produce the same hash, it's possible to access somebody's
account without actually knowing their password, you "just" have to
find a collision to their hash. Some smart folks that research
security for a living figured out a way to produce whatever hash
you want by manipulating what they input into MD5. That's why it's
considered a broken hashing function. 

Related to this, we have something called a Rainbow Table. They are
a list of hash with a known values that can produce said hash.
If you start hashing all letters of the alphabet, one by one,
all numbers one by one, you'll have 36 hashes. You can then
start hashing all possible combination of a string with 2 characters,
then 3 characters and so forth. You'd be building a Rainbow Table.
If you get a hold of somebody else's database full of hashed passwords
and you have a Rainbow Table for the algorithm they used, you can
start figuring out people's password. The problem is that the amount
of possible hashes is just too big and this becomes a very hard task
at scale, but still somewhat doable.
  
#### Hashing with Salt

When I first had contact with bcrypt for hashing passwords, part of
what I thought I knew about hashing collapsed for a minute: Every time
I hashed a password, it would produce a different hash. How am I
suppose to authenticate somebody with an algorithm like this?
If they give me their password during the login process and
the algorithm spits a different hash, I'll never be able to actually
validate their access. The problem was that I was only blindly
looking at the hash stored in the database without paying too
much attention to it. It is different every time, but it has
a specific structure: hashing + salt. What this essentially means
is that when hashing the password, the algorithm generates a random
string and uses it in combination with the user's password to generate
a unique hash. If two people have the same password, it still will
store 2 different hashes. This mechanism mitigates greatly the power
of Rainbow Tables. Think about it: you manage to get your hands on
a database full of username and passwords, but every password is hashed
with a salt. Let's say for the sake of argument 10 people had the
exact same password, but each of them have different hashes because
each of them had a different salt

| Password | Hash |
|---|---|
| 123456 | AAA |
| 123456 | BBB |
| 123456 | CCC |
| ... | ... |
| 123456 | III |
| 123456 | JJJ |

When generating your Rainbow Table, it's theoretically possible
that you can find a collision and figure out one of these passwords,
even though the chances are insanely small. Now the thing is, you
found only one password. That collision will not work for all other
9 people because they had different salt.

When I first learned about this I even found it counter-intuitive to
have the salt stored inside the hash, but after understanding it
I realized how simple and powerful this process is. It greatly
mitigates the power of Rainbow Table and collisions. When validating
a salted hash, the algorithm is fed with what the user typed
and the stored hash. The algorithm is then capable of combining
the actual password with the salt that was stored and check if 
the hash produced is valid. Two people with the exact same password
but different salt will therefore have different hashes and attacking
that hash is now harder. Guessing one password does not mean you guess
everybody's account with the same password. You have to start from
scratch. This is why it makes sense. An attacker with a database
full of hashed password will have the salt. But he will have to 
construct one Rainbow Table for each salt and go through the process
of testing billions of passwords to attack 1 single person and then
start all over again for the next person.

#### What's the deal with encryption at rest?

I may be wrong in many things and that is relatively fine, the 
basic knowledge still helps me shape and make some day to day
decision about software development. Encryption at rest is something
I have have just a very basic understanding as well. It usually
helps to protect data that might have been breached. For instance,
if your company uses AWS and store people's password, it's part of 
your responsibility to keep those password secure. But you
have no control over the datacenter AWS uses. What if someone
steals a hard drive that contains your entire database from their
physical location? Encryption at rest helps greatly here. The
entire database would be encrypted and it would be extremely hard
to decipher it to begin. If you did your due diligence in storing
safe passwords and your database is encrypted, it may take so many
years for the attacker to actually be able to decipher the entire
database and even if they succeeded at it, the passwords would
still be hashed and salted.

#### Conclusion

This game of catch that security plays is always about who is stronger.
There is no silver bullet for security and all we do is to try as
hard as we can to put as many layers of security as we can to make
it harder for the attacker. Do not confuse layers of security with
hashing twice or three times the password, that is counter-productive.
Use the algorithms the way they were designed and trust the smart
security folks that work to provide us with these algorithms. All
you need is basic understanding of when to use them and know how
to properly use them and your day to day job should be mostly fine.

Hope you enjoyed the reading. If you have any questions,
send them my way on [Twitter](https://twitter.com/deleugyn).

Cheers.