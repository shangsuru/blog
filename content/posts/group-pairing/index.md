---
title: "Group Pairing Schemes"
description: "The problem of establishing a shared key for group communication in an ad-hoc mobile network."
tags: [network security]
date: 2021-07-13
math: true
---
Cryptographic protocols used for secure message exchange most of the time already assume shared key material. It is a challenging problem to ensure the authenticity of initially exchanged public
keys in the first place and is conceptually approached either by building a chain of trust, i.e., having the own key signed by another trusted certificate authority, or by a form of out-of-band communication.

The goal of group pairing schemes is to extend the exchange of key material from two people to a larger set of users while not depending on a trusted third party or PKI. They are often targeted for implementation on smartphones or IoT devices.

Use cases could be to exchange contact information during a meet up or conference, but also in the context of smart home devices or wireless sensor networks. Because here we have a similar scenario that different entities want to communicate securely in an ad-hoc manner and setting up prior shared secrets is not feasible.

In this blog post, I give an overview of existing approaches, categorize them into two distinct groups, keyless schemes and secret key based schemes, and weigh them against each other in an attempt to expose the most promising solution.

# Background 

Most group pairing schemes are inspired from ideas in pairwise pairing schemes. So we will have a short look at them. 
It should be noted though that it is not possible to use pairwise schemes as direct building blocks for group pairing schemes by executing them between group members, as this would not scale well, when a separate key has to be established between each pair of users in a group.

## Diffie-Hellman Key Exchange

The first proposed solution for a key exchange protocol over an insecure channel and the most well-known is probably the Diffie-Hellman Key Exchange. 

![Diffie-Hellman Key Exchange](posts/group-pairing/images/1.png)

1. In the protocol, a multiplicative group of integers modulo $p$ is used, where $p$ is a prime number and a base $g$, which the two parties, for simplicity we name them Alice and Bob, agree on and don't need to be kept secret.
2. Then Alice chooses a secret integer $a$, while 
3. Bob does the same and chooses a secret integer $b$. 
4. Alice sends $g^a mod$ $p$ to Bob, 
5. Bob sends $g^b mod$ $p$ to Alice respectively. 
6. Both can now compute a shared secret $g^{ab} = (g^{a})^{b} = (g^{b})^{a}$. 

An attacker needs to solve the Computational Diffie-Hellman Problem (given $g^a$ and $g^b$, compute $g^{ab}$ or solve the Discrete Logarithm problem (given $g^a$ and $g$, compute $a$), which are both computational hardness assumptions.

Unfortunately, this protocol suffers from the infamous Man-in-the-Middle attack.

Due to the lack of missing authentication, an attacker, Eve, can put herself in between the two communication partners and intercept messages. Several approaches were developed to counter this, among others the so called Station-to-Station protocol [2], which needs a PKI in place however to provide authentication of the participants, which is what we want to avoid.

![Key Agreement Tree](posts/group-pairing/images/2.png)

The Diffie-Hellman Key Exchange protocol can also be extended to groups. Each participant resembles the leaf in a binary tree. Each node contains a public and private key of the participant. The parent node is constructed by computing the private key as the key that would result from an ordinary Diffie-Hellman key exchange between the two child nodes. For example, if the private-public key pairs of the two childs are 
$<x, g^x>$ and $<y, g^y>$, then the parent node becomes $<g^{xy}, g^{g^{xy}}>$. 
The resulting root node is the group key, which can be derived by all participants.

## Talking-to-Strangers and Seeing-is-Believing

The idea behind Talking-to-Strangers [3] is the following: Two parties can exchange public keys by utilizing an out-of-band channel to send each other the hash of their respective public keys as a commitment. Now the real public keys can be transferred over the insecure wireless channel and subsequently verified via the earlier received commitments. Seeing-is-Believing [4] uses the visual channel as an out-of-bands channel. More specifically, it requires smartphones with displays and a camera. One smartphone displays a bar code that encodes the commitment of its public key and another smartphone can scan this bar code to retrieve it.

## Don't Bump, Shake on It

![Don't Bump, Shake on It](posts/group-pairing/images/3.png)

"Don't bump, shake on it" [5] is a great example for a pairwise pairing protocol that makes use of shared sensor measurements between two devices and that derives a common key through the shared entropy created by those measurements. In this case it is a combination of vibrator's and accelerator's measurements when laying two devices on top of each other and shaking them in a random pattern that can create this shared source of randomness, from which a key can be derived.

# Design of Group Pairing Schemes

In this section, I enumerate the factors that make up a good group pairing scheme and outline the capabilities of the attacker, to lay the foundations of the design of group pairing schemes. After that, concrete existing solutions are described. Hereby, I divide into two types of protocols, "keyless" and "secret key based" schemes. 

## Requirements

The following are a compilation of explicitly and implicitly stated requirements from the papers presented in later sections. 


- **Scalability**: Most important for a group pairing scheme is good scalability. That means the communication overhead, or the communication rounds, should not grow linearly or worse with group size.
- **Fault Tolerance**: It should be tolerant to fault, not taking into consideration denial of service attacks, but human errors during protocol execution. There should be countermeasures in place that work against a careless or impatient user skipping certain actions, where human involvement is needed to verify a prior step of the protocol, like making sure that the correct data was exchanged.
- **Close to Zero Interaction**: We want to limit human involvement as much as possible, since human error affects security. A group pairing scheme that requires too much user interaction is not only insecure, but will also discourage users to use the system regularly or even consider using it at all. The best case would be if the system was autonomous, with zero user interaction. 
- **Speed of Execution**: The protocol should execute fast. Speed can depend on the level of necessary user interaction, the underlying communication technology, the way a shared secret is generated or how the exchanged data is verified.
- **Accessibility**: The schemes should only require hardware that is found in current off-the-shelf-smartphone devices. This accommodates also for the scenario, where different IoT devices want to participate in a group pairing scheme, as those will also have very heterogeneous hardware.
- **Infrastructure-less**: Another feature is whether the scheme is completely independent of infrastructure or not, since some approaches still rely on a server, although the server itself is considered untrusted.

Lastly, **security**. A huge challenge, especially in group pairing schemes, is to ensure the authenticity of the exchanged data. Security in the context of group pairing schemes can be defined through the following three properties, as defined in [6]:

1. **Consistency**: After the data exchange is finished, all group members should have gained the same information. It should not be the case that a participant did not get some part of the data, or the data of a user was modified.
2. **Exclusivity**: The exchanged information should only include the information of the intended group members. An outsider should not be able to secretly participate in the protocol and add his data, potentially by impersonating another participant. 
3. **Uniqueness**: No group member should be able to contribute information twice. If that would not be the case, a legitimate participant could introduce additional false information.

## Attacker model

As for the capabilities of the attacker, an attacker according to the Dolev-Yao threat model is assumed. He can eavesdrop on the ongoing communication and modify, remove, replay or inject messages. He can impersonate honest users and make other users believe that they are communicating with those honest users, but in reality they share a key with the attacker. However, the attacker cannot block or disable the channel. He cannot perform a denial of service attack or indefinitely jam a user's communication. It should be noted that the attackers capabilities do not extend to the separate out-of-band channel that is used in many protocols as a mean to exchange information shielded from an attacker's influence. Namely, an out-of-band channel can ensure authenticity, integrity and potentially also confidentiality of the messages sent, by being completely out of reach or unmodifiable for the attacker.

In the setting of group pairing schemes, the attacker might attempt to execute the following types of attacks: He can add fake members or remove valid members and potentially impersonate them. 

This can lead to a Group-in-the-Middle attack, where the attacker effectively divided the group into two subsets, each containing some number of attacker controlled participants that emulate the environment a member would be in, when executing the scheme within the intended group of trusted users.

The attacker can also just secretly add himself to the group as a malicious bystander or perform a so called Sybil attack, where the attacker introduces multiple made-up participants to the protocol. Therefore, it is almost always necessary for the participants to agree on a group size beforehand and verify the correct group size afterwards to counter this attack.

# Keyless security

For this type of protocol, the participants have no shared keys to base their initial trust on. Inevitably, because there is no shared secret to verify the legitimacy of the other participants, these protocols have to rely on human aid to verify the exchanged data. They commonly share the following 3 phases:

1. **Collection**, where the user selects and prepares the data he wants to share,
2. **Distribution**, where the data is exchanged between the participants, often making use of commitments,
3. **Verification**, where the exchanged information has to be verified. This is done by computing a hash over the exchanged information and displaying that hash in a human friendly way for comparison. 

## The SafeSlinger Protocol

SafeSlinger [7] is a system that allows a group of users to exchange public keys for secure communication.
The corresponding paper shows a practical solution to the problem by designing and implementing an easy to use smartphone app that enables a group of two or even more users to securely and privately exchange public keys and other contact information. This works without any external trusted parties and can either happen in person or remotely e.g., via a telephone or video conferencing line. The authors claim that it is the first complete system that provides this functionality. By the means of a user study, they showed that their app provides much better usability than comparable existing solutions.

In this section I want to summarize the workings of the underlying protocol of SafeSlinger in conjunction with how the user would interact with the app's interface.

### Collection

As a first step the participants of the key exchange gather either in person or meet online. It is important that in this setting the users can authenticate the intended group members via, e.g., appearance or voice in order to be able to exclude unwanted participants. 
Each user chooses the data to share and needs to select the total number of participants. This prevents an attacker to simply join the exchange as the number of participants would increase.

### Distribution (via multi commitment generation)

Then the user sends two nonces resembling protocol success or abortion, together with the personal data encrypted with the nonce for success, and a public key, which will be used at the end of the protocol  to share the nonce of success for decryption of the personal data, as a multi value hierarchical commitment to a server.

The idea of commitments is a cryptographic technique used in numerous group pairing schemes. By sending the hash of a value $V$, $H(V)$, first, you can later verify that the correct value was sent by comparing its hash with the previously sent commitment $H(V)$. A multi-value commitment structure is a tree of several values, where the leaves resemble the initial values, and parent nodes are computed as the hash of their children. This allows to gradually reveal the values, while not all children nodes have to be revealed together. E.g., consider the simple example of two values $V_1$ and $V_2$, and their respective commitments $H(V_1)$, $H(V_2)$. The root node of the commitment tree would then be $H( H(V_1) || H(V_2))$. To reveal $V_1$, one can make $V_1$ and $H(V_2)$ public, without leaking information about $V_2$. The commitment cannot be faked easily, because an attacker would need to find a collision for values he does not even know.

Now the server needs to group together the users that want to share key material among each other. Therefore it assigns each user a unique ID. In their app, the users are supposed to enter the smallest ID within their group, which enables the server to pair them together. The server is now able to distribute the IDs and commitments to all group members.

![SafeSlinger](posts/group-pairing/images/5.png)

### Verification (via Three Word Phrases)
To verify the authenticity of the exchanged data, the first level of each commitment is now opened, revealing the encrypted user information, the public key and another hierarchical multi value commitment of the two nonces. The users can now compute a common hash over all decommitments ordered by ID. This hash should be the same for everybody, else this would indicate that a malicious attacker has injected wrong information about another user (impersonation attack) or a fictitious person (Sybil attack), or conducted an entire Group-in-the-Middle attack.
To make the comparison easy, instead of the hash itself, a three word phrase derived from that hash using the PGP Word List is presented to the user along with two decoy phrases. The user has to select the right one that all the group members share. The decoys serve the purpose to prevent careless users from continuing without comparing the word phrase.

The nonce resembling abortion of the protocol would now be revealed in case the user selected the wrong word phrase or because there is no matching phrase due to the aforementioned attacks. The previous commitment of the nonces prevents an attacker to inject abort messages. Otherwise, the single level multi commitment of the success nonce is revealed. Now all users can verify that everyone has selected the correct phrase and if so, have successfully exchanged their public key pairs.

### Secret Sharing Round

The participants can use their exchanged public keys in a Group Diffie-Hellman Key Agreement protocol to derive a common group key, which is used to send the final success nonce, which can be  verified by the commitment revealed previously. Using the success nonce, every user is now able to decrypt the personal information of all other participants, completing the protocol.

### Similar protocols

Other protocols in this category mainly differ in the distribution phase, where different topologies like a ring or tree topology is used, which involves more manual steps, and in the verification phase, where a visual representation of the hash is used instead of the three word phrases, which are easier to compare, see GAnGS [6] and Spate [8]. But SafeSlinger has the better way of avoiding impatient users to compromise the security of the protocol.

## Chorus

Chorus [9] is not a full solution for group pairing, but offers an alternative to the verification phase, which was done via an out-of-band channel in the previous presented solutions. It presents a novel physical layer primitive, called Chorus, that is enabling authenticated fixed-length string comparison, held by multiple devices, within constant time over an insecure wireless channel. It does not require any prior shared secret. Unlike Chorus, other secure device pairing schemes of that kind require the security of an out-of-band channel, like the visual or audio channel. These schemes, however, usually have higher demands on human involvement and have additional hardware requirements that a smartphone might easily fulfill, but not the typical resource constrained IoT device.

The principle behind Chorus is the following: At the end of some information exchange protocol, participants would like to make sure that they have exchanged and received the same information, just like in the verification phase of the previous protocols. This resembles basically the authentication of a string of the protocol transcript. Chorus makes use of an encoding scheme that makes it possible that any differences of the strings of different participants are immediately noticed. 

![Chorus](posts/group-pairing/images/6.png)

In a wireless channel, an attacker can only manipulate bits by flipping "0" to "1", not vice versa. The core idea is to let every node broadcast their string simultaneously, hence the name "Chorus", making use of an error detection code.  The coordinator starts with a synchronization packet detected by the other nodes via threshold energy detection. Each node encodes its string using Manchester encoding, "0" becomes "01" and "1" becomes "10", and maps each 1/0 bit to an ON/OFF state respectively. Assuming the string is the same for every device, during the ON state every device should transmit random bits, and during the OFF state they should keep silent. Due to the choice of Manchester encoding, any bit flip can be recognized by all parties, when they expect everyone to be silent at an OFF state, but a device still transmits. Because the transmitted signal is unpredictable, an attacker cannot cancel out the signal and self-cancellation is very unlikely.  

# Secret key based security

The following schemes are based on password-authenticated group key exchanges. This type of protocol does not have to deal with the aforementioned attacks, because participants can be verified by the knowledge of the secret password and therefore less user involvement is necessary. The big disadvantage is the password itself: Passwords are known to be a huge problem in authentication scenarios. It is difficult to safely share them among a larger number of participants, and even if so, then they are often not random enough and can be guessed easily by an adversary.

A solution, key extraction, explored in the paper about "Password-Authenticated Group Key Exchange" [10] is to execute a Group Password-Authenticated Key Exchange protocol on the higher layers, but generate the underlying password on the physical layer via a shared source of randomness not accessible by the attacker, i.e., an out-of-band channel that can be shared properties of the wireless channel, audio, visual signals, vibrations or even the entropy created by shaking the devices together.

Key extraction procedures usually require 4 phases: The channel probing phase, where the random features of the channel are harvested through the exchange of probe packets. The probing rate should be as fast as possible, but also efficient by not probing too fast, when there was no change in the channel properties in the meantime. Then the quantization phase, where the analog data is converted into bits. The quantization function used should preserve the randomness and should not produce a long rows of 1s for example. A following information reconciliation phase is necessary, because the keys between devices might still differ after the quantization phase. To overcome this discrepancy, some information like the parity bits is exchanged, which all leak minimal information. Therefore, a privacy amplification phase follows, so that an eavesdropper cannot deduct the key from  leaked information during the information reconciliation phase.

## Password-Authenticated Group Key Exchange: A Cross-Layer Design

The paper suggests extracting common key bits using the wireless channel. Specifically, they argue that the fading of the channel is invariant between two communicating parties but decorrelates when not being within a distance of half the wavelength. So assuming the attacker is not in the proximity of the communicating pair, it is impossible for him to gain insights into the generated key bits. 

![Password Extraction](posts/group-pairing/images/7.png)

The users form a logical ring and perform the password extraction algorithm, e.g., via the shared properties of the wireless channel. User $U_i$ acquires two short passwords $pw \textsubscript{i, i-1}$ with $U \textsubscript{i-1}$ and $pw \textsubscript{i, i+1}$ with $U \textsubscript{i+1}$.

Now, any secure pairwise password authenticated key exchange can be employed, so that $U_i$ can obtain two secret values $K_i^l$ and $K_i^r$ with his respective neighbors in the logical ring. $U_i$ computes 

$$V_i^l = K_i^l * H(K_i^r), V_i^r = K_i^r * H(K_i^l)$$

and 
 
$$X_i = H(K_i^l) \oplus H(K_i^r)$$
 
Then, at the end he broadcasts $ m_i = < U_i , U\textsubscript{i-1} , V_i^l ; U_i , U\textsubscript{i+1}, V_i^r ; X_i > $

and saves the exchanged messages $m_i$. In order to establish a group key, each user first authenticates his neighbors in the logical ring and checks if $X_1 \oplus X_2 \oplus \dots \oplus X_n = 0$.

E.g., for $U_6$ to verify $U_5$, he computes 
$E_6^l = V_5^r / K_6^l$. 
 As a reminder, per definition, $V_5^r = K_5^r * H(K_5^l) = K_6^l * H(K_5^l)$, therefore $E_6^l = H(K_5^l)$. So he has to check if 
$X_5 = H(K_6^l) \oplus E_6^l = H(K_6^l) \oplus H(K_5^l) = H(K_5^r) \oplus H(K_5^l)$ as per definition of $X_5$. 

If this verification succeeds, he sets $K_i = H(K_i^l)$ and can compute the other n - 1 values $K_k = H(K_i^l) \oplus X_{i-1} \oplus \dots X_k$ where $k = i - j, j = 1, \dots, n-1$. 

As an example how $U_6$ can compute

$$K_4 = H(K_6^l) \oplus X_5 \oplus X_4 = H(K_6^l) \oplus H(K_5^l) \oplus H(K_5^r) \oplus H(K_4^l) \oplus H(K_4^r) = H(K_4^l)$$

Per definition $H(K_4^l) = K_4$ and $H(K_5^l) = H(K_4^r), H(K_5^r) = H(K_6^l)$. Therefore, those become 0 when XORing them together. The group key $K_g$ is then the product of all $K_i$.

## Other schemes

Other exemplary schemes in this category derive a common group key via audio or received signal strength. 
The shared secret key enables the devices to authenticate themselves. This makes those schemes much more easy-to-use, as there is no need of human involvement to verify the exchanged data. 

The biggest issue of these schemes is to extract a random enough key in a reasonable amount of time under a low bit mismatch rate, e.g., it would take more than a minute to produce a 128-bit key for AES, see [11], [12].

# Discussion

Coming back to the initially proposed list of requirements:

- **Scalability**: The human-aided verification phase of the keyless security protocols does not scale well beyond a certain group size. Therefore, key-based security schemes have an advantage here. 
- **Fault Tolerance**: For the secret key based security schemes, it is hard to keep the bit mismatch rate between the keys low. The keyless security schemes are arguably more robust, but they need to take human error into account during the verification phase. However, SafeSlinger, using decoy phrases, has an effective way to prevent careless users accepting wrong outputs.
- **Close to Zero Interaction**: Secret key based schemes operate autonomously, while keyless schemes usually require a human to verify the exchanged data. However, Chorus could be used to ensure the equality of the exchanged data without human aid.
- **Speed of Execution**: Counterintuitively, the autonomous secret key based schemes are not offering higher execution speed compared to keyless schemes, due to the low key extraction rates
- **Accessibility**: The verification phase in the keyless schemes require displays, and Spate even requires a camera, while wireless channel measurements like RSS are easily accessible. However, this is not a disadvantage, when considering smartphones as the target platform. 
- **Infrastructure-less**: All protocols operate without a fixed infrastructure. SafeSlinger relies on an untrusted server, but can also be used, when the users are not in the same location, e.g., via a conference line. 
- **Security**: SafeSlinger is based solely on cryptographic techniques, it makes it provably secure. So it's much easier to quantify security here, compared to key extraction, assuming the verification phase is done correctly.  

In conclusion, because they are required to work autonomously, IoT devices, e.g., in a wireless sensor network, are depending on the secret key based solution together with a key extraction mechanism. But their low secret key extraction rates are probably too bothersome for smartphone users. The verification phase of keyless schemes on the other hand can be implemented quite user friendly. Also, there exists an alternative to human-aided comparison of the exchanged data. I introduced an example of this with Chorus. Therefore, the use of keyless security schemes bundled with a physical layer primitive for secure comparison of the exchanged data seems to be promising, while secret key generation algorithms still struggle to provide high key generation rates.

# References

[1] T. Zillner. 2015. ZigBee Exploited: The Good, the bad and the ugly

[2] W. Diffie et al. 1992. Authentication and authenticated key exchanges

[3] D. Balfanz et al. 2002. Talking to strangers: Authentication in ad-hoc wireless networks

[4] A. Perrig et al. 2005. Seeing-is- believing: Using camera phones for human-verifiable authentication

[5] A. Studer et al. 2011. Don’t bump, shake on it: The exploitation of a popular accelerometer-based smart phone exchange and its secure replacement

[6] A. Studer et al. 2008. Gangs: gather, authenticate’n group securely

[7] A. Perrig et al. 2013. Safeslinger: easy-to-use and secure public-key exchange

[8] A. Perrig et al. 2010. Spate: Small-group PKI-less authenticated trust establishment

[9] Y. Hou et al. 2013. Chorus: Scalable in-band trust establishment for multiple constrained devices over the insecure wireless channel

[10] Y. Zhang et al. 2016. Password-authenticated group key exchange: A cross-layer design

[11] Z. Gu and Y. Liu. 2016. Scalable group audio-based authentication scheme for IoT devices

[12] H. Liu et al. 2014. Group secret key generation via received signal strength: Protocols, achievable rates, and implementation