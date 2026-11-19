# Lab 4 report

Answer from **your** build: every claim points to a file in this repository or to a line in `evidence/`.

## 1. Inventory

Every key and certificate you created: what it is, algorithm and size, where the key lives, who can read it,
validity, what it is used for.

| Item | Key location | Readable by | Valid until | Purpose |
|---|---|---|---|---|
| root CA, EC P-384, self-signed, `root/certs/ca.cert.pem` | `root/private/ca.key.pem`, encrypted (AES-256, passphrase) | the `root-ca` container (no network); on the host, my user account, but only I know the passphrase | 2046-11-11, 20 years (`openssl x509 -noout -enddate`) | signs the intermediate and the root CRL, nothing else; stopped the rest of the time |

**Lifetimes.** The root lasts 20 years, the intermediate 10, the server and client certificates 90 days.
Justify each number: 
**What would a shorter or a longer lifetime cost you ?**
> Le root (20 ans) : hors ligne, il ne signe qu'une fois et toute rotation oblige à redéployer une ancre de confiance partout — sa durée longue ne coûte rien car son exposition est nulle.
L'intermédiaire (10 ans, la moitié du root) : il ne dépasse jamais son émetteur, et se renouvelle sans toucher aux trust stores, contrairement au root.
Les feuilles (90 jours) : la révocation n'est pas fiable ici (nginx lit les CRL au démarrage, CRL valide 7 jours, curl ne vérifie pas le serveur), donc notAfter est la seule borne réelle sur une clé volée.
Pas 1 an : la clé d'alice volée en phase 5 resterait utilisable quatre fois plus longtemps (le maximum public est passé de 398 à 200 jours en mars 2026, 100 en 2027, 47 en 2029).
Pas 7 jours : 52 renouvellements manuels par an et par identité (genpkey, req, openssl ca, copie, restart nginx, CRL), sans ACME ici c'est une panne garantie la première semaine d'absence.
**Why 90 days for the server, and not 1 year, or 7 days?**
90 jours correspond à un trimestre. C'est tenable à la main, l'exposition est bornée et complémentaire de la CRL de 7 jours. La révocation se fait via un chemin rapide et l'expiration est un filet de sécurité.

## 2. Chain

Root, intermediate, leaves, with the extensions that constrain each one (pathlen, nameConstraints, EKU, SAN).

| Certificate | Subject, issuer | Constraining extensions | What they prevent |
|---|---|---|---|
| root CA | `C=FR, O=EPITA Lab, CN=EPITA Lab Root CA`, self-signed | `basicConstraints: critical, CA:TRUE, pathlen:1`; `keyUsage: critical, keyCertSign, cRLSign` | `pathlen:1`: below the root, one CA level only; a CA signed by the intermediate is refused. `keyUsage`: the root key signs certificates and CRLs, nothing else |

Read them from the certificates, not from the config:
`openssl x509 -in root/certs/ca.cert.pem -noout -subject -issuer -ext basicConstraints,keyUsage`.

## 3. Key compromise

For the server key, a client key, the intermediate key and the root key: how would you notice,
what stops working or becomes unsafe, how do you recover, how long does it take?

### Server key (`identities/www/www.key.pem`)

- How you notice: nothing in this build detects it: no monitoring, no alert. You learn it from outside: the key
  found in a backup, a leak, an intrusion on the host.
- What becomes unsafe: whoever can redirect traffic (DNS, same network) can pose as `www.lab.internal`: clients send
  it their requests and trust its answers. It still cannot reach the API as alice: it has no client key. Past
  sessions stay unreadable: TLS 1.3 key exchange is ephemeral (forward secrecy). Nothing stops working.
- Recovery: new key and CSR in `client`, sign in `issuing-ca`, copy to `identities/www/`, restart nginx. Revoke the
  old serial and publish the CRL, but the clients do not check the server's revocation (`curl` reads no CRL by
  default): the old certificate stays accepted.
- How long: 10 minutes to reissue; the stolen certificate stays usable until it expires, at most 90 days.

### Client key

- How you notice:
- What becomes unsafe:
- Recovery:
- How long:

### Intermediate key

- How you notice:
- What becomes unsafe:
- Recovery:
- How long:

### Root key

- How you notice:
- What becomes unsafe:
- Recovery:
- How long:

## 4. Operational gaps

search answer to those questions/weaknesses

- the root key's passphrase: 
**Where is it ?**
> La phrase secrète se trouve dans un gestionnaire de mots de passe. La clé chiffrée root/private/ca.key.pem est sur l'hôte via le bind mount ./root:/pki/root ; la phrase, elle, doit être ailleurs (sinon les deux moitiés sont au même endroit et le chiffrement ne sont plus efficaces).
**Who knows it ?**
> Idéalement plus d'une personne. Sinon il faut au minimum une trace écrite : un second détenteur désigné ou la phrase découpée en plusieurs morceaux stockés chez plusieurs personnes.
**What if that person leaves the company?**
> Sans trace, le root CA est perdue et il est impossible de signer un nouvel intermédiaire ni de publier une CRL root, donc à terme toute la PKI meurt à l'expiration. La sortie de secours coûte une nouvelle racine à redéployer dans tous les trust stores.

- the CRL expires after 7 days: what happens on day 8, and who regenerates it?
- the lab root is in no trust store: how would every laptop, server and container get it?
- the intermediate's key is not encrypted, unlike the root's. Search how it would be possible to use KMS for intermediate issuing CA.

