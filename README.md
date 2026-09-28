
# E2E Chat

<p align="center">
  <img src="./images/logo.png" width="300">
</p>

## Yleiskuva

Projektin alkuperäinen chat oli kahden käyttäjän välinen **E2E-salattu viestintäjärjestelmä**, jossa viestit salattiin asiakkaan puolella ennen niiden lähettämistä palvelimelle.

Ratkaisu perustui **static-key hybrid E2E encryption** -malliin:

* jokaisella käyttäjällä oli oma julkinen ja yksityinen avainpari
* käyttäjän julkinen avain (`publicKey`) voitiin tallentaa palvelimelle
* varsinainen yksityinen avain oli käyttäjän hallussa 
* tietokannassa voitiin säilyttää yksityisestä avaimesta salattu versio (`encryptedKey`)
* viestin sisältö salattiin AES:llä
* AES-avain suojattiin julkisen avaimen kryptografialla
* sama AES-avain suojattiin sekä vastaanottajalle että lähettäjälle
* avaimia ei vaihdettu tai ratchetoitu jokaisen viestin yhteydessä

Kyseessä oli karvalakki mallin E2E chät, yksinkertaisempi käyttäjäkohtaisiin pysyviin avaimiin perustuva E2E-ratkaisu.

---

# Architecture

Chat used WebSocket communication for realtime messaging and GraphQL for application-level data access.

```text
+-------------------+
|     Chat Client   |
|                   |
|  Private Key      |
|  Encryption       |
|  Decryption       |
+---------+---------+
          |
          | WebSocket
          |
          v
+-------------------+
|      Backend      |
|                   |
|  Chat / GraphQL   |
|  Message routing  |
|  Database access  |
+---------+---------+
          |
          v
+-------------------+
|     Database      |
|                   |
|  Public Keys      |
|  Encrypted Keys   |
|  Ciphertext       |
+-------------------+
```

Palvelimen tehtävänä oli käsitellä chatin tietoja ja välittää salattuja viestejä. Viestin varsinainen plaintext ei ollut palvelimen tarvitsema osa viestin välitystä.

---

# User Keys

Jokaisella käyttäjällä oli oma avainpari:

```text
+----------------------+
|      User Key Pair   |
+----------------------+
|                      |
|   Public Key         |
|      │               |
|      └── Database    |
|                      |
|   Private Key        |
|      │               |
|      └── User        |
|          Device      |
|                      |
+----------------------+
```

## Public key

`publicKey` oli käyttäjän julkinen avain.

Muut käyttäjät pystyivät käyttämään sitä salatakseen kyseiselle käyttäjälle tarkoitettua kryptografista materiaalia.

Julkinen avain voitiin siis säilyttää tietokannassa, eikä sen paljastuminen itsessään paljastanut käyttäjän private keytä.

## Private key

Private key oli käyttäjän salainen avain.

Sen varsinainen käyttö tapahtui käyttäjän omalla laitteella.

Tietokannassa oleva `encryptedKey` ei ollut sama asia kuin käyttäjän avoin private key. Se oli private keystä muodostettu **salattu avainmateriaali**, jonka tarkoituksena oli mahdollistaa private keyn turvallisempi säilytys ja palauttaminen käyttäjän tunnistautumiseen liittyvän salaisen tiedon avulla.

Palvelimelle ei siis ollut tarkoitus antaa käyttäjän salaista private keytä avoimessa muodossa.

---

# Password and Private Key

Käyttäjän salasana oli sidoksissa private keyn suojaamiseen.

Yksinkertaistettuna rakenne oli:

```text
User Password
      |
      v
Unlock / decrypt
      |
      v
Encrypted Private Key
      |
      v
Private Key
      |
      v
Decrypt Message Key
```

Tästä seurasi tärkeä ominaisuus.

Jos käyttäjä unohti salasanansa eikä vanhaa private keytä enää pystytty palauttamaan, käyttäjän täytyi luoda **uusi avainpari**.

Tällöin uusi private key ei pystynyt avaamaan vanhoja omalle käyttäjälle salattuja AES-avaimia.

---

# What Happens When the Password Is Lost?

Vanhoja viestejä ei välttämättä poistettu tietokannasta.

Ongelma oli kryptografinen:

```text
OLD KEY PAIR
     |
     +---- encryptedAesKeyOwn
     |
     X
     |
NEW KEY PAIR
     |
     +---- Cannot decrypt old own messages
```

Käyttäjä menetti siis pääsyn omiin vanhoihin viesteihinsä, jos niiden avaamiseen tarvittavaa vanhaa private keytä ei enää ollut saatavilla.

Keskustelukumppanin näkökulmasta tilanne oli erilainen.

Vastaanottajalle tarkoitettu AES-avain oli salattu hänen julkisella avaimellaan:

```text
                 Message
                    |
                    v
                 AES Key
                    |
          +---------+---------+
          |                   |
          v                   v
 Recipient Public Key     Sender Public Key
          |                   |
          v                   v
encryptedAesKeyFriend   encryptedAesKeyOwn
```

Siksi keskustelukumppani pystyi edelleen avaamaan hänelle tarkoitetun viestin omalla private keyllään, vaikka toinen käyttäjä olisi vaihtanut oman avainparinsa.

---

# Message Encryption

Varsinainen viesti salattiin symmetrisellä AES-salauksella.

Julkisen avaimen kryptografiaa ei käytetty varsinaisen viestin tekstin salaamiseen, vaan AES-avaimen suojaamiseen.

```text
Plaintext Message
       |
       | AES encryption
       |
       v
   Ciphertext
       |
       +-------------------+
       |                   |
       v                   v
    IV                AES Key
                           |
                    +------+------+
                    |             |
                    v             v
          Recipient Public    Sender Public
                 Key               Key
                    |             |
                    v             v
       encryptedAesKeyFriend  encryptedAesKeyOwn
```

Tämä on hybridisalauksen perusidea:

* AES soveltuu varsinaisen datan tehokkaaseen salaamiseen.
* Julkisen avaimen kryptografia soveltuu AES-avaimen suojaamiseen.

---

# Message Database Model

Alkuperäisessä `Message`-mallissa oli muun muassa seuraavat kentät:

```text
Message
├── id
├── ChatID
├── AccountID
├── Status
├── encrypted
├── encryptedAesKeyFriend
├── encryptedAesKeyOwn
├── iv
└── cipherText
```

## encrypted

Kertoi, oliko viesti salattu.

## encryptedAesKeyFriend

AES-avain salattuna vastaanottajan public keyllä.

Sen avulla vastaanottaja pystyi private keyllään palauttamaan viestin avaamiseen tarvittavan AES-avaimen.

## encryptedAesKeyOwn

Sama AES-avain salattuna lähettäjän public keyllä.

Tämän tarkoituksena oli mahdollistaa myös lähettäjän pääsy omaan lähettämäänsä viestiin.

Ilman tätä rakennetta lähettäjä ei välttämättä pystyisi avaamaan omaa viestiään myöhemmin, koska viesti olisi salattu ainoastaan vastaanottajan avaimella.

## iv

AES-salauksen yhteydessä käytetty initialization vector.

## cipherText

Varsinainen salattu viestisisältö.

Tietokannassa ei siis ollut plaintext-viestin sisältöä tässä rakenteessa.

---

# UserKey Database Model

Alkuperäisessä `UserKey`-mallissa oli:

```text
UserKey
├── AccountID
├── encryptedKey
├── publicKey
├── createdAt
└── updatedAt
```

Oleellinen jako oli:

```text
publicKey
    |
    +---- Public information
    |
    +---- Stored in database


encryptedKey
    |
    +---- Encrypted private-key material
    |
    +---- Stored in database


Private Key
    |
    +---- Secret key
    |
    +---- Used by the user's device
```

`encryptedKey` ei siis tarkoittanut, että tietokannassa olisi ollut käyttäjän private key avoimena.

---

# Message Decryption

Vastaanottajan laitteella prosessi tapahtui konseptuaalisesti näin:

```text
Database
   |
   +---- cipherText
   |
   +---- iv
   |
   +---- encryptedAesKeyFriend
              |
              v
        User Private Key
              |
              v
           AES Key
              |
              v
      AES Decryption
              |
              v
        Plaintext Message
```

Laitteen piti siis saada käyttöönsä vastaanottajan private key.

Palvelin pystyi säilyttämään ja välittämään salattua viestidataa ilman että se tarvitsi käyttäjän avointa private keytä viestin avaamiseen.

---

# Static-Key Architecture

Projektin kannalta tärkeä termi on **Static-Key Hybrid E2E Encryption**.

"Static" tarkoittaa tässä sitä, että käyttäjän avainpari oli käyttäjäkohtainen ja pysyvämpi avainidentiteetti.

Järjestelmässä ei ollut:

* per-message key rotation -mekanismia
* ratcheting-mekanismia
* Signal Double Ratchet -tyyppistä avainketjua
* automaattisesti vaihtuvia käyttäjäkohtaisia avainpareja jokaiselle viestille

Mallia voi kuvata näin:

```text
USER
 |
 +---- Persistent Public Key
 |
 +---- Persistent Private Key
              |
              v
        Message Decryption


MESSAGE
 |
 +---- AES Encryption
 |
 +---- AES Key protected with recipient public key
 |
 +---- AES Key protected with sender public key
```

Tämä oli huomattavasti yksinkertaisempi kuin nykyiset ratcheting-pohjaiset E2E-protokollat, mutta se mahdollisti kahden käyttäjän välisen salatun viestinnän ilman että plaintext-viestiä tarvittiin tallentaa palvelimelle.

---

# Key Replacement

Kun käyttäjä joutui vaihtamaan avainparinsa:

```text
OLD KEY PAIR
     |
     +---- Old messages
     |
     +---- Old encrypted AES keys
     |
     X
     |
NEW KEY PAIR
     |
     +---- New messages
```

Uusi avainpari oli kryptografisesti uusi identiteetti.

Siksi vanhojen viestien avaaminen ei automaattisesti siirtynyt uudelle avainparille.

Tämä oli suora seuraus siitä, että viestien avaamiseen tarvittava avainmateriaali oli sidottu vanhaan private keyhin.

Keskustelukumppanin omat avaimet eivät kuitenkaan muuttuneet tämän seurauksena, joten hän pystyi edelleen lukemaan omat vastaanottamansa vanhat viestinsä.

---

# Security Boundary

Projektin alkuperäinen turvallisuusmalli voidaan tiivistää näin:

```text
+---------------------+
|      User Device    |
|                     |
|  Private Key        |
|  Plaintext          |
|  Encryption         |
|  Decryption         |
+----------+----------+
           |
           | Encrypted data
           v
+---------------------+
|       Server        |
|                     |
|  Public Keys        |
|  Encrypted Keys     |
|  Ciphertext         |
|  IV                  |
+---------------------+
```

Palvelimen ei ollut tarkoitus toimia käyttäjän private keynä tai tietää käyttäjän plaintext-viestejä.

Suojaus perustui siihen, että käyttäjän salainen avainmateriaali pysyi käyttäjän hallinnassa ja tietokantaan tallennettu viestidata oli salattua.

---

# Limitations

Alkuperäinen toteutus oli tarkoituksella yksinkertaisempi kuin modernit E2E-protokollat.

Keskeiset rajoitteet olivat:

1. Käyttäjäkohtainen avainpari oli staattinen.
2. Viestikohtaista ratcheting-mekanismia ei ollut.
3. Avainparin vaihtaminen saattoi katkaista käyttäjän pääsyn vanhoihin omiin viesteihin.
4. Salasanan unohtaminen saattoi johtaa vanhan private keyn menettämiseen.
5. Järjestelmä oli suunniteltu erityisesti kahden käyttäjän väliseen chat-käyttöön.
6. Malli ei vastannut modernien Signal-tyyppisten E2E-protokollien rakennetta.

Tämä ei kuitenkaan tarkoita, että alkuperäinen idea olisi ollut "ei-salattu" tai pelkkä palvelimen toteuttama salaustekniikka. Kyseessä oli asiakkaan ja käyttäjäkohtaisten avainten ympärille rakennettu hybridimuotoinen E2E-malli.

---

# Terminology

Projektista voidaan käyttää seuraavia termejä:

**Static-Key Hybrid E2E Encryption**

tai suomeksi:

**Staattisiin käyttäjäkohtaisiin avaimiin perustuva hybridimuotoinen E2E-salaus**

Tärkeimmät käsitteet:

| Termi                   | Merkitys                                                   |
| ----------------------- | ---------------------------------------------------------- |
| `publicKey`             | Käyttäjän julkinen avain                                   |
| Private Key             | Käyttäjän salainen yksityinen avain                        |
| `encryptedKey`          | Salattu private-key -materiaali                            |
| AES Key                 | Varsinaisen viestin salaamiseen käytetty symmetrinen avain |
| `encryptedAesKeyFriend` | AES-avain salattuna vastaanottajalle                       |
| `encryptedAesKeyOwn`    | AES-avain salattuna lähettäjälle                           |
| `iv`                    | AES-salauksen initialization vector                        |
| `cipherText`            | Salattu viestisisältö                                      |
| Static-Key              | Käyttäjäkohtainen pysyvämpi avainpari                      |
| Hybrid Encryption       | AES + public-key cryptography                              |

---

# Summary

Alkuperäinen E2E-chat perustui seuraavaan ideaan:

```text
User
 |
 +---- Public Key --------------------+
 |                                    |
 +---- Private Key                    |
 |                                    |
 +---- Password                       |
                                      |
                                      v
                              Message Encryption
                                      |
                    +-----------------+-----------------+
                    |                                   |
                    v                                   v
              Recipient Public Key                 Own Public Key
                    |                                   |
                    v                                   v
          encryptedAesKeyFriend                 encryptedAesKeyOwn
                    |                                   |
                    +-----------------+-----------------+
                                      |
                                      v
                                  Ciphertext
                                      |
                                      v
                                  Database
```

Järjestelmän keskeinen idea oli, että **itse viesti salattiin AES:llä ja AES-avain suojattiin käyttäjien public keyillä**.

Jokaisella käyttäjällä oli oma pysyvämpi avainpari. Private key oli käyttäjän salainen avainmateriaali, kun taas `publicKey` voitiin julkaista ja tallentaa palvelimelle. Tietokannassa oleva `encryptedKey` oli salattu private-key -materiaali, ei avoin private key.

Koska avainpari oli staattinen eikä järjestelmä käyttänyt viestikohtaista ratchetingia, avainparin vaihtaminen vaikutti myös vanhojen omien viestien avaamiseen. Keskustelukumppanin pääsy omiin vanhoihin viesteihinsä säilyi kuitenkin hänen oman avainparinsa ansiosta.

**Projektin alkuperäinen kryptografinen malli:**

> **Static-Key Hybrid E2E Encryption**
>
> AES for message encryption + public-key cryptography for protecting the AES key + persistent user key pairs.
