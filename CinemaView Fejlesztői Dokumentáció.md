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
- `JWT_ISSUER`: Tetszőleges string, ez kerül bele a JWT token `iss` mezejébe.
- `REFRESH_TOKEN_EXPIRATION`: A refresh JWT token lejárati ideje, percben. (Alapból 1 év; 525960 perc)
- `ACCESS_TOKEN_EXPIRATION`: Az access JWT token lejárati ideje, percben. (Alapból 5 perc)
- `UPLOAD_DIRECTORY`: A mappa ahova a feltöltött film boritok, bannerek kerulnek.
- `PORT`: A port amin a backend kiszolgálja a lekéréseket.
- `BASE_URL`: A külső alap URL, ez alapján generálja a feltöltött kepék elérési útvonalait.
- `TOTP_ISSUER`: Tetszőleges string, ez kerül bele a TOTP tokenbe.
- `ENV`: A környezet ahol fut a backend, a kettő lehetséges értéke `dev`, `prod`

### Endpointok
Az endpointok nincsenek ebben a dokumentumban leírva, mivel magától generál a backend egy OpenAPI v3 schemát a backend, a validációs szabályok segítségével. A Swagger oldal elérhető a backendről, a `/docs` route alatt (például: ` http://localhost:8085/docs`), ha az `env` változó `dev`-re van állítva.

### CORS
A CORS szabalyok is az `env` valtozotol fuggenek, ha `dev` akkor minden origin-t enged. Ha `prod`-ra van állítva akkor csak a megadott regex-el matchelő origineket engedi. (Ami most `leventea.hu`-ra van állítva, mivel ott van egy test deployment). Ez a regex az `src/index.ts` fileban talalhato.

### Tesztelés
Tesztelésre [insomnia](https://insomnia.rest)-t használunk, amit lehetséges automatikusan is futtatni az `inso` CLI program segítségével. Jelenleg nincsen CI pipeline-ba integrálva.

Az insomnia fájl megtalálható a backend repository gyökerében, `Insomnia.json` néven, ezt közvetlen importálni lehet a programba.

### Mappa struktúra
A `src/` mappán belül található az összes forráskód állomány. A gyökerében minden olyan modul megtalálható ami közös a route-ok között. Routing a [`@fastify/autoload`](https://github.com/fastify/fastify-autoload) package segítségével van megoldva.

A `routes/` mappán belül minden Fastify plugin automatikus betöltésre kerul, es az endpointjaik a mappajuk neve alapjan lesz elerheto a REST API segitsegevel. Peldaul a `routes/auth/index.ts` állományban exportalt plugin összes endpointja a `/auth/` prefix-el lesz elérhető.

Az autoload nem szab meg semmilyen kikötéseket a fájlnevekkel kapcsolatban, de ebben a projektben ezeket a konvenciókat használjuk:
- `index.ts`: Az összes route, schemaval együtt
- `model.ts`: Az adatbázishoz való interakcióhoz használt függvények, nagyrészt queryk.
- `service.ts`: Az adatbázison felüli absztrakciók amelyek komplikáltabbak egy lekérésnél.
- `types.ts`: Minden schema és model amit a slonik (zod) és a [[#Validacio|Fastify validátora]] (TypeBox) használ, egyéb releváns típusokkal együtt.

Vannak olyan route-ok, ahol mas fájlok is találhatóak, például a `movies/` erőforrásban a feltöltessél foglalkozó modulok.

### Authentication
A backend egy egyszerűsített bearer tokenes rendszert használ, ahol bejelentkezéskor/regisztrációkor kiad egy refresh és egy access tokent. Általában a refresh token hosszabb ideig el mint az access token. 

A refresh token csak a felhasználó azonosító kódját tartalmazza (UUIDv4). Az access token tartalmaz mindent amire szüksége lehet a backendnek a felhasználóval való interakciókor, mint például az E-Mail címe és a jogköre. Ennek az egyik legnagyobb előnye mas rendszerekkel szemben (pl.: session cookiek), hogy nem kell minden lekéréskor az adatbázis alapján hitelesíteni a felhasználó jogkörét és adatait.

Az access token lejártakor a `POST /auth/refresh` endpoint segítségével tudunk újat kerni, feltételezve hogy a refresh tokenünk nem járt le. Refresh token lejártakor a felhasználónak újra be kell jelentkeznie.

A TOTP bekapcsolási flow-ja nem teljesen egyértelmű első ránézésre: 
1. `POST /auth/totp`: Vissza adja a közös eceteset és a URI-t amiből generálni lehet a QR kódot, amivel hozzá tudja adni a felhasználó bármilyen TOTP alkalmazáshoz (például Google Authenticator, Bitwarden)
2. `PUT /auth/totp`: Ebben a hívásban vissza kell küldeni a szervernek a felhasználó jelszavat, és az akkori érvényes TOTP kódot. Ezzel aktiválja a második faktort bejelentkezéskor. 

#### Az access tokenben található mezők, és értékeik
- `id`: A felhasználóhoz tartozó UUIDv4
- `type`: Érteke mindig `access`. (Refresh tokeneknél ez mindig `refresh`)
- `firstName`: A felhasználó által regisztrációkor megadott keresztnév.
- `lastName`: A felhasználó által regisztrációkor megadott vezetéknév.
- `role`: A felhasználó [[#Jogosultsagi szintek|jogosultsági szintje]].
- `totpEnabled`: `true`, ha a felhasználó bekapcsolta a TOTP kétfaktoros bejelentkezést.

### Authorization
Viszonylag egyszerűen kezeli a backend a jogosultsági szinteket. Minden 'szint' (kódban role-nak nevezzük), eléri az alatta levő szintnek az összes funkcionalitását.

#### Jogosultsági szintek
Csökkenő sorrendben, az elérhető endpointok száma alapján:
- Admin
- Manager
- Employee
- Customer
- Guest (nincs bejelentkezve)

### Validáció

### Ismert hibák

## Desktop

Az asztali alkalmazásba csak admin jogosultsággal lehetséges bejelentkezni.

### Stack

## Frontend

### Stack