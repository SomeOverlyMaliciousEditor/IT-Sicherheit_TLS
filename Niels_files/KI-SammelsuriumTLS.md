# TLS

## Vertraulichkeit, Integrität, Authentizität

Die Auswirkungen von Fehlkonfigurationen auf die drei Schutzziele sind gravierend:

*   **Szenario 1: Veraltete Protokolle (z.B. TLS 1.0) & Downgrade-Angriffe**
    *   **Vertraulichkeit:** **Stark gefährdet.** Angriffe wie BEAST (gegen CBC-Modus in TLS 1.0) oder POODLE (gegen SSLv3) ermöglichen es einem Man-in-the-Middle (MitM), Teile des verschlüsselten Datenstroms zu entschlüsseln, insbesondere sicherheitskritische Informationen wie Session-Cookies.
    *   **Integrität:** **Geschwächt.** Obwohl MACs verwendet werden, können einige Angriffe auf den CBC-Modus auch die Integrität der übertragenen Daten beeinträchtigen.
    *   **Authentizität:** **Indirekt gefährdet.** Wenn ein Angreifer die Vertraulichkeit bricht und ein Session-Cookie stiehlt, kann er die Identität des Nutzers übernehmen (Session Hijacking). Die Server-Authentizität selbst ist nicht direkt betroffen, solange das Zertifikat gültig ist, aber die Authentizität der Sitzung geht verloren.

*   **Szenario 2: Schwache Cipher Suites (z.B. RC4, 3DES, Export-Ciphers)**
    *   **Vertraulichkeit:** **Direkt gebrochen.** RC4 hat bekannte statistische Schwächen, die es ermöglichen, den Klartext mit ausreichend Datenverkehr zu rekonstruieren. 3DES ist anfällig für den Sweet32-Angriff (Kollisionen auf 64-Bit-Blöcken). Export-Ciphers (z.B. 512-Bit-RSA) können durch Brute-Force in kurzer Zeit gebrochen werden (FREAK, Logjam Angriffe).
    *   **Integrität:** **Nicht direkt betroffen,** wenn ein starker Hash-Algorithmus (wie SHA-256) für das MAC verwendet wird. Bei modernen AEAD-Ciphers sind Vertraulichkeit und Integrität jedoch untrennbar gekoppelt.
    *   **Authentizität:** **Gefährdet,** wenn schwache Hash-Algorithmen (MD5, SHA-1) in der Zertifikatssignatur verwendet werden. Kollisionsangriffe auf SHA-1 sind praktisch durchführbar, was die Fälschung von Zertifikaten und damit die Kompromittierung der Server-Identität ermöglichen könnte.

*   **Szenario 3: Selbstsigniertes oder ungültiges Zertifikat**
    *   **Authentizität:** **Vollständig kompromittiert.** Der Client hat keine Möglichkeit, die Identität des Servers zu überprüfen, da das Zertifikat nicht von einer vertrauenswürdigen Stelle (CA) signiert ist. Ein MitM-Angreifer kann sein eigenes selbstsigniertes Zertifikat präsentieren. Ignoriert der Benutzer die Browser-Warnung, baut er eine Verbindung zum Angreifer auf, nicht zum eigentlichen Server.
    *   **Vertraulichkeit & Integrität:** **Als Konsequenz gebrochen.** Da der Client mit dem Angreifer kommuniziert, kann dieser den gesamten Datenverkehr mitlesen (Vertraulichkeit gebrochen) und nach Belieben manipulieren (Integrität gebrochen), auch wenn die Verbindung zum Angreifer selbst "verschlüsselt" ist.

## Härtungsmaßnahmen

Um die identifizierten Schwachstellen zu beheben, sind folgende konkrete Härtungsmaßnahmen serverseitig umzusetzen:

1.  **Protokollversionen einschränken**
    *   **Maßnahme:** Deaktivieren Sie serverseitig alle veralteten und unsicheren Protokolle wie SSLv2, SSLv3, TLS 1.0 und TLS 1.1. Erzwingen Sie die Verwendung von **TLS 1.2** und, wo immer möglich, **TLS 1.3**.
    *   **Begründung:**
        *   TLS 1.3 ist inhärent sicherer: Es entfernt veraltete Kryptographie (z.B. CBC-Modus, RSA-Key-Exchange), beschleunigt den Handshake und bietet robusten Schutz vor Downgrade-Angriffen.
        *   Die Deaktivierung älterer Versionen verhindert Downgrade-Angriffe wie POODLE und schließt bekannte Schwachstellen (z.B. BEAST) von vornherein aus. Die serverseitige Konfiguration ist entscheidend, da sie die "unterste" Sicherheitsgrenze festlegt, die kein Client unterschreiten kann.

2.  **Sichere Cipher Suites konfigurieren**
    *   **Maßnahme:** Definieren Sie eine explizite, kurze Liste von starken Cipher Suites und kontrollieren Sie deren Reihenfolge (`server-side preference`). Priorisieren Sie Suiten mit **AEAD** (Authenticated Encryption with Associated Data) und **Perfect Forward Secrecy (PFS)**.
        *   **Für TLS 1.3 (Standard):** `TLS_AES_256_GCM_SHA384`, `TLS_CHACHA20_POLY1305_SHA256`, `TLS_AES_128_GCM_SHA256`.
        *   **Für TLS 1.2 (Beispiele):** `ECDHE-ECDSA-AES256-GCM-SHA384`, `ECDHE-RSA-AES256-GCM-SHA384`, `ECDHE-ECDSA-CHACHA20-POLY1305`, `ECDHE-RSA-CHACHA20-POLY1305`.
    *   **Begründung:**
        *   **AEAD-Ciphers (AES-GCM, ChaCha20-Poly1305):** Bieten gleichzeitig Vertraulichkeit und Integrität in einem Schritt. Sie sind immun gegen Padding-Oracle-Angriffe, die CBC-basierte Ciphers plagten.
        *   **PFS (via ECDHE):** Stellt sicher, dass die Kompromittierung des langlebigen privaten Schlüssels des Servers nicht zur Entschlüsselung vergangener Sitzungen führt. Für jede Sitzung wird ein neuer, kurzlebiger Schlüssel ausgehandelt und nach der Sitzung verworfen.
        *   Das Entfernen schwacher Ciphers (RC4, 3DES, NULL, anonyme Ciphers) verhindert, dass diese überhaupt ausgehandelt werden können.

3.  **Zertifikatsmanagement und Vertrauensketten sicherstellen**
    *   **Maßnahme:**
        *   Verwenden Sie ausschließlich Zertifikate von öffentlich vertrauenswürdigen Certificate Authorities (CAs) mit einer Schlüssellänge von mindestens 2048 Bit (RSA) oder 256 Bit (ECDSA).
        *   Stellen Sie sicher, dass die gesamte Zertifikatskette (Root-CA -> Intermediate-CA -> Server-Zertifikat) an den Client gesendet wird, um "Chain of Trust"-Fehler zu vermeiden.
        *   Implementieren Sie **OCSP Stapling**, um die Überprüfung des Widerrufsstatus zu beschleunigen und die Privatsphäre der Clients zu schützen.
    *   **Begründung:**
        *   Ein von einer öffentlichen CA signiertes Zertifikat ist der einzige Weg, um die **Authentizität** des Servers gegenüber beliebigen Clients im Internet zu beweisen. Es bricht den MitM-Angriff, da der Angreifer kein gültiges Zertifikat für Ihre Domain von einer vertrauenswürdigen CA erhalten kann.
        *   OCSP Stapling verhindert, dass der Client selbst eine Anfrage an die CA stellen muss, was den Handshake verlangsamt und der CA verrät, welche Seite der Nutzer besucht (Privacy Leak).

4.  **HTTP Strict Transport Security (HSTS) aktivieren**
    *   **Maßnahme:** Senden Sie den HSTS-Header vom Server für alle HTTPS-Antworten, z.B. `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload`.
    *   **Begründung:**
        *   HSTS weist den Browser an, für die angegebene Domain für die Dauer von `max-age` (hier: 1 Jahr) ausschließlich HTTPS-Verbindungen zu verwenden.
        *   Dies verhindert **SSL-Stripping-Angriffe**, bei denen ein Angreifer eine initiale, ungesicherte HTTP-Anfrage abfängt und den Nutzer auf einer gefälschten HTTP-Seite hält, um den Verkehr mitzulesen. Der Browser wird die unsichere Verbindung gar nicht erst versuchen. Die `preload`-Direktive ermöglicht die Aufnahme in eine vom Browser geführte Liste, die diesen Schutz von der ersten Verbindung an bietet.

---

## Kommentierte Gesamtanalyse und Demonstration

Dieser Abschnitt fasst alle vorherigen Punkte zusammen und demonstriert die Funktionsweise sowie die Schwachstellen von TLS anhand eines konkreten, kommentierten Szenarios, wie es in der Aufgabenstellung gefordert wird.

### 1. Theoretische Grundlagen: Der TLS-Handshake (Kurzüberblick)

Der TLS-Handshake dient der Aushandlung einer sicheren Verbindung. Die wichtigsten Unterschiede zwischen den relevanten Versionen sind:

*   **TLS 1.2 Handshake (vereinfacht):**
    1.  `ClientHello`: Client sendet unterstützte TLS-Versionen, Cipher Suites und eine Zufallszahl.
    2.  `ServerHello`: Server wählt die höchste gemeinsame TLS-Version und eine Cipher Suite aus und sendet seine Zufallszahl.
    3.  `Certificate`: Server sendet sein Zertifikat und die Zertifikatskette.
    4.  `ServerKeyExchange` (optional, für PFS): Server sendet seine Diffie-Hellman-Parameter.
    5.  `ServerHelloDone`: Server ist fertig.
    6.  `ClientKeyExchange`: Client sendet seinen DH-Anteil.
    7.  `ChangeCipherSpec`: Client schaltet auf Verschlüsselung um.
    8.  `Finished`: Verschlüsselte Prüfnachricht vom Client.
    9.  `ChangeCipherSpec` & `Finished`: Server tut dasselbe.
    *   **Nachteile:** Zwei Round-Trips (hohe Latenz), viele veraltete und komplexe Optionen, die zu Fehlkonfigurationen führen können.

*   **TLS 1.3 Handshake (vereinfacht):**
    1.  `ClientHello`: Client sendet unterstützte Versionen, eine Zufallszahl und **spekulativ bereits seine DH-Schlüsselanteile** für die wahrscheinlichste Cipher Suite.
    2.  `ServerHello`: Server wählt Version und Cipher, sendet sein **Zertifikat und seine DH-Anteile** und eine `Finished`-Nachricht – **alles in einer einzigen Antwort**.
    3.  `Finished`: Client prüft alles und sendet seine `Finished`-Nachricht.
    *   **Vorteile:** Nur ein Round-Trip (schneller), alle unsicheren Optionen (statische RSA-Schlüsselaustausche, CBC-Ciphers, schwache Hashes) wurden entfernt. Sicherer und einfacher per Design.

### 2. Szenario-Design

Wir definieren zwei Server, um die Unterschiede zwischen einer sicheren und einer unsicheren Konfiguration zu demonstrieren.

*   **Server 1: `secure.example.com` (Sichere Konfiguration)**
    *   **Protokolle:** Nur TLS 1.3 und TLS 1.2 aktiviert.
    *   **Cipher Suites:** Nur moderne AEAD-Ciphers mit PFS, z.B. `TLS_AES_256_GCM_SHA384` (für TLS 1.3) und `ECDHE-RSA-AES256-GCM-SHA384` (für TLS 1.2).
    *   **Zertifikat:** Gültiges Zertifikat, signiert von einer vertrauenswürdigen CA (z.B. Let's Encrypt), inklusive der vollständigen Kette.

*   **Server 2: `insecure.example.com` (Unsichere Konfiguration)**
    *   **Protokolle:** Veraltetes TLS 1.0 aktiviert.
    *   **Cipher Suites:** Schwache Cipher Suites wie `RC4-MD5` oder `DES-CBC-SHA` sind erlaubt.
    *   **Zertifikat:** Ein selbstsigniertes Zertifikat.

### 3. Simulation und Analyse von Paketmitschnitten (via `openssl`)

Wir simulieren die Verbindungsversuche mit dem Kommandozeilen-Tool `openssl s_client`, das uns eine detaillierte Ausgabe des Handshakes liefert.

#### **Fall 1: Sicherer Verbindungsaufbau zu `secure.example.com`**

Ein moderner Client verbindet sich mit dem sicher konfigurierten Server.

```bash
# Simulation eines modernen Clients
openssl s_client -connect secure.example.com:443 -tls1_3
```

**Analyse des (erwarteten) Outputs:**
*   `CONNECTED(00000003)`: TCP-Verbindung erfolgreich.
*   `New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384`: **Erfolg!** Es wurde das stärkste Protokoll (TLS 1.3) und eine starke AEAD-Cipher Suite ausgehandelt.
*   `Server public key is 2048 bit`: Die Schlüssellänge ist ausreichend.
*   `Verify return code: 0 (ok)`: **Authentizität bestätigt.** Der Client konnte die Zertifikatskette erfolgreich gegen seinen lokalen Trust Store validieren.

**Bewertung der Schutzziele:**
*   **Authentizität:** Vollständig gewährleistet. Wir sprechen garantiert mit `secure.example.com`.
*   **Vertraulichkeit:** Vollständig gewährleistet durch AES-256-GCM.
*   **Integrität:** Vollständig gewährleistet durch den GCM-Teil der Cipher Suite.

#### **Fall 2: Downgrade-Szenario mit `insecure.example.com`**

Ein Angreifer zwingt einen (veralteten) Client, sich über TLS 1.0 zu verbinden.

```bash
# Simulation eines Downgrades oder eines alten Clients
openssl s_client -connect insecure.example.com:443 -tls1
```

**Analyse des (erwarteten) Outputs:**
*   `New, TLSv1.0, Cipher is RC4-MD5`: **Sicherheitslücke!** Die Verbindung wurde mit einem veralteten Protokoll und einer als gebrochen geltenden Cipher Suite (RC4) ausgehandelt.
*   `verify error:num=18:self signed certificate`: **Authentizität gebrochen!** Der Client erkennt, dass das Zertifikat nicht vertrauenswürdig ist. Ein Browser würde hier eine unübersehbare Warnung anzeigen. Ignoriert der Nutzer diese, ist der Angriff erfolgreich.

**Bewertung der Schutzziele:**
*   **Authentizität:** **Kompromittiert.** Der Client hat keine Ahnung, ob er mit dem echten Server oder einem Man-in-the-Middle-Angreifer spricht.
*   **Vertraulichkeit:** **Gebrochen.** Der Datenverkehr kann mit bekannt gewordenen Angriffen auf RC4 entschlüsselt werden.
*   **Integrität:** **Stark geschwächt.** MD5 ist für Kollisionen anfällig.

#### **Fall 3: Handshake-Fehler (Sicherheitsmechanismus greift)**

Ein moderner, sicher konfigurierter Client versucht, sich mit `insecure.example.com` zu verbinden.

```bash
# Simulation eines modernen Clients, der nur TLS 1.2+ erlaubt
openssl s_client -connect insecure.example.com:443 -no_tls1 -no_tls1_1
```

**Analyse des (erwarteten) Outputs:**
*   `sslv3 alert handshake failure`: **Verbindung fehlgeschlagen.**
*   `reason(107)`: Der Client und der Server konnten sich auf keine gemeinsamen Parameter einigen. Der Client bot nur sichere Ciphers und Protokolle an, die der unsichere Server nicht unterstützte (oder umgekehrt).

**Bewertung der Schutzziele:**
*   **Dies ist ein Erfolg aus Sicherheitssicht!** Der Client hat sich geweigert, eine unsichere Verbindung aufzubauen. Alle Schutzziele bleiben intakt, da keine Kommunikation stattfindet.

### 4. Fazit und Wirksamkeit der Härtungsmaßnahmen

Die Simulation zeigt deutlich, wie die zuvor definierten Härtungsmaßnahmen die identifizierten Schwachstellen verhindern:

1.  **Protokollversionen einschränken:** Hätte der `insecure.example.com` nur TLS 1.2+ erlaubt, wäre Fall 2 (Downgrade) unmöglich gewesen und hätte wie in Fall 3 zu einem sicheren Abbruch geführt.
2.  **Sichere Cipher Suites konfigurieren:** Hätte der Server keine schwachen Ciphers wie `RC4-MD5` angeboten, wäre ebenfalls keine unsichere Verbindung zustande gekommen. Die serverseitige Präferenz (`server-side preference`) stellt sicher, dass immer die stärkste gemeinsame Cipher gewählt wird.
3.  **Korrektes Zertifikatsmanagement:** Die Verwendung eines gültigen, von einer CA signierten Zertifikats ist die **Grundvoraussetzung** zur Abwehr von Man-in-the-Middle-Angriffen. Ohne sie sind Vertraulichkeit und Integrität wertlos, da man mit dem Angreifer verschlüsselt kommuniziert.
4.  **HSTS implementieren:** HSTS agiert als zusätzliche Schutzschicht und verhindert SSL-Stripping-Angriffe, bei denen ein Angreifer den Nutzer von vornherein auf einer unverschlüsselten HTTP-Verbindung hält. Es zwingt den Browser, direkt eine sichere TLS-Verbindung zu versuchen, wodurch die Angriffsfläche für Downgrades massiv reduziert wird.

---

## Übersicht der TLS-Konfigurationen und deren Sicherheit

Die folgende Tabelle gibt einen Überblick über verschiedene Cipher Suites in unterschiedlichen TLS/SSL-Versionen und bewertet deren Sicherheit.

| Protokoll | Schlüsselaustausch | Verschlüsselung | Integrität (MAC) | Sicherheitsbewertung | Bekannte Schwachstellen / Anmerkungen |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **SSL 3.0** | RSA | RC4 | MD5 | **Gebrochen** | POODLE, RC4-Schwächen, MD5-Kollisionen, kein PFS. **Sofort deaktivieren.** |
| **SSL 3.0** | RSA | 3DES-CBC | SHA-1 | **Gebrochen** | POODLE, Sweet32 (gegen 3DES), SHA-1-Schwächen, kein PFS. **Sofort deaktivieren.** |
| **TLS 1.0** | RSA | AES-128-CBC | SHA-1 | **Unsicher** | BEAST, Lucky13 (gegen CBC-Modus), kein PFS. Anfällig für Downgrade-Angriffe. |
| **TLS 1.0** | DHE | AES-128-CBC | SHA-1 | **Unsicher** | Bietet zwar PFS, aber ist anfällig für BEAST/Lucky13 und Logjam (schwache DH-Parameter). |
| **TLS 1.1** | ECDHE-RSA | AES-128-CBC | SHA-1 | **Veraltet** | Mildert BEAST, aber andere CBC-Schwächen und SHA-1 bleiben. Bietet PFS, aber gilt nicht mehr als sicher. |
| **TLS 1.2** | RSA | AES-256-GCM | GCM (AEAD) | **Akzeptabel (Legacy)** | **Kein Perfect Forward Secrecy (PFS)!** Kompromittierung des Server-Schlüssels deckt alle vergangenen Sitzungen auf. |
| **TLS 1.2** | ECDHE-RSA | AES-128-CBC | SHA-256 | **Akzeptabel (Legacy)** | Bietet PFS, aber der CBC-Modus ist theoretisch anfällig für Padding-Oracle-Angriffe. AEAD-Ciphers sind vorzuziehen. |
| **TLS 1.2** | ECDHE-RSA | **AES-256-GCM** | **GCM (AEAD)** | **Sicher (Best Practice)** | **Starke Konfiguration.** Bietet Perfect Forward Secrecy (PFS) und Authenticated Encryption (AEAD). |
| **TLS 1.2** | ECDHE-ECDSA | **ChaCha20-Poly1305** | **Poly1305 (AEAD)** | **Sicher (Best Practice)** | **Starke Konfiguration.** Alternative zu AES, oft performanter auf Geräten ohne AES-Hardwarebeschleunigung. |
| **TLS 1.3** | (EC)DHE | **AES-256-GCM** | **AEAD + HKDF-SHA384** | **Sehr Sicher (State-of-the-Art)** | **Standard in TLS 1.3.** PFS ist obligatorisch. Veraltete Krypto wurde entfernt. Schnellerer Handshake. |
| **TLS 1.3** | (EC)DHE | **ChaCha20-Poly1305** | **AEAD + HKDF-SHA256** | **Sehr Sicher (State-of-the-Art)** | **Standard in TLS 1.3.** Exzellente Sicherheit und Performance. |
| **TLS 1.3** | (EC)DHE | **AES-128-GCM** | **AEAD + HKDF-SHA256** | **Sehr Sicher (State-of-the-Art)** | **Standard in TLS 1.3.** AES-128 ist für die meisten Anwendungsfälle mehr als ausreichend sicher. |

**Legende der Sicherheitsbewertung:**

*   **Gebrochen:** Praktische Angriffe existieren, die die Sicherheit vollständig aushebeln. Die Nutzung ist grob fahrlässig.
*   **Unsicher:** Signifikante theoretische und/oder praktische Schwächen sind bekannt. Sollte nicht mehr verwendet werden.
*   **Veraltet:** Wurde durch modernere, robustere Alternativen ersetzt. Bietet keinen Schutz nach heutigem Stand der Technik.
*   **Akzeptabel (Legacy):** Kann aus Kompatibilitätsgründen noch notwendig sein, hat aber Nachteile (z.B. fehlendes PFS oder kein AEAD).
*   **Sicher (Best Practice):** Entspricht den aktuellen Empfehlungen für eine sichere Konfiguration (z.B. von BSI, Mozilla).
*   **Sehr Sicher (State-of-the-Art):** Die modernste und sicherste verfügbare Konfiguration ohne bekannte Schwächen.

---

## Glossar und Quellen

### Abkürzungen

*   **3DES**: Triple Data Encryption Standard. Ein symmetrisches Verschlüsselungsverfahren, das als unsicher gilt (anfällig für Sweet32).
*   **AEAD**: Authenticated Encryption with Associated Data. Ein Verschlüsselungsmodus, der gleichzeitig Vertraulichkeit und Integrität sicherstellt (z.B. AES-GCM).
*   **AES**: Advanced Encryption Standard. Ein weit verbreiteter und sicherer symmetrischer Verschlüsselungsalgorithmus.
*   **BEAST**: Browser Exploit Against SSL/TLS. Ein Angriff auf den CBC-Modus in TLS 1.0.
*   **BSI**: Bundesamt für Sicherheit in der Informationstechnik.
*   **CA**: Certificate Authority. Eine Zertifizierungsstelle, die digitale Zertifikate ausstellt und deren Vertrauenswürdigkeit bürgt.
*   **CBC**: Cipher Block Chaining. Ein Betriebsmodus für Blockchiffren, der in älteren TLS-Versionen anfällig für Angriffe war.
*   **CVE**: Common Vulnerabilities and Exposures. Ein System zur Katalogisierung bekannter Sicherheitsschwachstellen.
*   **DHE**: Diffie-Hellman Ephemeral. Ein Schlüsselaustauschprotokoll, das Perfect Forward Secrecy (PFS) ermöglicht.
*   **ECDHE**: Elliptic-Curve Diffie-Hellman Ephemeral. Eine effizientere Variante von DHE, die auf elliptischen Kurven basiert.
*   **ECDSA**: Elliptic Curve Digital Signature Algorithm. Ein Signaturalgorithmus, der auf elliptischen Kurven basiert.
*   **GCM**: Galois/Counter Mode. Ein AEAD-Betriebsmodus für Blockchiffren, z.B. in AES-GCM.
*   **HKDF**: HMAC-based Key Derivation Function. Eine Funktion zur Ableitung kryptographischer Schlüssel aus einem Geheimnis.
*   **HMAC**: Hash-based Message Authentication Code. Ein Verfahren zur Sicherstellung der Integrität und Authentizität von Nachrichten.
*   **HSTS**: HTTP Strict Transport Security. Ein Mechanismus, der Browser anweist, eine Website nur über HTTPS zu besuchen.
*   **MAC**: Message Authentication Code. Ein kurzes Stück Information, das zur Authentifizierung einer Nachricht verwendet wird.
*   **MD5**: Message Digest 5. Eine veraltete und unsichere Hashfunktion.
*   **MitM**: Man-in-the-Middle. Ein Angreifer, der sich zwischen zwei Kommunikationspartner schaltet.
*   **OCSP**: Online Certificate Status Protocol. Ein Protokoll zur Überprüfung des Gültigkeitsstatus von Zertifikaten in Echtzeit.
*   **PFS**: Perfect Forward Secrecy. Eine Eigenschaft von Schlüsselaustauschprotokollen, die sicherstellt, dass die Kompromittierung eines Langzeitschlüssels nicht zur Entschlüsselung vergangener Sitzungen führt.
*   **PKI**: Public Key Infrastructure. Ein System zur Verwaltung und Nutzung von öffentlichen Schlüsseln und Zertifikaten.
*   **POODLE**: Padding Oracle On Downgraded Legacy Encryption. Ein Angriff, der eine Schwachstelle im CBC-Modus in SSL 3.0 ausnutzt.
*   **RC4**: Rivest Cipher 4. Eine Stromchiffre, die als unsicher gilt und bekannte Schwächen aufweist.
*   **RFC**: Request for Comments. Eine Reihe von technischen und organisatorischen Dokumenten des Internets.
*   **RSA**: Rivest–Shamir–Adleman. Ein weit verbreitetes asymmetrisches Kryptosystem für Verschlüsselung und digitale Signaturen.
*   **SHA**: Secure Hash Algorithm. Eine Familie von kryptographischen Hashfunktionen (z.B. SHA-1, SHA-256, SHA-384).
*   **SSL**: Secure Sockets Layer. Der Vorgänger von TLS, gilt als veraltet und unsicher.
*   **TLS**: Transport Layer Security. Ein Verschlüsselungsprotokoll zur sicheren Datenübertragung im Internet.

### Quellen und weiterführende Informationen

*   **Vorlesungsfolien und Übungsaufgaben zur Vorlesung "IT-Sicherheit"**: Die primäre Grundlage für die erstellten Inhalte.
*   **BSI TR-02102-1**: [Kryptographische Verfahren: Empfehlungen und Schlüssellängen](https://www.bsi.bund.de/DE/Themen/Unternehmen-und-Organisationen/Standards-und-Zertifizierung/Technische-Richtlinien/TR-nach-Thema-sortiert/tr02102/tr02102_node.html). Technische Richtlinie des Bundesamts für Sicherheit in der Informationstechnik.
*   **Mozilla SSL Configuration Generator**: Ein praktisches Werkzeug und eine Referenz für sichere Serverkonfigurationen. [ssl-config.mozilla.org](https://ssl-config.mozilla.org/)
*   **RFC 5246**: [The Transport Layer Security (TLS) Protocol Version 1.2](https://datatracker.ietf.org/doc/html/rfc5246).
*   **RFC 8446**: [The Transport Layer Security (TLS) Protocol Version 1.3](https://datatracker.ietf.org/doc/html/rfc8446).
*   **RFC 2104**: [HMAC: Keyed-Hashing for Message Authentication](https://datatracker.ietf.org/doc/html/rfc2104).
*   **RFC 6797**: [HTTP Strict Transport Security (HSTS)](https://datatracker.ietf.org/doc/html/rfc6797).
*   **OWASP (Open Web Application Security Project)**: Eine wichtige Quelle für Informationen über Web-Sicherheit, einschließlich Angriffe und sichere Konfigurationen. [owasp.org](https://owasp.org/)




___
___
___

<thinking>
Now I have good information about TLS hardening measures from the BSI. Let me now write a comprehensive ~1.5 DIN A4 page (approximately 600-900 words) covering:

1. Simulation eines Downgrade- oder Handshake-Fehlerfalls (with steps and log analysis)
2. Bewertung der Auswirkungen auf CIA
3. Härtungsmaßnahmen

The user asked specifically for the LAST TWO sections:
- "Simulieren Sie einen Downgrade- oder Handshake-Fehlerfall. Beschreiben Sie die einzelnen Schritte des Verbindungsaufbaus und analysieren Sie anhand von Logs oder Paketmitschnitten, an welcher Stelle die Sicherheitsmechanismen versagen oder geschwächt werden."
- "Bewerten Sie die Auswirkungen auf Vertraulichkeit, Integrität und Authentizität."
- "Entwickeln Sie abschließend konkrete Härtungsmaßnahmen und begründen Sie, warum diese die identifizierten Schwachstellen wirksam verhindern."

Let me write this up properly with sources.
</thinking>

***

## Simulation eines Downgrade-Angriffs und Handshake-Fehleranalyse

### Szenario: POODLE-basierter Downgrade-Angriff

Zur Veranschaulichung werden zwei Konfigurationspaare gegenübergestellt: **Client A / Server A** mit einer sicheren TLS 1.3-Konfiguration sowie **Client B / Server B**, der aus Kompatibilitätsgründen noch SSLv3 und TLS 1.0 aktiviert hat.

Mallory, ein aktiver Man-in-the-Middle-Angreifer im selben Netzwerksegment (z. B. öffentliches WLAN), manipuliert den Verbindungsaufbau von Client B.

**Schrittweise Verbindungsaufbau mit Fehlereinleitung:**

```
1. Client B  →  Server B:  ClientHello (max. TLS 1.2, Cipher-Liste inkl. RC4, 3DES)
2. Mallory   →             Blockiert das TCP-Paket / injiziert TCP-RST
3. Client B  →  Server B:  ClientHello (Fallback: TLS 1.0) – „Downgrade Dance"
4. Mallory   →             Erneute Unterbrechung
5. Client B  →  Server B:  ClientHello (Fallback: SSLv3, CBC-Cipher)
6. Server B  →  Client B:  ServerHello (SSLv3, TLS_RSA_WITH_3DES_EDE_CBC_SHA)
7.                         Handshake abgeschlossen – Verbindung aktiv mit SSLv3
```

Relevanter Log-Auszug (OpenSSL-Debug):

```
>>> TLS client hello, version TLS 1.0 ... connection reset
>>> TLS client hello, version SSL 3.0
<<< ServerHello, version SSL 3.0, cipher TLS_RSA_WITH_3DES_EDE_CBC_SHA
<<< Certificate, length 1024-bit RSA key
>>> ClientKeyExchange
... [Handshake complete, SSLv3 active]
```

**Wo versagen die Sicherheitsmechanismen?** Erstens fehlt dem Server das `TLS_FALLBACK_SCSV`-Signal (RFC 7507), das dem Server signalisieren würde, dass der Client eigentlich eine höhere Version beherrscht und ein Downgrade daher unzulässig ist. Zweitens erlaubt die CBC-Cipher-Suite in SSLv3 den klassischen POODLE-Angriff (Padding Oracle on Downgraded Legacy Encryption): Da SSLv3 kein fixes Padding-Schema erzwingt, kann Mallory durch gezieltes Einschleusen präparierter CBC-Blöcke systematisch einzelne Klartextbytes des verschlüsselten Cookies rekonstruieren – typischerweise nach 256 Anfragen pro Byte. [bsi.bund](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Publikationen/TechnischeRichtlinien/TR02102/BSI-TR-02102-2.pdf?__blob=publicationFile&v=14)

Beim zweiten Szenario – einem selbstsignierten Zertifikat – bricht der TLS-Handshake an anderer Stelle:

```
<<< Certificate: CN=intern.server, Issuer=CN=intern.server (self-signed)
>>> [Browser: CERTIFICATE_VERIFY_FAILED – Issuer not in trusted root store]
    Fehlercode: ERR_CERT_AUTHORITY_INVALID
```

Client A akzeptiert das Zertifikat nicht, da kein vertrauenswürdiger CA-Anker in der Vertrauenskette vorhanden ist. Sicherheitsbewusste Nutzer oder strikte Clients terminieren die Verbindung; ältere oder falsch konfigurierte Anwendungen ignorieren den Fehler – womit die Zertifikatsprüfung vollständig wirkungslos wird.

***

## Auswirkungen auf Vertraulichkeit, Integrität und Authentizität

**Vertraulichkeit** ist im Downgrade-Szenario unmittelbar gefährdet: SSLv3 mit CBC-Cipher bietet keinen wirksamen Schutz gegen den POODLE-Angriff. Mallory kann nach erfolgreicher Entschlüsselung Session-Cookies im Klartext lesen und damit Session-Hijacking betreiben. RC4, das in veralteten Konfigurationen ebenfalls auftaucht, ist seit RFC 7465 (2015) explizit verboten, da statistische Biases eine Klartextrekonstruktion ermöglichen. [twingate](https://www.twingate.com/blog/glossary/tls-downgrade-attack)

**Integrität** wird durch das Fehlen von AEAD-Cipher-Suites (Authenticated Encryption with Associated Data) verletzt. SSLv3 und frühe TLS-Versionen nutzen MAC-then-Encrypt statt Encrypt-then-MAC, was Padding-Oracle-Angriffe erst möglich macht. Im selbstsignierten Zertifikats-Szenario besteht zudem keine verifizierbare Bindung zwischen dem präsentierten Schlüssel und einer überprüften Identität – Mallory kann ein eigenes selbstsigniertes Zertifikat einschleusen, ohne dass Client-Software (bei deaktivierter Zertifikatsprüfung) dies bemerkt.

**Authentizität** ist das Kernproblem des zweiten Szenarios: Ohne vertrauenswürdige CA-Kette kann nicht sichergestellt werden, dass der Kommunikationspartner der erwartete Server ist. Ein Angreifer kann sich mit beliebigem selbstsigniertem Zertifikat als legitimer Server ausgeben. Die gesamte Authentisierungsleistung von TLS – die Bindung eines öffentlichen Schlüssels an eine überprüfte Identität durch eine PKI-Vertrauenskette – entfällt. [bsi.bund](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Publikationen/TechnischeRichtlinien/TR02102/BSI-TR-02102-2.html)

***

## Konkrete Härtungsmaßnahmen

Das BSI definiert in der **TR-02102-2** (Version 2026-01) verbindliche Mindestanforderungen, aus denen sich folgende Maßnahmen ableiten: [bsi.bund](https://www.bsi.bund.de/DE/Themen/Unternehmen-und-Organisationen/Standards-und-Zertifizierung/Technische-Richtlinien/TR-nach-Thema-sortiert/tr02102/tr02102_node.html)

**1. Ausschließlich TLS 1.2 und TLS 1.3 zulassen**
SSLv2, SSLv3, TLS 1.0 und TLS 1.1 sind gemäß BSI TR-02102-2 und RFC 8996 nicht mehr zu verwenden. Auf Apache/Nginx-Servern: [bsi.bund](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Publikationen/TechnischeRichtlinien/TR02102/BSI-TR-02102-2.pdf?__blob=publicationFile&v=14)
```nginx
ssl_protocols TLSv1.2 TLSv1.3;
```
Dies verhindert Downgrade-Angriffe auf SSLv3 strukturell, da der Verhandlungsspielraum eliminiert wird.

**2. TLS_FALLBACK_SCSV aktivieren (RFC 7507)**
Das Signaling Cipher Suite Value teilt dem Server mit, dass der Client einen Protokoll-Fallback durchführt. Ein RFC 7507-konformer Server lehnt Verbindungen ab, bei denen ein Downgrade erzwungen wurde – POODLE und verwandte Angriffe werden so auf Protokollebene verhindert. [kryptus](https://kryptus.com/de/ataque-poodle-quebrando-o-tls-com-ssl-3/)

**3. Ausschließlich AEAD-Cipher-Suites verwenden**
Das BSI empfiehlt für TLS 1.2 nur noch ECDHE-basierte Suites mit GCM oder ChaCha20-Poly1305; für TLS 1.3 ist AEAD obligatorisch. Empfohlene Konfiguration: [bsi.bund](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Publikationen/TechnischeRichtlinien/TR03116/BSI-TR-03116-4.pdf?__blob=publicationFile&v=8)
```
TLS_AES_256_GCM_SHA384           (TLS 1.3, Prio 1)
TLS_CHACHA20_POLY1305_SHA256     (TLS 1.3, Prio 2)
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384  (TLS 1.2)
```
RC4, 3DES und CBC-basierte Suites sind vollständig zu deaktivieren. AEAD verhindert Padding-Oracle-Angriffe, da Integrität und Verschlüsselung atomisch geprüft werden.

**4. Zertifikate von vertrauenswürdigen CAs mit OCSP Stapling**
Selbstsignierte Zertifikate sind ausschließlich in isolierten Testumgebungen akzeptabel. Für Produktivumgebungen sind Zertifikate einer anerkannten CA (z. B. DFN-PKI für Hochschulen, Let's Encrypt für öffentliche Dienste) einzusetzen. OCSP Stapling stellt sicher, dass der Server selbst einen aktuellen Sperrstatus-Nachweis mitliefert, ohne dass der Client einen separaten OCSP-Request senden muss – dies verhindert auch Sperrstatus-Angriffe bei unterbrochener OCSP-Erreichbarkeit.

**5. HTTP Strict Transport Security (HSTS) mit langer max-age**
```
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```
HSTS verhindert, dass Browser nach einem initialen HTTPS-Aufruf jemals wieder auf HTTP oder eine schwächere TLS-Version wechseln. In Kombination mit dem HSTS-Preload-Verzeichnis wird selbst die allererste Verbindung abgesichert. [all-about-security](https://www.all-about-security.de/bsi-setzt-auslaufdatum-fuer-klassische-verschluesselungsverfahren/)

**6. RSA-Schlüssellänge und Forward Secrecy**
Das BSI fordert für RSA-Schlüssel mindestens 3000 Bit (Empfehlung bis 2031). Zugleich ist ephemerer Schlüsselaustausch (ECDHE, DHE) zu bevorzugen, da er **Perfect Forward Secrecy** garantiert: Ein kompromittierter langfristiger privater Schlüssel entschlüsselt keine vergangenen Sitzungen. [heise](https://www.heise.de/news/BSI-Verwirrung-um-Anforderungen-an-Schluessellaengen-fuer-TLS-Verbindungen-9596072.html)



___
___


<thinking>
Now I have enough information to write a well-sourced Ausarbeitung for the last two sections: "Simulation eines Downgrade- oder Handshake-Fehlerfalls" and "Härtungsmaßnahmen". Let me write this up properly.

The task says to write about:
1. Simulation eines Downgrade- oder Handshake-Fehlerfalls - describe the individual steps of the connection setup and analyze using logs or packet traces where security mechanisms fail or are weakened. Evaluate the effects on confidentiality, integrity, and authenticity.
2. Develop concrete hardening measures and explain why they effectively prevent the identified vulnerabilities.

The target length is slightly more than one DIN A4 page (~600-800 words for the two sections combined).

Let me write this properly in German, citing BSI TR-02102-2 and other relevant sources.

The two scenarios mentioned should be:
- A downgrade attack (POODLE-style: TLS → SSL 3.0)
- A handshake failure (self-signed certificate / weak cipher suite)

Let me structure this well.

For Section "Simulation eines Downgrade- oder Handshake-Fehlerfalls":
- Describe the POODLE-style downgrade scenario
- Show a simulated log/trace of the handshake
- Analyze where security fails
- Evaluate impact on CIA

For Section "Härtungsmaßnahmen":
- BSI TR-02102-2 recommendations
- TLS version restrictions
- Cipher suite hardening
- Certificate validation
- HSTS/HPKP

Let me write this now.
</thinking>

***

## Simulation eines Downgrade-Angriffs und Handshake-Fehlerfalls

### Szenario: POODLE-Downgrade (Client A ↔ Server S1, TLS 1.0 + SSL 3.0 aktiv)

Der POODLE-Angriff (*Padding Oracle On Downgraded Legacy Encryption*, CVE-2014-3566) zeigt exemplarisch, wie ein Angreifer in der Mitte (Mallory) einen modernen TLS-Verbindungsversuch systematisch auf das unsichere Protokoll SSL 3.0 herabstuft. Voraussetzung ist, dass sowohl Client als auch Server SSL 3.0 als Fallback unterstützen und der sogenannte „Downgrade Dance" nicht durch TLS_FALLBACK_SCSV (RFC 7507) blockiert wird. [a10networks](https://www.a10networks.com/de/blog/cve-2014-3566-beast-poodle-or-dancing-beast/)

**Schrittweiser Verbindungsaufbau mit simulierten Log-Auszügen:**

```
[Client→Server]  ClientHello  TLS 1.2  Cipher: TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
[Mallory]        TCP-RST → erzwingt Verbindungsabbruch
[Client→Server]  ClientHello  TLS 1.1  (Retry-Downgrade)
[Mallory]        TCP-RST → erneuter Abbruch
[Client→Server]  ClientHello  TLS 1.0  (weiterer Downgrade)
[Mallory]        TCP-RST
[Client→Server]  ClientHello  SSL 3.0  Cipher: SSL_RSA_WITH_RC4_128_MD5
[Server]         ServerHello  SSL 3.0  → Handshake erfolgreich
[Server]         Certificate  (RSA 1024 bit, SHA-1-signiert)
[Client]         ClientKeyExchange, ChangeCipherSpec, Finished
[Server]         ChangeCipherSpec, Finished
[Mallory]        Padding-Oracle-Angriff auf CBC-Blöcke → Session-Cookie lesbar
```

In Schritt 4 hat Mallory das Protokoll durch gezielte TCP-Resets auf SSL 3.0 erzwungen. Da SSL 3.0 im CBC-Modus keine kryptographisch valide Padding-Überprüfung kennt, lässt sich das Padding als Orakel nutzen: Mallory sendet manipulierte verschlüsselte Blöcke und leitet aus den differenziellen Serverantworten (Fehler vs. kein Fehler) iterativ den Klartextinhalt ab. Nach durchschnittlich 256 Anfragen pro Byte ist ein Session-Cookie vollständig rekonstruierbar. [thomas-krenn](https://www.thomas-krenn.com/de/wiki/SSL_3.0_POODLE_Attack)

**Zweites Szenario: Handshake-Fehler durch selbstsigniertes Zertifikat (Client B ↔ Server S2)**

```
[Client B→S2]   ClientHello  TLS 1.3
[S2]            ServerHello, Certificate (self-signed, CN=internal.example.local)
[Client B]      CERTIFICATE_UNKNOWN (Alert Level: fatal, Description: 46)
                → Verbindung abgebrochen, da Zertifikat nicht in Trust Store
```

Der Client bricht den Handshake nach der Zertifikatsprüfung ab, da die Zertifikatskette nicht zu einer vertrauenswürdigen Root-CA führt. Ohne gültige Signatur einer CA kann keine Authentizität des Servers sichergestellt werden – ein Man-in-the-Middle könnte sein eigenes selbstsigniertes Zertifikat einschleusen, ohne dass der Client es von einem legitimen unterscheiden kann.

### Bewertung der Auswirkungen auf Schutzziele

| Schutzziel | Szenario 1 (POODLE-Downgrade) | Szenario 2 (Selbstsigniertes Zertifikat) |
|---|---|---|
| **Vertraulichkeit** | ✗ verletzt – Klartextrekonstruktion via Padding-Oracle | ✗ gefährdet – kein vertrauenswürdiger verschlüsselter Kanal ohne Zertifikatsprüfung |
| **Integrität** | ✗ verletzt – manipulierte CBC-Blöcke werden akzeptiert | ~ partiell – TLS-MAC schützt, aber Kanalaufbau unsicher |
| **Authentizität** | ✗ verletzt – Server-Identität kann nicht verifiziert werden (kein SCSV-Schutz) | ✗ verletzt – keine vertrauenswürdige CA-Bindung, Impersonation möglich |

***

## Härtungsmaßnahmen

### 1. TLS-Versionsbeschränkung

Das BSI schreibt in der **TR-02102-2 (Version 2026-01)** verbindlich vor, dass ausschließlich TLS 1.2 und TLS 1.3 eingesetzt werden sollen. SSL 2.0, SSL 3.0 und TLS 1.0 sind vollständig zu deaktivieren, TLS 1.1 wird nicht mehr empfohlen. In OpenSSL und nginx lässt sich dies explizit konfigurieren: [bsi.bund](https://www.bsi.bund.de/DE/Themen/Unternehmen-und-Organisationen/Standards-und-Zertifizierung/Technische-Richtlinien/TR-nach-Thema-sortiert/tr02102/tr02102_node.html)

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
```

Durch die Deaktivierung aller Legacy-Protokolle entfällt die Angriffsfläche für Downgrade-Angriffe wie POODLE oder BEAST grundlegend, da kein Fallback mehr möglich ist. Ergänzend **muss TLS_FALLBACK_SCSV** (RFC 7507) implementiert werden: Erkennt der Server diesen Signaling Cipher Suite Value in einem Downgrade-Versuch, bricht er den Handshake mit einem `inappropriate_fallback`-Alert ab. [thomas-krenn](https://www.thomas-krenn.com/de/wiki/SSL_3.0_POODLE_Attack)

### 2. Sichere Cipher Suites

Schwache Algorithmen wie RC4, DES/3DES, Export-Cipher oder MD5-basierte MACs sind vollständig zu entfernen. Gemäß BSI TR-02102-2 sind ausschließlich Cipher Suites mit **Perfect Forward Secrecy (PFS)** zu verwenden, da nur dann kompromittierte Langzeitschlüssel keine vergangenen Sessions entschlüsselbar machen. Empfohlene Suiten für TLS 1.2: [epflicht.ulb.uni-bonn](https://epflicht.ulb.uni-bonn.de/download/pdf/594088?originalFilename=true)

```
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
TLS_DHE_RSA_WITH_AES_256_GCM_SHA384
```

Für TLS 1.3 sind die Cipher Suites im Standard bereits auf sichere AEAD-Verfahren reduziert; RC4, CBC und statische RSA-Schlüsselübertragung sind protokollseitig nicht mehr möglich. [der-windows-papst](https://www.der-windows-papst.de/2024/06/24/recommended-cipher-suites-2025/)

### 3. Zertifikatsvalidierung und Vertrauensketten

Selbstsignierte Zertifikate sind in produktiven Umgebungen unzulässig. Stattdessen sind Zertifikate einer anerkannten Certificate Authority (CA) einzusetzen. Zur Absicherung der Vertrauenskette empfiehlt das BSI:

- **Certificate Transparency (CT):** Alle ausgestellten Zertifikate werden in öffentliche, append-only Logs eingetragen. Clients können prüfen, ob ein präsentiertes Zertifikat dort gelistet ist – unautorisierte Ausstellungen werden erkennbar.
- **OCSP Stapling:** Der Server liefert den aktuellen Sperrstatus seines Zertifikats direkt im TLS-Handshake mit, ohne dass der Client einen separaten OCSP-Request senden muss. Dies verhindert, dass ein Angreifer OCSP-Anfragen blockiert und ein bereits widerrufenes Zertifikat weiter nutzbar bleibt.
- **CAA-DNS-Einträge:** Certification Authority Authorization Records begrenzen auf DNS-Ebene, welche CAs überhaupt Zertifikate für eine Domain ausstellen dürfen.

### 4. HTTP Strict Transport Security (HSTS)

Der Server teilt dem Browser über den `Strict-Transport-Security`-Header mit, dass die Domain ausschließlich über HTTPS erreichbar ist. Auch bei einem aktiven Downgrade-Versuch baut der Browser keine unverschlüsselte HTTP-Verbindung auf:

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

Durch Aufnahme in die HSTS-Preload-Liste wird dieser Schutz bereits beim ersten Verbindungsversuch greifen, ohne dass eine initiale HTTP-Anfrage nötig ist.

### 5. Regelmäßige Überprüfung und Patch-Management

Da neue Schwachstellen kontinuierlich entdeckt werden (vgl. CVE-Datenbank des MITRE), ist eine **regelmäßige Überprüfung der TLS-Konfiguration** zwingend erforderlich. Werkzeuge wie `testssl.sh` oder der Qualys SSL Labs Server Test erlauben eine automatisierte Analyse der aktiven Protokollversionen, Cipher Suites und Zertifikatseigenschaften. Das BSI macht die TR-02102-2 für Bundesbehörden gemäß § 8 Abs. 1 BSIG verbindlich und veröffentlicht regelmäßig aktualisierte Versionen der Richtlinie – zuletzt im Januar 2026. [kes-informationssicherheit](https://www.kes-informationssicherheit.de/print/titelthema-malware-trends-und-abwehr-2018/technische-richtlinien-des-bsi-zur-kryptografie/)