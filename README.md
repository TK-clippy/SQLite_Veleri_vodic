# SQLite_Veleri_vodic
Praktični vodič za SQLite s primjerom baze fakulteta, SQL skriptom i uputama za rad u GUI alatu, terminalu ili SQLite Fiddleu. Uključuje pregled podataka, izradu kopije baze i izvoz u CSV.

# Kratki vodič za SQLite

U ovoj ćemo vježbi napraviti malu bazu fakulteta, pogledati podatke, spremiti kopiju baze i izvesti podatke u CSV. Ako imate instaliran GUI alat i `sqlite3`, možete raditi na računalu. Ako nemate, sve možete odraditi u pregledniku kroz [SQLite Fiddle](https://sqlite.org/fiddle/index.html), bez instalacije i administratorskih prava.

## 1. Priprema baze

Odaberite opciju koja vam više odgovara. U oba slučaja koristimo istu SQL skriptu.

### Opcija A: GUI alat

Otvorite **DBeaver i odaberite SQLite vezu** ili koristite DB Browser for SQLite. Pripazite da radite sa SQLite bazom jer ova skripta neće raditi u MariaDB-u. Zbog opcije `STRICT` potreban je SQLite 3.37.0 ili noviji.

Napravite novu bazu `fakultet.db` u mapi za vježbu i otvorite njezin SQL editor. U njega ćete zalijepiti skriptu koja slijedi.

### Opcija B: online u SQLite Fiddleu

1. Otvorite **[SQLite Fiddle](https://sqlite.org/fiddle/index.html)** i pričekajte učitavanje.
2. Obrišite početni tekst iz polja za unos i zalijepite cijelu SQL skriptu ispod.
3. Kliknite **Run**. Ako ste označili dio teksta, pokrenut će se samo taj dio, pa za prvo pokretanje nemojte ništa označavati.
4. Kad želite pokrenuti novi upit, zamijenite tekst u polju i ponovno kliknite **Run**. Baza ostaje sačuvana dok mijenjate tekst u editoru.

### Zajednička SQL skripta

Skriptu pokrenite jednom, dok je baza još prazna:

```sql
PRAGMA foreign_keys = ON;

CREATE TABLE student (
  student_id INTEGER PRIMARY KEY,
  ime        TEXT NOT NULL,
  prezime    TEXT NOT NULL,
  email      TEXT NOT NULL UNIQUE,
  created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
) STRICT;

CREATE TABLE kolegij (
  kolegij_id INTEGER PRIMARY KEY,
  naziv      TEXT NOT NULL UNIQUE,
  ects       INTEGER NOT NULL CHECK (ects BETWEEN 1 AND 60)
) STRICT;

CREATE TABLE upis (
  student_id  INTEGER NOT NULL REFERENCES student(student_id) ON DELETE CASCADE,
  kolegij_id  INTEGER NOT NULL REFERENCES kolegij(kolegij_id) ON DELETE RESTRICT,
  datum_upisa TEXT NOT NULL DEFAULT CURRENT_DATE,
  ocjena      INTEGER CHECK (ocjena BETWEEN 1 AND 5),
  PRIMARY KEY (student_id, kolegij_id)
) STRICT;

CREATE TABLE audit_log (
  audit_id   INTEGER PRIMARY KEY,
  tablica    TEXT NOT NULL,
  operacija  TEXT NOT NULL,
  kljuc      TEXT,
  stara_vrijednost TEXT,
  nova_vrijednost  TEXT,
  vrijeme    TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
) STRICT;

INSERT INTO student(ime, prezime, email) VALUES
 ('Ana','Horvat','ana.horvat@student.hr'),
 ('Marko','Kovačević','marko.kovacevic@student.hr'),
 ('Ivana','Babić','ivana.babic@student.hr');

INSERT INTO kolegij(naziv, ects) VALUES
 ('Baze podataka',6),('Upravljanje bazama podataka',5),('Programiranje',6);

INSERT INTO upis(student_id,kolegij_id,ocjena) VALUES
 (1,1,5),(1,2,4),(2,1,3),(3,2,NULL),(3,3,5);

CREATE INDEX idx_student_prezime ON student(prezime);

CREATE VIEW pregled_upisa AS
SELECT s.student_id, s.ime, s.prezime, k.naziv AS kolegij, u.ocjena
FROM student AS s
JOIN upis AS u ON u.student_id = s.student_id
JOIN kolegij AS k ON k.kolegij_id = u.kolegij_id;

CREATE TRIGGER trg_student_update
AFTER UPDATE OF prezime ON student
FOR EACH ROW
BEGIN
  INSERT INTO audit_log(tablica, operacija, kljuc, stara_vrijednost, nova_vrijednost)
  VALUES ('student','UPDATE', CAST(NEW.student_id AS TEXT), OLD.prezime, NEW.prezime);
END;
```

Sada imamo tri studenta, tri kolegija i pet upisa. Skripta je dodala i indeks za prezime, pogled koji povezuje podatke o upisima te okidač koji bilježi promjene prezimena. Provjerimo jesu li podaci tu:

```sql
SELECT * FROM pregled_upisa ORDER BY student_id, kolegij;
```

**Ako radite u GUI alatu**, potvrdite promjene gumbom **Commit** ako ih alat ne sprema automatski. Zatim zatvorite vezu s bazom i prijeđite u terminal.

**Ako radite u Fiddleu**, ostanite u istoj kartici preglednika. Prije zatvaranja stranice spremite bazu na računalo pomoću gumba **Download DB** u izborniku **Options** jer je baza u pregledniku privremena. Preuzetu datoteku možete nazvati `fakultet.db`, a kasnije je ponovno otvoriti gumbom **Load DB...**.

## 2. Demonstracija u terminalu ili Fiddleu

Sada ćemo pogledati podatke iz pripremljene baze, napraviti kopiju i promijeniti jednu ocjenu. Na kraju ćemo izvesti pregled upisa u CSV. Sve potrebne naredbe nalaze se ispod.

### Otvaranje i pregled

**Ako koristite terminal**, otvorite mapu u kojoj ste spremili `fakultet.db` i pokrenite:

```bash
sqlite3 fakultet.db
```

**Ako koristite Fiddle**, odmah krenite od sljedećeg bloka. Zalijepite naredbe u polje za unos i kliknite **Run**. Početnu skriptu više ne trebate pokretati.

Ako radite u terminalu, naredbe do `.quit` upisujete unutar programa `sqlite3`, uz oznaku `sqlite>`:

```text
PRAGMA foreign_keys = ON;
.databases
.tables
.schema student
.headers on
.mode column
.nullvalue NULL
SELECT * FROM pregled_upisa ORDER BY student_id, kolegij;
```

`.databases` pokazuje putanju otvorene baze, `.tables` tablice i poglede, a `.schema` strukturu. Dot-naredbe počinju točkom i **nemaju završni `;`**. SQL naredbe završavaju s `;`.

### Kopija i promjena podatka

```text
.backup 'kopija.db'
UPDATE upis SET ocjena = 2 WHERE student_id = 1 AND kolegij_id = 1;
SELECT * FROM pregled_upisa WHERE student_id = 1 ORDER BY kolegij;
```

Ani smo iz kolegija Baze podataka promijenili ocjenu na `2`. Budući da smo kopiju napravili prije promjene, u njoj je ocjena i dalje `5`. Ako ponavljate vježbu i želite sačuvati staru kopiju, novoj dajte drugo ime.

### Izvoz u CSV

**U terminalu:**

```text
.headers on
.mode csv
.once upisi.csv
SELECT * FROM pregled_upisa ORDER BY student_id, kolegij;
.mode column
.quit
```

Naredba `.once` sprema rezultat sljedećeg upita u datoteku. U CSV-u ćete zato vidjeti trenutačne podatke, uključujući Aninu ocjenu `2`. Za razliku od tog izvoza, `.backup` sprema cijelu bazu, zajedno s njezinom strukturom.

Nakon `.quit` ponovno ste u **Linux terminalu**. Provjerite datoteke i otvorite CSV:

```bash
ls -lh fakultet.db kopija.db upisi.csv
xdg-open upisi.csv
```

CSV možete otvoriti i u LibreOffice Calcu. Pri uvozu odaberite UTF-8 i zarez kao razdjelnik ako ih program ne prepozna automatski.

**Ako radite u Fiddleu**, kopija `kopija.db` ostaje unutar preglednika i neće se sama pojaviti u mapi za preuzimanja. Za CSV koristite ovaj blok koji rezultat prikazuje na ekranu:

```text
.headers on
.mode csv
SELECT * FROM pregled_upisa ORDER BY student_id, kolegij;
.mode column
```

Kopirajte prikazani CSV u tekstualnu datoteku i spremite je kao `upisi.csv`. Cijelu trenutačnu bazu možete preuzeti gumbom **Download DB**. Naredbe `ls`, `xdg-open` i `.quit` ovdje vam ne trebaju.

Više o radu u pregledniku možete pročitati na stranici [SQLite Fiddle](https://sqlite.org/fiddle/index.html), u dijelu **About**.
