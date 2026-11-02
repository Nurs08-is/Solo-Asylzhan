Markdown · README.md

# Esign-Util — RSA-2048 электрондық қолтаңба утилитасы

Java тілінде жазылған консольдік утилита: RSA-2048 кілт жұбын жасайды, файлға **SHA256withRSA** алгоритмімен қол қояды және қолтаңбаны тексереді. Кілттер **PEM** форматында сақталады.

> Оқу жобасы: «Комп жүйе қауіпсіздігі», ИП2408k, 11-тақырып.

## Мазмұны

1. [Мақсаты](#%D0%BC%D0%B0%D2%9B%D1%81%D0%B0%D1%82%D1%8B)
2. [Research: теория](#research-%D1%82%D0%B5%D0%BE%D1%80%D0%B8%D1%8F)
3. [Диаграммалар](#%D0%B4%D0%B8%D0%B0%D0%B3%D1%80%D0%B0%D0%BC%D0%BC%D0%B0%D0%BB%D0%B0%D1%80)
4. [Орнату](#%D0%BE%D1%80%D0%BD%D0%B0%D1%82%D1%83)
5. [Қолдану мысалы](#%D2%9B%D0%BE%D0%BB%D0%B4%D0%B0%D0%BD%D1%83-%D0%BC%D1%8B%D1%81%D0%B0%D0%BB%D1%8B)
6. [Жоба құрылымы](#%D0%B6%D0%BE%D0%B1%D0%B0-%D2%9B%D2%B1%D1%80%D1%8B%D0%BB%D1%8B%D0%BC%D1%8B)
7. [Тесттер](#%D1%82%D0%B5%D1%81%D1%82%D1%82%D0%B5%D1%80)
8. [Қауіпсіздік ескертпесі](#%D2%9B%D0%B0%D1%83%D1%96%D0%BF%D1%81%D1%96%D0%B7%D0%B4%D1%96%D0%BA-%D0%B5%D1%81%D0%BA%D0%B5%D1%80%D1%82%D0%BF%D0%B5%D1%81%D1%96)
9. [Лицензия](#%D0%BB%D0%B8%D1%86%D0%B5%D0%BD%D0%B7%D0%B8%D1%8F)

## Мақсаты

- Электрондық қолтаңбаның жұмыс принципін практикада көрсету.
- Тек стандартты Java криптография архитектурасын (JCA) қолдану: `KeyPairGenerator`, `Signature`, `KeyFactory`.
- Құжат өзгерсе, қолтаңба тексеруден өтпейтінін дәлелдеу.

| Команда | Әрекет |
| --- | --- |
| `keygen` | RSA-2048 кілт жұбын жасап, PEM файлдарға сақтайды |
| `sign` | Файлға жеке кілтпен қол қояды (`.sig` файл жасайды) |
| `verify` | Файл мен `.sig` қолтаңбасын ашық кілтпен тексереді |

## Research: теория

### Электрондық қолтаңба дегеніміз не?

Құжаттың хэшін жеке кілтпен өңдеу арқылы алынатын мән. Ол үш қасиетті береді:

- **Тұтастық** — құжат өзгерсе, қолтаңба сәйкес келмейді.
- **Авторлық** — тек жеке кілт иесі қол қоя алады.
- **Бас тарта алмау** — қол қойған адам кейін жоққа шығара алмайды.

Қолтаңба шифрлау емес: құжаттың құпиялылығын қорғамайды.

### Алгоритмдерді таңдау

| Параметр | Таңдау | Негіздеме |
| --- | --- | --- |
| Асимметриялық алгоритм | RSA | JCA-да стандартты, кең таралған |
| Кілт ұзындығы | 2048 бит | Қазіргі ең төменгі ұсынылатын деңгей (NIST); 1024 — әлсіз |
| Хэш | SHA-256 | MD5 және SHA-1 коллизияға осал |
| Қолтаңба схемасы | SHA256withRSA (PKCS#1 v1.5) | Талап бойынша; балама — RSASSA-PSS |
| Кездейсоқ сан | `SecureRandom` | Криптографиялық берік генератор |

### PEM форматы

PEM — DER байттарының Base64 түрі, тақырып жолдарымен:

```
-----BEGIN PRIVATE KEY-----   ← PKCS#8 (key.getEncoded())
-----BEGIN PUBLIC KEY-----    ← X.509 SubjectPublicKeyInfo
```

Оқығанда тақырыптарды алып тастап, Base64 декодтап, `KeyFactory` + `PKCS8EncodedKeySpec` / `X509EncodedKeySpec` арқылы кілт қалпына келтіріледі.

### Негізгі Java API

- `KeyPairGenerator.getInstance("RSA")` → `initialize(2048, new SecureRandom())` → `generateKeyPair()`
- `Signature.getInstance("SHA256withRSA")` → `initSign` / `initVerify` → `update` → `sign()` / `verify()`
- Үлкен файлдар буферлеп (`update` бірнеше рет) өңделеді.

## Диаграммалар

### 1. Қол қою және тексеру процесі

```mermaid
sequenceDiagram
    autonumber
    actor A as Қол қоюшы
    participant U as Esign-Util
    actor B as Тексеруші

    A->>U: keygen
    U-->>A: private.pem + public.pem
    A->>U: sign (document.pdf, private.pem)
    U->>U: SHA-256 хэш + RSA жеке кілтпен қол қою
    U-->>A: document.pdf.sig
    A->>B: document.pdf + .sig + public.pem
    B->>U: verify (document.pdf, .sig, public.pem)
    U->>U: хэшті есептеу + RSA ашық кілтпен тексеру
    alt Файл өзгермеген
        U-->>B: VALID
    else Файл өзгертілген / басқа кілт
        U-->>B: INVALID
    end
```

### 2. Жоба архитектурасы

```mermaid
flowchart LR
    CLI["Main (CLI)<br/>keygen / sign / verify"] --> KS[KeyService]
    CLI --> SS[SignService]
    KS --> PU[PemUtil]
    SS --> PU
    KS -->|KeyPairGenerator RSA-2048| JCA[(JCA: java.security)]
    SS -->|Signature SHA256withRSA| JCA
    PU -->|KeyFactory| JCA
    PU <--> FS[("PEM файлдар<br/>private.pem / public.pem")]
    SS <--> SIG[("Құжат және .sig файл")]
```

### 3. Қауіпсіздік шекарасы: не GitHub-қа түседі

```mermaid
flowchart TB
    subgraph LOCAL["Жергілікті компьютер (құпия)"]
        PRIV["private.pem"]
        ENV[".env"]
    end
    subgraph REPO["GitHub репозиторийі (ашық)"]
        SRC["src/main, src/test"]
        DOC["README.md, LICENSE"]
        GI[".gitignore"]
    end
    GI -.->|"*.pem, keys/, .env — бұғатталған"| LOCAL
    LOCAL x--x|"commit жасалмайды"| REPO
```

### 4. UML: Use Case диаграммасы

```mermaid
flowchart LR
    U(["Пайдаланушы<br/>(қол қоюшы)"])
    V(["Тексеруші"])
    subgraph SYS["Esign-Util"]
        UC1(["Кілт жұбын жасау"])
        UC2(["Файлға қол қою"])
        UC3(["Қолтаңбаны тексеру"])
        UC4(["PEM файлға сақтау"])
        UC5(["PEM файлдан оқу"])
    end
    U --- UC1
    U --- UC2
    V --- UC3
    UC1 -.->|include| UC4
    UC2 -.->|include| UC5
    UC3 -.->|include| UC5
```

### 5. UML: Class диаграммасы

```mermaid
classDiagram
    class Main {
        +main(args) void
        -printUsage() void
    }
    class KeyService {
        -KEY_SIZE : int = 2048
        +generateKeyPair() KeyPair
    }
    class PemUtil {
        +writePrivateKey(key, path) void
        +writePublicKey(key, path) void
        +readPrivateKey(path) PrivateKey
        +readPublicKey(path) PublicKey
    }
    class SignService {
        -ALGORITHM : String = SHA256withRSA
        +sign(file, privateKey) byte[]
        +verify(file, signature, publicKey) boolean
    }
    Main --> KeyService : uses
    Main --> SignService : uses
    Main --> PemUtil : uses
    KeyService ..> PemUtil : кілтті сақтау
    SignService ..> PemUtil : кілтті оқу
```

### 6. UML: Activity диаграммасы (тексеру алгоритмі)

```mermaid
flowchart TD
    S((Бастау)) --> A["Файл, .sig және public.pem беріледі"]
    A --> B{"Файлдар бар ма?"}
    B -- Жоқ --> E1["Қате: файл табылмады"]
    B -- Иә --> C["public.pem оқу (X.509)"]
    C --> D["Signature.initVerify(publicKey)"]
    D --> F["Файлды буфермен update() ету"]
    F --> G{"verify(.sig) нәтижесі"}
    G -- true --> H["VALID"]
    G -- false --> I["INVALID"]
    E1 --> Z((Соңы))
    H --> Z
    I --> Z
```

## Орнату

Талаптар: JDK 17+, Maven 3.8+ (немесе Gradle 8+).

```bash
git clone https://github.com/<username>/esign-util.git
cd esign-util
mvn clean package
```

Нәтиже: `target/esign-util.jar`.

## Қолдану мысалы

```bash
# 1. Кілт жұбын жасау
java -jar target/esign-util.jar keygen --out keys/
#  -> keys/private.pem, keys/public.pem

# 2. Құжатқа қол қою
java -jar target/esign-util.jar sign --key keys/private.pem --file document.pdf
#  -> document.pdf.sig

# 3. Қолтаңбаны тексеру
java -jar target/esign-util.jar verify --key keys/public.pem --file document.pdf --sig document.pdf.sig
#  -> VALID  (файл өзгертілсе: INVALID)
```

### Скриншот

![Демо](docs/screenshot.png)

## Жоба құрылымы

```
esign-util/
├── pom.xml
├── README.md
├── LICENSE
├── .gitignore
├── src/
│   ├── main/java/kz/esign/
│   │   ├── Main.java          # CLI
│   │   ├── KeyService.java    # кілт жасау
│   │   ├── PemUtil.java       # PEM оқу/жазу
│   │   └── SignService.java   # қол қою / тексеру
│   └── test/java/kz/esign/    # JUnit 5 тесттер (≥10)
└── .github/workflows/ci.yml   # бонус: GitHub Actions
```

## Тесттер

```bash
mvn test
```

JUnit 5, кемінде 10 тест:

1. Кілт ұзындығы 2048 бит
2. Екі генерация әртүрлі кілт береді
3. Private PEM `BEGIN PRIVATE KEY` тақырыбымен
4. Public PEM `BEGIN PUBLIC KEY` тақырыбымен
5. PEM жазып қайта оқығанда кілт сәйкес келеді
6. Дұрыс қолтаңба → `VALID`
7. Файл бір байтқа өзгертілсе → `INVALID`
8. Басқа кілт жұбымен тексеру → `INVALID`
9. Бүлінген қолтаңба → `INVALID`
10. Бос файлға қол қою және тексеру
11. Үлкен файл (5 МБ) өңделеді
12. Бар емес файл — түсінікті қате

## Қауіпсіздік ескертпесі

- **`private.pem` файлын ешқашан GitHub-қа жүктемеңіз!** `.gitignore` ішінде `*.pem`, `keys/`, `.env` болуы міндетті.
- Жеке кілт PKCS#8, ашық кілт X.509 форматында сақталады.
- Файл рұқсаттарын шектеңіз: `chmod 600 private.pem`.
- Оқу жобасы: жеке кілт шифрланбаған күйде сақталады. Нақты жүйеде пароль, KeyStore немесе HSM қолданылуы керек.
- Қолтаңба тек тұтастық пен авторлықты қамтамасыз етеді, файлды шифрламайды.
- Сертификатсыз ашық кілттің кімдікі екенін растау мүмкін емес (қараңыз: X.509, 12-тақырып).
- MD5 және SHA-1 қолданылмайды.

## Лицензия

MIT — қараңыз [LICENSE](LICENSE).

## Дереккөздер

- Oracle: Java Cryptography Architecture (JCA) Reference Guide
- NIST SP 800-57, FIPS 186-5
- RFC 8017 (PKCS #1), RFC 7468 (PEM)