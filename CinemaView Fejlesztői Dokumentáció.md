A CinemaView Mozi Kezelő Csomag három részre osztható: Backend, Desktop (asztali), Frontend. A Backend kiszolgálja mind az asztali és a webes alkalmazást. Az asztali alkalmazás adminisztrációs feladatokra lett tervezve.

## Backend

### Stack
- Node
- Typescript
- Fastify
- Postgres

### Telepítés és futtatás
A projekt npm helyett pnpm-et használ, de a workflow-t ez csak annyiban változtatja hogy más a package lock file. Ajánlott az npm kerülése.

A backend futtatásához Docker és Docker Compose szükséges (ha egyben akarjuk kezelni az adatbázissal)

### Adatbázis, migrációk
Az adatbázis migrációkat [flyway CLI](https://documentation.red-gate.com/fd/welcome-to-flyway-184127914.html) segitsegevel futtatjuk. Jelenleg ez nincs integralva az egyik Docker workflow-ba se, ezert manualisan kell futtatni.

Teszt es fejlesztoi kornyezetekben az alabbi parancsal lehet egyszeruen elinditani egy postgres instance-t.
```bash
docker run -p 5432:5432 -e POSTGRES_USER=vizsgaremek -e POSTGRES_PASSWORD=vizsgaremek -d --name postgres postgres:15.1
```

Az osszes migracio futtatasa az alabbi parancsal lehetseges, feltetelezve hogy vizsgaremek mind az adatbazis, a felhasznalonev es a jelszo.
```bash
docker run --network=host -v $PWD/migrations:/sql --rm flyway/flyway:9.8.1 -user=vizsgaremek -password=vizsgaremek -url="jdbc:postgresql://localhost:5432/vizsgaremek" -locations=filesystem:/sql migrate
```

### Konfiguracio

### Ismert hibak

### Tesztelés
Tesztelésre [insomnia](https://insomnia.rest)-t használunk, amit lehetséges automatikusan is futtatni az `inso` CLI program segítségével. Jelenleg nincsen CI pipeline-ba integrálva.

Az insomnia file megtalalhato a backend repository gyokereben, `Insomnia.json` neven, ezt kozvetlen importalni lehet a programba.

## Desktop

Az asztali alkalmazásba csak admin jogosultsággal legetséges bejelentkezni.

### Stack

## Frontend

### Stack