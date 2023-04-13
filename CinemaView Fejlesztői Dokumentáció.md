A CinemaView Mozi Kezelő Csomag három részre osztható: Backend, Desktop (asztali), Frontend. A Backend kiszolgálja mind az asztali és a webes alkalmazást. Az asztali alkalmazás adminisztrációs feladatokra lett tervezve.

## Backend

### Stack
- Node
- Typescript
- Fastify
- Postgres

### Telepítés és futtatás
A projekt npm helyett pnpm-et használ, de a workflow-t ez csak annyiban változtatja meg, hogy más a package lock file. Ajánlott az npm kerülése.

A backend futtatásához [Docker](https://www.docker.com/) és [Docker Compose](https://docs.docker.com/compose/install/) szükséges (ha egyben akarjuk kezelni az adatbázissal).

A `pnpm i` parancs sikeres futtatása után a következő npm scriptek válnak elérhetővé:
- `pnpm run watch`: Automatikusan újra indítja a projektet, ha azt érzékeli hogy az egyik forrás fájl megváltozott.
- `pnpm run build`: Sima build script, a fordított fájlok elérhetőek a `build/` mappa alatt.

### Adatbázis, migrációk
Az adatbázis migrációkat [flyway CLI](https://documentation.red-gate.com/fd/welcome-to-flyway-184127914.html) segítségével futtatjuk. Jelenleg ez nincs integrálva az egyik Docker workflow-ba se, ezért manuálisan kell futtatni.

A backend nem használ semmilyen ORM-et, helyette slonikkal folytatunk lekéréseket az adatbázis felé, ami egy egyszerűbb wrapper az eredeti `pg` package körül, típusokkal kiegészítve. Slonik lehetővé teszi a string template queryk írását, ami sokkal kényelmesebb mintha prepared statementeket kéne kézzel írnunk (persze a háttérben meg így is prepared statementeket használ). 

Tesztelési és fejlesztési környezetekben az alábbi paranccsal lehet egyszerűen elindítani egy Postgres instance-t.
```bash
docker run -p 5432:5432 -e POSTGRES_USER=vizsgaremek -e POSTGRES_PASSWORD=vizsgaremek -d --name postgres postgres:15.1
```

Az összes migráció futtatása az alábbi paranccsal lehetséges, feltételezve hogy a projekt gyökérmappája a jelenlegi munkakönyvtár, és `vizsgaremek` mind az adatbázis neve, a felhasználónév és a jelszó.
```bash
docker run --network=host -v $PWD/migrations:/sql --rm flyway/flyway:9.8.1 -user=vizsgaremek -password=vizsgaremek -url="jdbc:postgresql://localhost:5432/vizsgaremek" -locations=filesystem:/sql migrate
```

### Konfiguráció
Az összes értekre található példa a backend repo gyökerében található `.env.example` fájlban.

Az alábbi változókat lehet megadni `.env` fájlban vagy az adott operációs rendszer környezeti változóiban:
- `POSTGRES_CONN_STR`: Standard Postgres csatlakozási URI, [további dokumentáció itt található](https://www.postgresql.org/docs/current/libpq-connect.html#LIBPQ-CONNSTRING)
- `POSTGRES_PORT`: Ugyan annak kell lennie mint amit megadtunk a `POSTGRES_CONN_STR`-ben, ezt az érteket a Docker Compose használja.
- `JWT_SECRET`: A JWT token hash secretje
- `JWT_ISSUER`: Tetszoleges string, ez kerul bele a JWT token `iss` mezojebe.
- `REFRESH_TOKEN_EXPIRATION`: A refresh JWT token lejarati ideje, percben. (Alapbol 1 ev, 525960 perc)
- `ACCESS_TOKEN_EXPIRATION`: Az access JWT token lejarati ideje, percben. (Alapbol 5 perc)
- `UPLOAD_DIRECTORY`: A mappa ahova a feltoltott kep boritok, bannerek kerulnek.
- `PORT`: A port amin a backend figyel.
- `BASE_URL`: A kulso alap URL, ez alapjan generalja a feltoltott kepek eleresi utvonalat.
- `TOTP_ISSUER`: Tetszoleges string, ez kerul bele a TOTP tokenbe.
- `ENV`: A kornyezet ahol fut a backend, a ketto lehetseges erteke `dev`, `prod`

### Endpointok
Az endpointok nincsenek ebben a dokumentumban leirva, mivel magatol generalja a backend a validacios szabalyok segitsegevel. A Swagger oldal elerheto a backendrol, a `/docs` route alatt (pl ` http://localhost:8085/docs`), ha az `env` valtozo `dev`-re van allitva.

### CORS
A CORS szabalyok is az `env` valtozotol fuggenek, ha `dev` akkor minden origin-t enged. Ha `prod`-ra van allitva akkor csak a megadott regex-el matchelo origineket engedi. (Ami most `leventea.hu`-ra van allitva, mivel ott van egy test deployment). Ez a regex az `src/index.ts` fileban talalhato.

### Tesztelés
Tesztelésre [insomnia](https://insomnia.rest)-t használunk, amit lehetséges automatikusan is futtatni az `inso` CLI program segítségével. Jelenleg nincsen CI pipeline-ba integrálva.

Az insomnia file megtalalhato a backend repository gyokereben, `Insomnia.json` neven, ezt kozvetlen importalni lehet a programba.

### Mappa struktura
A `src/` mappan belul talalhato az osszes forraskod allomany. A gyokereben minden olyan modul megtalalhato ami kozos a route-ok kozott. Routing a [`@fastify/autoload`](https://github.com/fastify/fastify-autoload) package segitsegevel van megoldva. 

A `routes/` mappan belul minden Fastify plugin automatikus betoltesre kerul, es az endpointjaik a mappajuk neve alapjan lesz elerheto a REST API segitsegevel. Peldaul a `routes/auth/index.ts` allomanyban exportalt plugin osszes endpointja a `/auth/` prefix-el lesz elerheto.

Az autoload nem szab meg semmilyen kikoteseket a filenevekkel kapcsolatban, de ebben a projektben ezt a konvenciot hasznaljuk:
- `index.ts`: Az osszes route, schemaval egyutt
- `model.ts`: Az adatbazishoz valo interakciohoz hasznalt fuggvenyek, nagyreszt csak queryk.
- `service.ts`: Az adatbazison feluli absztrakciok amelyek komplikaltabbak egy lekeresnel.
- `types.ts`: Minden schema es model amit a slonik (zod) es a [[#Validacio|fastify validatora]] (TypeBox) hasznal, egyeb relevans tipusokkal egyutt.

Vannak olyan route-ok, ahol mas fileok is talalhatoak, peldalul a `movies/` eroforrasban a feltoltessel foglalkozo modulok.

### Authentication
A backend egy egyszerusitett bearer tokenes rendszert hasznal, ahol bejelentkezeskor/regisztraciokor kiad egy refresh es egy access tokent. Altalaban a refresh token hoszabb ideig el mint az access token. 

A refresh token csak a felhasznalo azonosito kodjat tartalmazza (UUIDv4). Az access token tartalmaz mindent amire szuksege lehet a backendnek a felhasznaloval valo interakciokor, mint peldaul az E-Mail cime es a jogkore. Ennek az egyik legnagyobb elonye mas rendszerekkel szemben (pl.: session cookiek), hogy nem kell minden lekereskor az adatbazis alapjan ellenorizni a felhasznalo jogkoret es adatait.

Az access token lejartakor a `POST /auth/refresh` endpoint segitsegevel tudunk ujat kerni, feltetelezve hogy a refresh tokenunk nem jart le. Refresh token lejartakor a felhasznalonak ujra be kell jelentkeznie.

A TOTP bekapcsolasi flow-ja nem teljesen egyertelmu elso ranezesre: 
1. `POST /auth/totp`: Vissza adja a kozos secretet es a URI-t amibol generalni lehet a QR kodot, amivel hozza tudja adni a felhasznalo barmilyen TOTP alkalmazashoz (peldaul Google Authenticator, Bitwarden)
2. `PUT /auth/totp`: Ebben a hivasban vissza kell kuldeni a szervernek a felhasznalo jelszavat, es az akkori ervenyes TOTP kodot. Ezzel aktivalja a masodik faktort bejelentkezeskor. 

#### Az access tokenben talalhato mezok, es az ertekeik
- `id`: A felhasznalohoz tartozo UUIDv4
- `type`: Erteke mindig `access`. (Refresh tokeneknel ez mindig `refresh`)
- `firstName`: A felhasznalo altal regisztraciokor megadott keresztnev.
- `lastName`: A felhasznalo altal regisztraciokor megadott vezeteknev.
- `role`: A felhasznalo [[#Jogosultsagi szintek|jogosultsagi szintje]].
- `totpEnabled`: `true`, ha a felhasznalo bekapcsolta a TOTP ketfaktoros bejelentkezest.

### Authorization
Viszonylag egyszeruen kezeli a backend a jogosultsagi szinteket. Minden 'szint' (kodban role-nak nevezzuk), eleri az alatta levo szintnek az osszes funkcionalitasat.

#### Jogosultsagi szintek
Csokkeno sorrendben, az elerheto endpointok merteke alapjan:
- Admin
- Manager
- Employee
- Customer
- Guest (nincs bejelentkezve)

### Validacio

### Ismert hibak

## Desktop

Az asztali alkalmazásba csak admin jogosultsággal legetséges bejelentkezni.

### Stack

## Frontend

### Stack