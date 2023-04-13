A CinemaView Mozi Kezelő Csomag három részre osztható: Backend, Desktop (asztali), Frontend. A Backend kiszolgálja mind az asztali és a webes alkalmazást. Az asztali alkalmazás adminisztrációs feladatokra lett tervezve.

## Backend

### Stack
- Node
- Typescript
- Fastify
- Postgres

### Telepítés és futtatás
A projekt npm helyett pnpm-et használ, de a workflow-t ez csak annyiban változtatja hogy más a package lock file. Ajánlott az npm kerülése.

A backend futtatásához [Docker](https://www.docker.com/) és [Docker Compose](https://docs.docker.com/compose/install/) szükséges (ha egyben akarjuk kezelni az adatbázissal).

A `pnpm i` parancs sikeres futtatasa utan a kovetkezo npm scriptek elerhetoek:
- `pnpm run watch`: Automatikusan ujra inditja a projektet, file szerkeztese utan.
- `pnpm run build`: Sima build script, a compileolt fileok elerhetoek `build/` alatt.

### Adatbázis, migrációk
Az adatbázis migrációkat [flyway CLI](https://documentation.red-gate.com/fd/welcome-to-flyway-184127914.html) segitsegevel futtatjuk. Jelenleg ez nincs integralva az egyik Docker workflow-ba se, ezert manualisan kell futtatni.

A backend nem hasznal semmilyen ORM-et, helyette slonikkal folytatunk lekereseket az adatbazis fele, ami egy egyszerubb wrapper az eredeti `pg` package korul, tipusokkal kiegeszitve. Slonik lehetove teszi a string template queryk irasat, ami sokkal kenyelmesebb mintha prepared queryket kene kezzel irnunk (persze a hatterben meg igy is prepared queryket hasznal). 

Teszt es fejlesztoi kornyezetekben az alabbi parancsal lehet egyszeruen elinditani egy postgres instance-t.
```bash
docker run -p 5432:5432 -e POSTGRES_USER=vizsgaremek -e POSTGRES_PASSWORD=vizsgaremek -d --name postgres postgres:15.1
```

Az osszes migracio futtatasa az alabbi parancsal lehetseges, feltetelezve hogy a projekt gyokermappaja a jelenlegi path, es `vizsgaremek` mind az adatbazis neve, a felhasznalonev es a jelszo.
```bash
docker run --network=host -v $PWD/migrations:/sql --rm flyway/flyway:9.8.1 -user=vizsgaremek -password=vizsgaremek -url="jdbc:postgresql://localhost:5432/vizsgaremek" -locations=filesystem:/sql migrate
```

### Konfiguracio
Az osszes ertekre talalhato pelda a backend repo gyokereben talalhato `.env.example` fileban.

Az alabbi valtozokat lehet megadni `.env` fileban vagy az adott operacios rendszer kornyezeti valtozoiban:
- `POSTGRES_CONN_STR`: Standard Postgres csatlakozasi URI, [tovabbi dokumentacio itt talalhato](https://www.postgresql.org/docs/current/libpq-connect.html#LIBPQ-CONNSTRING)
- `POSTGRES_PORT`: Ugyan annak kell lennie mint amit megadtunk a `POSTGRES_CONN_STR`-ben, ezt az erteket a Docker Compose hasznalja.
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

Az access tokenben talalhato mezok, es az ertekeik:
- `id`: A felhasznalohoz tartozo UUIDv4
- `type`: Erteke mindig `access`. (Refresh tokeneknel ez mindig `refresh`)
- `firstName`: A felhasznalo altal regisztraciokor megadott keresztnev.
- `lastName`: A felhasznalo altal regisztraciokor megadott vezeteknev.
- `role`: A felhasznalo [[#Authorization|jogosultsagi szintje]].
- `totpEnabled`: `true`, ha a felhasznalo bekapcsolta a TOTP ketfaktoros bejelentkezest.

### Authorization
Viszonylag egyszeruen kezeli a backend a jogosultsagi szinteket. Minden 'szint' (kodban role-nak nevezzuk), eleri az alatta levo szintnek az osszes funkcionalitasat.

A jogosultsagi szintek, csokkeno sorrendben, az elerheto endpointok merteke alapjan:
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