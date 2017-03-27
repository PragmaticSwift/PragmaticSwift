---
title: App Security in Swift
date: 2017-03-27 20:39 UTC
issue: april-2017
tags:
author: TBD
author_twitter: TBD
editor: TBD
editor_twitter: TBD
order: 2
---


# Vitae eadem Solem

## Rostro Iuppiter sine

Lorem markdownum et pete, nec curru mente voce et poscit. Maduere pars: non
nomina quod. Illa malae meliore: in festum Achillis Tritonida tenere cornua
Coeo.

- Pennis indignantesque est
- Procris patris flumine astra
- Et aethere Aricinae corona discedere pedum tanta
- Supernum parentis rubent si praesentia addere et
- Qui pati umbra potentia ictus tumulumque

Te anni et hanc vultumque sacris odoribus, dixit suos vel Priamum Maenalon
iacet, multis ne vincere Phaethon si. Ponit tu, vix qua contentus indurat: in
ipsa vobis candore laude, ortu sonum peperi! Domo decoram, equo ait, femina!

## Lepores ponti ubi

[Ululare](http://nubibus.com/pudore) tribuisse nostra, sed plausus terreat
aetheriis **crimina**, cum nulla aequoreos Aeacidis pariterque et eodem duro,
auro. Quae caelestibus quae. Sub sentite lora iam
[Stygiis](http://www.in.net/moenia) parte; causa orbes eras nutat. Loca titulum
ursa multaque ambit vel quae sine, revocabile orbem.

> Fortuna semina. Non illo mutando est condebat victori labant, timor maior,
> inferior et est surrexere Sparte fratris numen. Canitie pater. Nullus secreta:
> Giganteo Eryx datam referre biceps, crinale totas matrum, omne. Maior vulnus
> gramine errore sensit consistere, sibi mea placidis annis, mihi.

Captiva desiluit obscurum, si, manibus nec potius huc voti Petraeum hic. Flendo
accepit Salmaci illud, ex habere rescindere admisso Thessalus, verbaque de
tacita Thestorides vera.

```swift
import Foundation

struct Foo {
  let bar: String
  let goo: Float
}

func main() {
  let foo = "Hello"
  var bar = "bar"
  bar = "World"
  print("\(foo) World")
}
```

![Preface](2017-04-app-security-in-swift/dummy.gif)

Cum in fuerant puer. Crescit deus. Non nunc postquam teneris [quoque
parari](http://www.donec-nec.io/sicpassa).

Respersit iaceres erat vultusque ad crimine Plena, sermone! Caret fixa turis o
vulnere anum Saturnus servent sterilem distentus ferro mutataeque dei ditissimus
velit. Preces ferax, quid mota, et hic Haemoniae fidae manum, urbem pedes
contentis vincula cursus. Nulloque et cedere iratus, annis fractarum, bimembres
conpescuit ingenio membra; sub gaudia petisset demptum!


---
### ORIGINAL
---

# Todo:
- Maybe limit the scope? Just ephemeral key rotation, storing your data locally and trusted sessions using Themis, as a tutorial, with some links to more stuff, like Anastasiia's talks and slides.
- More pictures
    - ![](http://newmediarockstars.com/wp-content/uploads/2013/01/CatHacker.jpg)
- More source code examples
    - [Example](https://github.com/cossacklabs/theswiftalpsdemo/blob/master/ios-project/SwiftAlpsSecDemo/SwiftAlpsSecDemo/CellDemo.swift)
- Toolset recommendation
    - [Themis](https://github.com/cossacklabs/themis)
    - [Crypto](https://github.com/soffes/Crypto)
- User authentication flows
    - One-time passwords
    - Tokens
    - Storing session data
- Server trust flow
    - [Example](https://github.com/cossacklabs/theswiftalpsdemo/blob/master/ios-project/SwiftAlpsSecDemo/SwiftAlpsSecDemo/SessionDemo.swift)

#### Structure:

##### Intro

Keeping your user data safe is no easy task. There is no golden solution, there's no sure proof way of preventing user data from being stolen. If someone has the means and really wants to, the system will be compromised.

> Two people are walking in the woods, they spot a bear. Security is not outrunning the bear. It's outrunning the other person.

With that bleak image in mind, let's see what we can do, to make it either as hard as possible or as hard as feasible for data to be breached.

Sure security is hard, and there's no bullet proof solution. But  there's a lot of small things we can sill do that greatly improve our chances of keeping user data safe.

##### Storing keys and tokens, embracing the Keychain

Plaintext is your enemy. Don't store your keys as a variable in code. Don't store it inside a local file. If something is sensitve, use the Keychain. Although it has a quite unfriendly API, there are very [good wrappers](https://github.com/marketplacer/keychain-swift) out there.

```swift
// Don't do this:
let mySecretPassword: String = "Super secret password, tell nobody!"

// Try this instead:
let mySecretPassword: String = keychainWrapper.string(for: passKey)
```

It's a small step that already improves things by quite a lot.

Consider pinning your SSL certs and its public keys. Althoug there are many iOS libraries that will provide you with ways of doing SSL certificate pinning, you probably want to pin the certificate's public key instead, since pinning the SSL certificate itself means that if they are revoked or expired, your app might not work until you update the binary with the new certificates.

For more information on pinning, go [here](https://www.owasp.org/index.php/Certificate_and_Public_Key_Pinning#iOS).

##### How to easily and safely encrypt/decrypt user data

Themis provides a good context-based model for encrypting and decrypting data. A context-based model uses not only a password, but also a context, as a means of encryting and decrypting data. 

```swift
// Firstly, create a TSCellSeal using a password.
let masterKeyData = password.data(using: .utf8)
guard let cellSeal: TSCellSeal = TSCellSeal(key: masterKeyData) else {
    print("Error occurred while initializing object cellSeal")
    
    return nil
}

// Then we can quickly encrypt it. Behind the scenes, themis will ... what will it do behind the scenes?!
let encryptedMessage: Data
do {
    encryptedMessage = try cell.wrap(message.data(using: .utf8),
                    context: context.data(using: .utf8))
} catch let error as NSError {
    print("Error occurred while encrypting \(error)", #function)

    return nil
}

// Now to decrypt it, we need the same password as well as the same context
let decryptedMessage: Data
do {
  decryptedMessage = try cell.unwrapData(encryptedMessage, context: context.data(using: .utf8))
} catch let error as NSError {
  print("Error occurred while decrypting \(error)", #function)
 
  return nil
}
```


##### How to create trusted server connections and keep your transport layer secured

Again, a complex model made simpler with Themis, but still quite complicated. Using ephemeral keys, using key derivation, all behind the scenes, abstracting the complexity of security.

##### Some thoughts

Don't just give the illusion of security. Don't give the illusion of insecurity either. 

![](https://media.giphy.com/media/3o85xH5pfwcCXZyfg4/giphy.gif)

##### Conclusion
Wrap up, give a nice tl;dr. References.


# App Security

[![Crytpography](http://imgs.xkcd.com/comics/cryptography.png
)](https://xkcd.com/153/)

Keeping your user data safe is no easy task. There is no golden solution, there's no sure proof way of preventing user data from being stolen. If someone has the means and really wants to, the system will be compromised.

> Two people are walking in the woods, they spot a bear. Security is not outrunning the bear. It's outrunning the other person.

With that bleak image in mind, let's see what we can do, to make it either as hard as possible or as hard as feasible for data to be breached.

## Analyse.

The first thing you have to think about is if you do really need to store that data. A lot of the time, you're storing data that you don't really need. Do you really need that user's address? Their phone numbers? Their full names? Sometimes that answer is yes, but most of the time, it's no. Just remember that the more data you store, the more apetising your dataset becomes.

Once you figure out what data you need and what data you can discard, the next step will be figuring out your risk and threat models. What are the most vulnerable points? What is more likely to happen? What can't be prevented?

[![Security](http://imgs.xkcd.com/comics/security.png)](https://xkcd.com/538/)

There's no point in making your connection super secure, if you're private key is kept locally in plaintext, in a file called `myprivatekey.cert`.

> Security is always a trade-of. Between convenience and safety. Between the effort you can put in and the risk level you face.

## What can we do.

There are a simple set of tools and concepts that can go a long way with minimal effort. While not all applications deal in bank transactions, and therefore don't need the utmost security levels, most applications deal with a lot of user data, and that data should be ketp safe.

### Start Early.

There's a very simple reason why the iOS and the iPhone are so much more secure than Android. One of them was conceived with privacy and security in mind, the other wasn't. It's much easier to build something with security in mind from the start, than to build something and then try to tack some security on it. Sometimes making something that is already built secure might mean having to deconstruct the whole thing and put it back together again. Sometimes not even that can help.

### Never roll your own crypto.

Cryptography is not easy. It's not simple to conceive it, it's not easy to implement it, and it's not easy to test it. There's a plethora of examples of really good crytpographic tools being broken over time. Be it become computers become that much more powerful over time, be it because of bugs there weren't caught before. Tools will fail you, all you can do is try to make them have the least possible impact on your data.

For instance, if you're using ephemeral keys for your comunications, even if a tool is cracked or a key is leaked, the damage will be minimal, compared to using the same key for everything. Think of using ephemeral keys as not re-using your passwords.

### SSL is not enough. Certificate Pinning is not enough.

You've been reading a lot about the topic, and you feel confident that your app is well configure. It will only accept HTTPS connections. It pinned all the certificates for the critical stuff. But your traffic is still being sent as plain text. If someone manages to break through your SSL connection, all your user data is now visible.

### Plaintext is the enemy.

Be it passwords, API keys, your local database, your cache, your user personal details, one thing remains true: plaintext is your enemy. It doesn't matter how much care and effort you put into making your application secure, your server secure, your transport secure, if someone can just plug into iTunes and retrieve your certificates and keys, or even worse: lift them off from Github. Encryption is key.

![Not a pretty picture.](2017-04-app-security-in-swift/apikey.png)

### Example code, example app.

# TL;DR

- Trust no one, but the user.
- Don't trust the user is really is who they say they are too easily. Consider multi-factor authentications. Consider passphrases.
- Avoid plaintext at all cost. If you're not manually encrypting your API calls, they're virtually still plaintext, just camouflaged.
- The Keychain is your friend. Love the Keychain.
- Don't rely on SSL. Encrypt your communication. Encrypt your local data as required.
- Use ephemeral keys. Use key derivation methods.
- Store as litte data as possible.
- It's not paranoia if it's true.

> The best knowledge is zero-knowledge.