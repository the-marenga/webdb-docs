# Node Konfiguration

## Grundsätzliche Erklärung

Um das hier genutzte Crawling zu verstehen, muss man sich zunächst überlegen, wie
man naiv Daten aus Webseiten extrahieren könnte. Nehmen wir als Beispiel eine
einfache Webseite, die nur Daten zu einem Film enthällt.

```html
<html>
  <body>
    <h1>Mission Impossible</h1>
    <article>Ein toller Film</article>
  </body>
</html>
```

Wir könnten nun einfach für jedem Wert, den wir extrahieren wollen, einen XPath
zuweisen, also z.B.

```yaml
# Extraction Config
Title: descendant::h1
Description: descendant::article
```

Damit nun die Werte zu extrahieren und z.B. als CSV zu exportieren funktioniert
damit super. Zudem können wir die Daten auch genau so in eine Datenbank
schreiben. Sobald wird allerdings versuchen mehr als dieses simple Beispiel zu
extrahieren, wird es problematisch. Diese Probleme sind hauptsächlich:
**Datenbank Updates**, **Referenzierende Listen** und **Fehlende Listen Elemente**

### Datenbank Updates

Es gibt zwei Arten von Datenbank Updates: Crawling Interne & Iterative

**Crawling Interne** Updates liegen vor, wenn wir zwei Seiten haben, die auf die
selbe Zeile in der Datenbank verweisen sollen. Bei IMDB gibt es zum Beispiel nicht nur
die Hauptseite, sondern auch extra Seiten, die Details zu besonderen Aspekten,
wie dem Soundtrack beinhalten. Wir könnten für jede Seite eine neue Tabelle
anlegen, aber für eine sinvollere Datenbank Struktur ist es erforderlich einen
bestehenden Eintrag zu modifizieren, also fehlende Werte zu ergänzen, oder
zumindest einen bestehenden Eintrag zu referenzieren.

**Iterative Updates** treten auf, wenn wir den selben Crawling durchlauf
mehrfach starten. Dabei würden wir die selbe URL also mehrfach auswerten. Was
machen wir also nun mit den neuen Daten? Wir könnten sie einfach als neue
Zeile in die Datenbank schreiben und Duplikate später manuell hrausfiltern. Was
wir allerdings wirklich wollen ist eine Möglichkeit die alte Zeile wieder zu
finden, zu modifizieren und bei bedarf zu auf Veränderungen zu vergleichen.

In beiden Fällen müssen wir irgendwie bestehende Einträge wiederfinden. Die
Naive idee wäre es einfach die URL der Seite als Primary Key festzulegen. Damit
hätten wir eine garantierete möglichkeit alte einträge zu suchen. Diese Art der
Identifizierung reicht und allerdings noch nicht ganze aus.

### Referenzierende Listen

Webseiten beinhalten oft nicht nur einzelne Wert, sondern oft Listen:

```html
<html>
  <body>
    <h1>Mission Impossible</h1>
    <article>Ein toller Film</article>
    <div>
      <a href="https://filminfos.de/actor/a32342">
        <h2>Tom Cruise</h2>
      </a>
      <a href="https://filminfos.de/actor/a714241">
        <h2>Jon Voight</h2>
      </a>
    </div>
  </body>
</html>
```

Datenbanken wie Postgres beinhalten extra Array Datentypen, um diese Werte zu
speichern. Wir könnten hier mit einem Xpath wie "descendant::h2" also einfach
die Namen extrahieren und direkt mit in die Movies Tabelle schreiben. In der
Datenbank hätten wir für diesen Film also:

```
| id | Url | Title              | Description       | Actors                 |
|----|-----|--------------------|-------------------|------------------------|
| 1  | ... | Mission Impossible | Ein toller Film   | Tom Cruise, Jon Voight |

```

Wir haben allerdings wieder ein Problem und zwar das Crawling-Interne Update
Problem von zuvor. Wenn wir zum Beispiel jetzt die Seite der Actors crawlen wollen, um
Details zu den Actors zu extrahieren, wie würden wir diese nur anhand der Namen
zuordnen? Wenn wir wie zuvor vorgeschlagen die URl als Primary key nehmen,
müssten wir hier statdessen die URLs extrahieren:

```
| id | Url | Title              | Description       | Actors                                                                 |
|----|-----|--------------------|-------------------|------------------------------------------------------------------------|
| 1  | ... | Mission Impossible | Ein toller Film   | https://filminfos.de/actor/a32342, https://filminfos.de/actor/a714241 |
```

Eine Datenbank so zu strukturieren ist verschwenderisch und fehleranfällig. Was
wir lieber haben würden ist eine ID, die die betreffende Zeilen des Actors direkt
referenziert. Wir müssen also irgendwie konfigurieren, dass diese Werte auf die
Actors tabelle Verweisen. Um die wirklichen IDs zu finden, könnten wir z.B.
folgende funktion nutzen:

```js
function urlToDatabaseId(url, referencedTable) {
  let id = db.fetch(`SELECT id FROM ${referencedTable} WHERE url = ${url}`);
  if (id != null) {
    return id
  }
  return db.fetch(`INSERT INTO ${referencedTable} (url) VALUES (${url})`);
}
```

Wir hätten damit zum Beispiel:

```
#Movies
| id | Url | Title              | Description       | Actors  |
|----|-----|--------------------|-------------------|---------|
| 1  | ... | Mission Impossible | Ein toller Film   | 1,2     |

# Actors
| id | Url |
|----|-----|
| 1  | ... |
| 2  | ... |
```

Wenn wir die MovieId und die ActorId haben, könnte man sich auch überlegen die
Relation automatisch in eine Many2Many tabelle zu schreiben. Man könnte auch
zusätzlich extrahierte Daten, wie den Namen des Actors, oder die Rolle anhand
der IDs direkt in diese Tabellen schreiben. Das ganze hat nur einen finales
Problem.

##Fehlende Listen Elemente

Stellen wir uns vor, wir haben die folgenden Daten extrahiert:

```yaml
Title: "Mission Impossible"
ActorNames: ["Tom Cruise", "Jon Voight"]
ActorRoles: ["Ethan Hunt", "Jim Phelps"]
ActorUrls: ["https://filminfos.de/actor/a32342", "https://filminfos.de/actor/a714241"]
```

Wie zuvor beschrieben, könnten wir mit der richtigen Konfiguration mit diesen
Daten einen Eintrag in der Actor Tabelle und einer Many2Many tabelle finden und
die Daten dort korrekt einfügen:

```
#Movie
| id | Url | Title              | Description       |
|----|-----|--------------------|-------------------|
| 1  | ... | Mission Impossible | Ein toller Film   |

#Actor2Movie
| MovieID | ActorID | Role       |
|---------|---------|------------|
|    1    |    1    | Ethan Hunt |
|    1    |    2    | Jim Phelps |

# Actor
| id | Url | Name       |
|----|-----|------------|
| 1  | ... | Tom Cruise |
| 2  | ... | Jon Voight |
```

Wir verlassen uns dabei allerdings darauf, dass dass unsere erste extrahierte
Actor URL auch zu dem ersten Namen und der ersten Rolle gehört. Schauen wir uns
allerdings dieses Beispiel an:

```yaml
Title: "Mission Impossible"
ActorNames: ["Tom Cruise", "Jon Voight"]
ActorRoles: ["Ethan Hunt"]
ActorUrls: ["https://filminfos.de/actor/a32342", "https://filminfos.de/actor/a714241"]
```

In diesem Fall haben wir keine Möglichkeit mehr die Rollen korrekt zu zu ordnen.
Noch schwieriger wird es hier:

```yaml
Title: "Mission Impossible"
ActorNames: ["Tom Cruise", "Jon Voight"]
ActorRoles: ["Ethan Hunt", "Jim Phelps"]
ActorUrls: ["https://filminfos.de/actor/a32342"]
```

Wir wissen nicht zu wem die URL gehört, also können wir keine Actor Ergebnisse
mehr sicher nutzen, ohne die Gefahr einzugehen die Daten falsch zu zu ordnen.

Dies ist nicht nur ein Theoretisches Problem. Manchchmal ist ein Film einfach
noch nicht bewertet worden und hat deshalb keinen Score, manchmal hat der
Anbieter einer Immobilie aus Datenschutzgründen keine Addresse angegeben und
manchmal findet ein XPath einfach die falschen Daten.
