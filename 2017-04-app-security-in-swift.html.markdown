---
title: App Security in Swift
date: 2017-03-27 20:39 UTC
issue: april-2017
tags:
author: TBD
author_twitter: @elland
editor: Vixentael
editor_twitter: @vixentael
order: 2

---

# App Security

# TL;DR

- Security is a system, not a set of methods. It's better to think about threats and trust as early as you can.
- Avoid plaintext at all cost. Encrypt your communication. Encrypt your local data as required.
- If you don't store the passwors, it can't be leaked ;)
- But if you need to store password, do it right!
- It's not paranoia if it's true.

----

![Actual reality: nobody cares about his secrets. (Also, I would be hard-pressed to find that wrench for $5.)](https://imgs.xkcd.com/comics/security.png)

Keeping your user data safe and private is not the simplest of tasks. There is no golden solution, there's no sure proof way of preventing user data from being stolen. If someone has the means and really wants to, the system will be compromised.

> Two people are walking in the woods, they spot a bear. Security is not outrunning the bear. It's outrunning the other person.

With that bleak image in mind, let's see what we can do, to make it either as hard as possible or as hard as feasible for data to be breached.

Sure, security is hard, and there's no bullet proof solution. But there's a lot of small things we can sill do that greatly improve our chances of keeping user data safe.

## Take a step back

The first thing we should think about is if we really need to store that data. We’re often storing data we don't really need, out of habit or because of external pressures. Is it really important that we store the user's address? Their phone numbers? Their full names? What kind of genitalia they have? Sometimes that answer is yes, but most of the time, it's no. Remember: the more data you have, the more appetising your dataset becomes, and the more you have to work you have to secure it.

Once we figured out what data is really needed and what data can be discarded, the next step is figuring out risk and threat models. What are the most vulnerable points? What is more likely to happen? What can't be prevented?

There's no point in making a connection super secure, if the private key is kept locally in plaintext, in a file called `myprivatekey`.

> Security is always a trade-of, between convenience and safety, between the effort you can put in and the risk level you face.

### Start Early

There's a very simple reason why the iOS and the iPhone are so much more secure than Android. One of them was conceived with privacy and security in mind, the other wasn't. 

> It's much easier to build something with security in mind from the start, than to build something and then try to tack some security on it.

Sometimes making something that is already built secure might mean having to deconstruct the whole thing and put it back together again. Sometimes not even that can help.

### Never roll your own crypto

Cryptography is not easy. It's not simple to conceive it, it's not easy to implement it, and it's not easy to test it. There's a plethora of examples of really good cryptographic tools being broken all the time. Because computers become more powerful over time, because of bugs that weren't caught before. Tools will fail you, all you can do is try to make them have the least possible impact on your data.

![](2017-04-app-security-in-swift/schneier.png)

For instance, if you're using [ephemeral keys](https://en.wikipedia.org/wiki/Ephemeral_key) for your communications, even if a tool is cracked or a key is leaked, the damage will be minimal, compared to using the same key for everything. Think of using ephemeral keys as not re-using your passwords.

## Storing keys and tokens, embracing the Keychain

Plaintext is your enemy. Don't store your keys as a variable in code. Don't store it inside a local file. If something is sensitive, use the Keychain. Although it has a quite unfriendly API, there are [good wrappers](https://github.com/marketplacer/keychain-swift) out there.

```swift
// Don't do this:
let mySecretPassword: String = "Super secret password, tell nobody!"

// Try this instead:
let mySecretPassword: String = keychainWrapper.string(for: passKey)
```

It's a small step that already improves things by quite a lot.


> Do you want know more about key management? What goals and actions should key management system have, how to generate & store keys, should we obfuscate or encrypt keys, take a look on [@vixentael](https://twitter.com/vixentael)'s [last talk](https://speakerdeck.com/vixentael/keys-from-the-castle-ancient-art-of-managing-keys-and-trust).


Consider **pinning your SSL certs** and its public keys. There are many iOS libraries that will provide you with ways of doing SSL certificate pinning, but you probably want to pin the certificate's public key instead, since pinning the SSL certificate itself means that if they are revoked or expired, your app might not work until you update the binary with new certificates. As with most things security, you’ll have to weight the pros and cons of each solution. Sometimes it makes sense to give away a bit of extra security in the name of user convenience, sometimes it doesn’t. 

Just remember: if it’s too user-hostile, they won’t use it. The worst thing you can do is making your own users fight your system, instead of embracing it.

You’ll find more information on pinning certificates and public keys [in this great OWASP cheat sheet](https://www.owasp.org/index.php/Certificate_and_Public_Key_Pinning#iOS).

## Plaintext is the enemy

Be it passwords, API keys, your local database, your cache, your user personal details, one thing remains true: plaintext is your enemy. It doesn't matter how much care and effort you put into making your application secure, your server secure, your transport secure, if someone can just plug into iTunes and retrieve your certificates and keys, or even worse: lift them off from Github. Encryption is key.

![Not a pretty picture.](2017-04-app-security-in-swift/apikey.png)


## How to easily and safely encrypt/decrypt user data

There're many tools for symmetric and asymmetric encryption. Most of them are built on a top of existing community-proven libraries. We can name only few, but there're more, you know.

- [CryptoSwift](https://github.com/krzyzanowskim/CryptoSwift)
- [RNCrypto](https://github.com/RNCryptor/RNCryptor)
- [Themis](https://github.com/cossacklabs/themis)

For example, [Themis](https://github.com/cossacklabs/themis) provides a good context-based model for encrypting and decrypting data. A context-based model uses not only a password, but also a context, as a means of encrypting and decrypting data. We’ll be using the [Secure Cell](https://github.com/cossacklabs/themis/wiki/Secure-Cell-cryptosystem) model for this.

```swift
// Firstly, create a TSCellSeal using a password. Remember to keep this password safe! Embrace the Keychain.
let masterKeyData = password.data(using: .utf8)
guard let cellSeal: TSCellSeal = TSCellSeal(key: masterKeyData) else {
    print("Error occurred while initializing object cellSeal")
    
    return nil
}

// Then we can encrypt some data.
// Now the context can be anything, we recommend you pick something that creates a secure hard-to-guess association to the data being de/encrypted. For example: it's database row number.
let encryptedMessage: Data
do {
    encryptedMessage = try cell.wrap(message.data(using: .utf8),
                    context: context.data(using: .utf8))
} catch let error as NSError {
    print("Error occurred while encrypting \(error)", #function)

    return nil
}

// Now to decrypt it, we need the same password as well as the same context.
let decryptedMessage: Data
do {
  decryptedMessage = try cell.unwrapData(encryptedMessage, context: context.data(using: .utf8))
} catch let error as NSError {
  print("Error occurred while decrypting \(error)", #function)
 
  return nil
}
```

Again, a complex model made simpler with Themis, but still quite complicated. Using ephemeral keys, using key derivation, all behind the scenes, abstracting the complexity of security.

## SSL is not enough. Certificate Pinning is not enough.

You've been reading a lot about the topic, and you feel confident that your app is well configured. It will only accept HTTPS connections. It pinned all the certificates for the critical stuff. 

Unfortunately, SSL is a fragile system. (Remember, that [story about WoSign](https://security.googleblog.com/2016/10/distrusting-wosign-and-startcom.html), that issued certificates even for not domain owners?) Do not rely on SSL encryption if your app is dealing with *sensitive data*. 


- [Why you should avoid SSL in your next app?](https://www.cossacklabs.com/avoid-ssl-for-your-next-app.html)

Add additional layer of encryption for your network connection. Encrypt data before sending, decrypt after receiving. In the end-to-end world, server wouldn't even know what this data is.


## Conclusion
There are a simple set of tools and concepts that can go a long way with minimal effort. While not all applications deal with bank transactions, and therefore don't need the utmost security levels, most applications deal with a lot of user data, and that data should be kept just as safe.

Don't just give the illusion of security. Don't give the illusion of insecurity either. 

![](https://media.giphy.com/media/3o85xH5pfwcCXZyfg4/giphy.gif)


# Read more

- [Securing Communications on iOS](https://code.tutsplus.com/articles/securing-communications-on-ios--cms-28529)
- [Building a User-Centric Security Model in iOS Applications](https://realm.io/news/tryswift-anastasiia-voitova-building-user-centric-security-model-ios-applications-swift)
- [Upgrading Approaches to the Secure Mobile Architectures](https://medium.com/@vixentael/upgrading-approaches-to-the-secure-mobile-architectures-7a8fcb10d28a)
- [Engineering Privacy for Your Users](https://developer.apple.com/videos/play/wwdc2016/709/)