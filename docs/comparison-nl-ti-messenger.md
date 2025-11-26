# Vergelijking: Nederlandse Healthcare Messaging vs. Duitse TI-Messenger

Dit document vergelijkt de Nederlandse aanpak voor instant messaging in de zorg (op basis van de NUTS/mCSD-specificatie) met de Duitse TI-Messenger standaard van gematik.

## Samenvatting

| Aspect          | Nederlandse Aanpak                | Duitse TI-Messenger                    |
|-----------------|-----------------------------------|----------------------------------------|
| **Protocol**    | Matrix                            | Matrix                                 |
| **Directory**   | Gedistribueerde mCSD endpoints    | Centrale VZD-FHIR-Directory            |
| **Vertrouwen**  | mCSD-verificatie + UZI/URA 3PIDs  | Centrale IDP + smartcards (eHBA/SMC-B) |
| **Federatie**   | Open federatie via mCSD-discovery | Gesloten federatie via federatielijst  |
| **Adressering** | URA→mCSD→Matrix ID                | TelematikID→VZD→Matrix ID              |

---

## 1. Matrix Protocol Gebruik

### Nederlandse Aanpak

- **Provider-gehoste homeservers**: Zorgaanbieders kiezen een leverancier die een (multi-tenant) homeserver host
- **Organisatie kan wisselen**: Via het migratieprotocol kan een organisatie van provider wisselen met behoud van zorgcontexten
- **Volledige federatie**: Standaard Matrix Server-Server API voor cross-organisatie communicatie
- **Geen centrale proxy**: Directe federatie tussen homeservers
- **Homeserver adres via mCSD**: Het homeserver adres wordt gepubliceerd in de mCSD van de organisatie

### Duitse TI-Messenger

- **Homeserver per provider**: Vergelijkbaar model met onafhankelijke homeservers
- **Messenger-Proxy**: Alle communicatie loopt via een proxy die autorisatie controleert
- **Gesloten federatie**: Alleen communicatie met goedgekeurde partners op de federatielijst
- **BSI-accreditatie**: Alle servers moeten geaccrediteerd zijn en in Duitsland gehost

### Vergelijking

| Eigenschap | NL | DE |
|------------|----|----|
| Homeserver model | Multi-tenant per leverancier | Per provider |
| Directe federatie | Ja | Nee (via proxy) |
| Expliciete federatielijst | Nee | Ja |
| Centrale controle | Minimaal | Significant |
| Server locatie-eis | Nee | Ja (Duitsland) |
| Leverancier wisselen | Ja (migratieprotocol) | Onbekend |

---

## 2. Architectuur & Structuur

### Nederlandse Aanpak

```
┌─────────────────────────────────────────────────────────────┐
│                    Gedistribueerd Model                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────────────────────┐   ┌──────────────┐        │
│   │ Leverancier X (multi-tenant)│   │ Leverancier Y│        │
│   │ ┌─────────────────────────┐ │   │ ┌──────────┐ │        │
│   │ │      Homeserver         │◄────┼─►Homeserver│ │ Matrix │
│   │ │  (Org A, Org B, ...)    │ │   │ │ (Org C)  │ │ Feder. │
│   │ └─────────────────────────┘ │   │ └──────────┘ │        │
│   └─────────────────────────────┘   └──────────────┘        │
│                                                             │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│   │ mCSD Org A   │  │ mCSD Org B   │  │ mCSD Org C   │      │
│   │ (homeserver: │  │ (homeserver: │  │ (homeserver: │      │
│   │  lev-x.nl)   │  │  lev-x.nl)   │  │  lev-y.nl)   │      │
│   └──────────────┘  └──────────────┘  └──────────────┘      │
│            ▲              ▲                 ▲               │
│            │   Endpoint Discovery           │               │
│            └──────────────┬─────────────────┘               │
│                           ▼                                 │
│              ┌──────────────┐                               │
│              │    LRZA      │  Nationaal Register           │
│              │  (toekomst)  │  (alleen URLs naar mCSD)      │
│              │      of      │                               │
│              │NUTS Discovery│  ← huidige interim oplossing  │
│              │  (interim)   │                               │
│              └──────────────┘                               │
└─────────────────────────────────────────────────────────────┘
```

**Kerncomponenten:**
- **Matrix Homeserver**: Multi-tenant, gehost door leverancier (meerdere organisaties per homeserver)
- **mCSD Endpoint**: Per organisatie (FHIR-gebaseerde directory, bevat homeserver adres)
- **LRZA**: Nationaal register voor mCSD endpoint locaties (toekomst)
- **NUTS Discovery**: Interim oplossing voor endpoint discovery (totdat LRZA beschikbaar is)
- **IdentityServer (ma1sd)**: 3PID naar Matrix ID mapping
- **Migratieprotocol**: Organisaties kunnen van leverancier wisselen met behoud van rooms/spaces

### Duitse TI-Messenger

```
┌─────────────────────────────────────────────────────────────┐
│                    Gecentraliseerd Model                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│              ┌──────────────────────┐                       │
│              │   Centrale VZD-FHIR  │                       │
│              │     Directory        │                       │
│              └──────────┬───────────┘                       │
│                         │                                   │
│              ┌──────────┴───────────┐                       │
│              │   Registration       │                       │
│              │   Service            │                       │
│              └──────────┬───────────┘                       │
│                         │                                   │
│   ┌──────────────┐      │      ┌──────────────┐             │
│   │ Provider A   │      │      │ Provider B   │             │
│   │ ┌──────────┐ │      │      │ ┌──────────┐ │             │
│   │ │  Proxy   │◄───────┼───────►│  Proxy   │ │             │
│   │ └────┬─────┘ │             │ └────┬─────┘ │             │
│   │ ┌────▼─────┐ │             │ ┌────▼─────┐ │             │
│   │ │Homeserver│ │             │ │Homeserver│ │             │
│   │ └──────────┘ │             │ └──────────┘ │             │
│   └──────────────┘             └──────────────┘             │
│                                                             │
│              ┌──────────────────────┐                       │
│              │   Centrale IDP       │                       │
│              │   (smartcard auth)   │                       │
│              └──────────────────────┘                       │
└─────────────────────────────────────────────────────────────┘
```

**Kerncomponenten:**
- **Matrix Homeserver**: Per messenger-service provider
- **Messenger-Proxy**: Verplichte gateway voor alle communicatie
- **VZD-FHIR-Directory**: Centrale nationale directory
- **Registration Service**: Centrale registratie en validatie
- **Centrale IDP**: Smartcard-gebaseerde authenticatie

### Vergelijking

De Nederlandse aanpak is **fundamenteel gedecentraliseerd**: elke organisatie beheert haar eigen directory (mCSD) en alleen de *locatie* van deze directories wordt centraal geregistreerd.

De Duitse aanpak is **gecentraliseerd voor controle**: één centrale directory, één centrale authenticatieservice, en verplichte proxies die alle communicatie valideren.

---

## 3. Vertrouwen & Authenticatie

### Nederlandse Aanpak

**Vertrouwensmodel: Gedistribueerd via 3PIDs**

```
Zorgverlener authenticeert
         │
         ▼
    Organisatie IdP
         │
         ▼
┌────────────────────┐
│ UZI-nummer         │ ─── Persoonlijke zorgverlener ID
│ URA-nummer         │ ─── Organisatie ID
│ Rolcode (SNOMED)   │ ─── Functie/specialisme
└────────────────────┘
         │
         ▼
    Matrix Account
    (3PIDs opgeslagen)
```

**Mechanismen:**
- **UZI-nummer**: Unieke Zorgverlener Identificatie (persoonlijk)
- **URA-nummer**: Unieke Zorgaanbieder Registratie (organisatie)
- **Rolcode**: SNOMED CT code voor specialisme
- **mCSD**: Gedistribueerde directory met FHIR resources per organisatie
- **3PID lookup**: Reverse lookup van Matrix ID naar FHIR resources

**Verificatie:**
- Elk systeem kan onafhankelijk de 3PIDs verifiëren via mCSD queries
- Geen centrale autoriteit nodig voor runtime verificatie
- Vertrouwen gebaseerd op FHIR resources in mCSD (Practitioner, PractitionerRole)

### Duitse TI-Messenger

**Vertrouwensmodel: Centraal via Smartcards**

```
Zorgverlener authenticeert
         │
         ▼
    Smartcard Reader
         │
    ┌────┴────┐
    │         │
    ▼         ▼
  eHBA      SMC-B
(persoon) (organisatie)
    │         │
    └────┬────┘
         ▼
   Centrale IDP
         │
         ▼
┌────────────────────┐
│ TelematikID        │ ─── Unieke ID uit smartcard
│ ProfessionOID      │ ─── Beroepsgroep
└────────────────────┘
         │
         ▼
   JWT ID_TOKEN
         │
         ▼
  Registration Service
         │
         ▼
   Matrix Account
```

**Mechanismen:**
- **eHBA**: Elektronischer Heilberufsausweis (persoonlijke smartcard)
- **SMC-B**: Institutional Security Module Card (organisatie smartcard)
- **TelematikID**: Centrale identifier uit TI-infrastructuur
- **Centrale IDP**: Enige bron van authenticatie
- **JWT tokens**: Ondertekende tokens met identiteitsclaims

**Verificatie:**
- Alle verificatie loopt via centrale diensten
- Messenger-Proxy valideert elke actie
- Registration Service controleert federatielijst

### Vergelijking

| Aspect                  | NL                    | DE                |
|-------------------------|-----------------------|-------------------|
| Hardware vereist        | Nee                   | Ja (smartcards)   |
| Centrale authenticatie  | Nee (per org IdP)     | Ja (centrale IDP) |
| Runtime verificatie     | Gedistribueerd (mCSD) | Centraal (proxy)  |
| Single point of failure | Nee                   | Ja                |
| Offline authenticatie   | Mogelijk              | Niet mogelijk     |

---

## 4. Netwerk Opbouw & Federatie

### Nederlandse Aanpak

**Open Federatie Model**

- **Geen expliciete toestemming nodig**: Als je iemands Matrix ID kent, kun je communiceren
- **Discovery via mCSD**: Zoek praktijken/zorgverleners via FHIR queries
- **Endpoint discovery**: Organisaties publiceren hun mCSD endpoint locatie (nu via NUTS, straks via LRZA)
- **Organische groei**: Netwerk breidt uit naarmate meer organisaties mCSD publiceren

**Netwerk toetreding:**
1. Organisatie zet mCSD endpoint op
2. Registreert endpoint locatie (nu: NUTS Discovery, toekomst: LRZA)
3. Andere organisaties kunnen nu deze organisatie vinden
4. Directe federatie met alle andere Matrix homeservers

**Voordelen:**
- Lage toetredingsdrempel
- Geen centrale goedkeuring nodig
- Snelle uitrol mogelijk
- Geen vendor lock-in

**Nadelen:**
- Geen centrale controle over wie mag meedoen
- Vertrouwen volledig afhankelijk van 3PID verificatie

### Duitse TI-Messenger

**Gesloten Federatie Model**

- **Expliciete federatielijst**: Alleen communicatie met goedgekeurde partners
- **Centrale registratie**: Via Registration Service
- **Allowlist per organisatie**: Operators beheren wie mag communiceren
- **Gecontroleerde groei**: Alleen BSI/BfDI-geaccrediteerde diensten

**Netwerk toetreding:**
1. Provider vraagt accreditatie aan bij BSI
2. Implementeert tegen gematik referentie-implementatie
3. Registreert bij Registration Service
4. Wordt toegevoegd aan federatielijst
5. Andere providers kunnen nu federeren (als ze willen)

**Voordelen:**
- Sterke controle over netwerkdeelnemers
- Compliance met strenge Duitse regelgeving
- Duidelijke verantwoordelijkheid

**Nadelen:**
- Hoge toetredingsdrempel
- Langzame uitrol
- Centrale afhankelijkheid

### Vergelijking

```
Nederlandse Aanpak               Duitse Aanpak
─────────────────               ─────────────

  ┌───┐ ┌───┐ ┌───┐               ┌───────────┐
  │ A │─│ B │─│ C │               │Federatie- │
  └─┬─┘ └─┬─┘ └─┬─┘               │   lijst   │
    │     │     │                 └─────┬─────┘
    └──┬──┴──┬──┘                       │
       │     │                    ┌─────┼─────┐
     ┌─┴─┐ ┌─┴─┐                  │     │     │
     │ D │─│ E │                ┌─┴─┐ ┌─┴─┐ ┌─┴─┐
     └───┘ └───┘                │ A │ │ B │ │ C │
                                └───┘ └───┘ └───┘
  Mesh netwerk                  Hub-and-spoke
  (iedereen met iedereen)       (via centrale controle)
```

---

## 5. Adressering & Discovery

### Nederlandse Aanpak

**Gedistribueerde Discovery via mCSD**

```
┌─────────────────────────────────────────────────────────────┐
│                    Discovery Flow                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Ken URA-nummer van doelorganisatie                      │
│                      │                                      │
│                      ▼                                      │
│  2. Query endpoint registry: "Waar is mCSD voor URA-12345?" │
│     (nu: NUTS Discovery, toekomst: LRZA)                    │
│                      │                                      │
│                      ▼                                      │
│  3. Ontvang mCSD endpoint URL                               │
│                      │                                      │
│                      ▼                                      │
│  4. Query mCSD: GET /Practitioner?identifier=uzi|123        │
│                      │                                      │
│                      ▼                                      │
│  5. Ontvang FHIR Practitioner resource met Matrix ID        │
│     {                                                       │
│       "telecom": [{                                         │
│         "system": "other",                                  │
│         "value": "@dr.jansen:hospital-a.nl"                 │
│       }]                                                    │
│     }                                                       │
│                      │                                      │
│                      ▼                                      │
│  6. Start Matrix conversatie met @dr.jansen:hospital-a.nl   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**FHIR Resources voor adressering:**
- `Organization.telecom` → Homeserver adres
- `Practitioner.telecom` → Matrix user ID
- `PractitionerRole` → Koppeling persoon-organisatie-rol

**Voordelen:**
- Rijk datamodel (FHIR)
- Per-organisatie beheer van eigen data
- Flexibele queries (naam, specialisme, locatie, etc.)
- Geen centrale bottleneck

### Duitse TI-Messenger

**Centrale Discovery via VZD-FHIR-Directory**

```
┌─────────────────────────────────────────────────────────────┐
│                    Discovery Flow                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Authenticeer bij centrale IDP                           │
│                      │                                      │
│                      ▼                                      │
│  2. Query VZD: FHIRDirectorySearchAPI                       │
│     GET /Practitioner?name=Müller&address-city=Berlin       │
│                      │                                      │
│                      ▼                                      │
│  3. Ontvang zoekresultaten met TI-Messenger adressen        │
│                      │                                      │
│                      ▼                                      │
│  4. Proxy valideert: Is doelserver op federatielijst?       │
│                      │                                      │
│                      ▼                                      │
│  5. Proxy valideert: Staat contact op allowlist?            │
│                      │                                      │
│                      ▼                                      │
│  6. Communicatie toegestaan → start Matrix conversatie      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**VZD APIs:**
- `FHIRDirectorySearchAPI` → Zoeken naar organisaties/personen
- `FHIRDirectoryTIMProviderAPI` → Federatielijst beheer
- `FHIRDirectoryOwnerAPI` → Eigen gegevens beheren

**Voordelen:**
- Eén centrale plek voor alle zorgverleners
- Consistente data (centraal beheerd)
- Eenvoudiger zoeken (één API)

**Nadelen:**
- Single point of failure
- Centrale afhankelijkheid voor elke lookup
- Minder flexibiliteit in datamodel

### Vergelijking

| Aspect | NL (mCSD) | DE (VZD) |
|--------|-----------|----------|
| Type | Gedistribueerd | Centraal |
| Beheer | Per organisatie | Centraal + self-service |
| Queries | Naar individuele endpoints | Naar centrale directory |
| Consistentie | Variabel | Hoog |
| Beschikbaarheid | Hoog (geen SPOF) | Risico op SPOF |
| Schaalbaarheid | Onbeperkt | Afhankelijk van centrale capaciteit |

---

## 6. Zorgnetwerk & Patiëntcontext

Dit is een van de meest fundamentele verschillen tussen de twee aanpakken.

### Nederlandse Aanpak

**Gestructureerd zorgnetwerk via Matrix Spaces**

```
┌─────────────────────────────────────────────────────────────┐
│              Matrix Space = Zorgnetwerk                     │
│              (rond één cliënt/patiënt)                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Space: "Zorgnetwerk Patiënt X"                            │
│   ├── Room: Huisarts <-> Specialist                         │
│   ├── Room: Wijkverpleging coördinatie                      │
│   ├── Room: Mantelzorger communicatie                       │
│   └── Room: Patiënt betrokken gesprek                       │
│                                                             │
│   Space invite bevat: BSN (patiëntidentificatie)            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Kernprincipes:**
- **Space = Zorgnetwerk**: Elke Matrix Space representeert het complete zorgnetwerk rond één cliënt
- **BSN in invite**: Het uitnodigingsbericht voor een Space bevat het BSN, zodat ontvangende systemen de Space kunnen koppelen aan de juiste patiënt
- **Transparant netwerk**: Alle leden van een Space zien wie er betrokken is bij de zorg
- **Rooms binnen Space**: Individuele gesprekken (1:1, groep, met/zonder patiënt) leven binnen de Space context

**Netwerk opbouw:**

| Actor         | Hoe uitgenodigd           | Account model                             |
|---------------|---------------------------|-------------------------------------------|
| Professionals | Via mCSD lookup (UZI/URA) | Één account, gekoppeld aan organisatie    |
| Patiënten     | Direct door zorgverlener  | Meerdere accounts mogelijk (per platform) |
| Mantelzorgers | Direct door zorgverlener  | Meerdere accounts mogelijk (per platform) |

**Voordelen:**
- Volledig inzicht in wie betrokken is bij de zorg
- Patiëntcontext automatisch beschikbaar via Space membership
- Eenvoudige overdracht: nieuwe zorgverlener joint de Space
- Archivering: Space bevat complete communicatiegeschiedenis

### Duitse TI-Messenger

**Ad-hoc communicatie zonder structurele patiëntcontext**

```
┌─────────────────────────────────────────────────────────────┐
│              Ad-hoc Rooms (geen Space structuur)            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Room A: Dr. Müller <-> Dr. Schmidt                        │
│           (over patiënt X, context in berichten)            │
│                                                             │
│   Room B: Praxis ABC groepschat                             │
│           (interne communicatie)                            │
│                                                             │
│   Room C: Dr. Müller <-> Patiënt Y                          │
│           (via ePA integratie)                              │
│                                                             │
│   Geen overkoepelende Space per patiënt                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Kenmerken:**
- **Ad-hoc communicatie**: TI-Messenger focust op "sichere Ad-hoc-Kommunikation" - directe berichten tussen zorgverleners
- **Geen Space-hiërarchie**: Geen gebruik van Matrix Spaces om zorgnetwerken te structureren
- **Patiëntcontext in AIS/KIS**: Chats worden gearchiveerd en gekoppeld aan patiëntdossiers in het bronsysteem (AIS/KIS), niet in Matrix zelf
- **Groepschats voor coördinatie**: Behandelteams gebruiken groepschats, maar deze zijn niet structureel gekoppeld aan een patiënt op Matrix-niveau

**Patiënttoegang (TI-M ePA & Connect):**
- **TI-M ePA**: Patiënten krijgen messenger geïntegreerd in hun ePA (elektronisch patiëntendossier) app
- **TI-M Connect**: Integratie in bestaande apps (DiGA, patiëntportalen, videoconsult)
- **Eén identiteit**: Patiënt heeft één TI-Messenger identiteit via ePA, niet meerdere per platform

### Vergelijking

| Aspect                    | Nederlandse Aanpak           | Duitse TI-Messenger           |
|---------------------------|------------------------------|-------------------------------|
| **Structuur**             | Spaces per patiënt           | Ad-hoc rooms                  |
| **Patiëntcontext**        | In Space (BSN in invite)     | In bronsysteem (AIS/KIS)      |
| **Netwerk zichtbaarheid** | Alle Space-leden zien elkaar | Niet structureel beschikbaar  |
| **Patiënt accounts**      | Meerdere (per platform)      | Één (via ePA)                 |
| **Focus**                 | Zorgnetwerk coördinatie      | Ad-hoc berichten uitwisseling |

### Implicaties

**Nederlandse aanpak biedt:**
- Structurele transparantie over het zorgnetwerk
- Automatische context bij elk gesprek (via Space membership)
- Eenvoudige onboarding van nieuwe zorgverleners
- Complete audittrail per patiënt

**Duitse aanpak biedt:**
- Eenvoudiger model (alleen rooms, geen Spaces)
- Flexibiliteit in ad-hoc communicatie
- Integratie met bestaande patiëntdossier-systemen
- Één patiëntidentiteit (geen fragmentatie)

---

## 7. Conclusie & Aanbevelingen

### Fundamentele Ontwerpfilosofie

| Nederlandse Aanpak                                 | Duitse TI-Messenger                               |
|----------------------------------------------------|---------------------------------------------------|
| **Vertrouwen aan de rand**                         | **Vertrouwen in het centrum**                     |
| Organisaties beheren eigen identiteit en directory | Centrale diensten beheren identiteit en directory |
| Open federatie, verificatie via 3PIDs              | Gesloten federatie, verificatie via proxies       |
| Bottom-up netwerk groei                            | Top-down netwerk controle                         |

### Sterke Punten

**Nederlandse Aanpak:**
- Hoge veerkracht (geen centrale afhankelijkheden)
- Snelle adoptie mogelijk
- Data-soevereiniteit voor organisaties
- Flexibel en uitbreidbaar

**Duitse TI-Messenger:**
- Sterke compliance-garanties
- Eenvoudiger beheer en monitoring
- Consistente gebruikerservaring
- Duidelijke verantwoordelijkheid

### Overwegingen voor Harmonisatie

Beide systemen gebruiken Matrix en FHIR, wat interoperabiliteit in principe mogelijk maakt. Belangrijke aandachtspunten:

1. **Directory mapping**: NL mCSD ↔ DE VZD-FHIR
2. **Identity bridging**: UZI/URA ↔ TelematikID/eHBA
3. **Federatie**: Open (NL) vs. gesloten (DE) model
4. **Proxy transparantie**: DE proxies moeten NL verkeer accepteren

---

## Referenties

- [Matrix.org - Germany's national healthcare system adopts Matrix!](https://matrix.org/blog/2021/07/21/germany-s-national-healthcare-system-adopts-matrix/)
- [TI-Messenger - fachportal.gematik.de](https://fachportal.gematik.de/anwendungen/ti-messenger)
- [GitHub - gematik/api-ti-messenger](https://github.com/gematik/api-ti-messenger)
- [Element - TI-Messenger](https://element.io/en/matrix-in-germany/projects/ti-messenger)
- [TI-Messenger Advancing Secure Healthcare Communication (Matrix Conference 2024)](https://cfp.matrix.org/matrixconf2024/talk/JZCAFV/)
- [Adesso - The Matrix Protocol: A look inside the TI-Messenger](https://www.adesso.de/en/news/blog/the-matrix-protocol-a-look-inside-the-ti-messenger.jsp)
- [Healthcare Digital - TI-Messenger: Kommunikation für alle](https://www.healthcare-digital.de/ti-messenger-kommunikation-fuer-alle-a-59d476a4b39ac79da8e3e1260148fd02/)
- [Famedly - Anwendungsfälle: TI-Messenger im Klinikalltag](https://www.famedly.com/blog/anwendungsfaelle-ti-messenger)
