# Konzept

Um das hier genutzte System zu verstehen, muss man sich zunächst überlegen, wie
man naiv Daten aus Webseiten extrahieren könnte. Nehmen wir als Beispiel eine
einfache Webseite, die nur Daten zu einem Film enthällt.

```html
<html>
  <body>
    <h1>Mission Impossible</h1>
    <h2>1996 - PG-13 - 1h 50m</h2>
    <h3>
      An American agent, under false suspicion of disloyalty, must discover and
      expose the real spy without the help of his organization.
    </h3>
    <div id="actors">
      <div class="actor">
        <a href="https://filminfos.de/actor/a32342">Link</a>
        <h2>Tom Cruise</h2>
        <h3>Ethan Hunt</h3>
      </div>
      <div class="actor">
        <a href="https://filminfos.de/actor/a714241">Link</a>
        <h2>Jon Voight</h2>
        <h3>Jim Phelps</h3>
      </div>
    </div>
  </body>
</html>
```

Wir könnten nun einfach für jedem Wert, den wir extrahieren wollen, einen XPath
zuweisen, also z.B.

```yaml
Title: descendant::h1
ActorNames: descendant::div[@class='actors']/descendant::h3
ActorUrl: descendant::div[@class='actors']/descendant::a
ActorRoles: descendant::div[@class='actors']/descendant::h3
```

Die Werte zu extrahieren funktioniert damit problemlos:

```yaml
Title: ["Mission Impossible"]
ActorNames: ["Tom Cruise", "Jon Voight"]
ActorRoles: ["Ethan Hunt", "Jim Phelps"]
ActorUrls:
  ["https://filminfos.de/actor/a32342", "https://filminfos.de/actor/a714241"]
```

Dank spezieller Array Datentypen in Datenbanken wie Postgres, könnten wir die
Daten auch direkt in eine Datenbank einfügen.

Später können wir dann ggf. manuell zuordnen was zu was gehört, anhand der
Position innerhalb der arrays.

Wir verlassen uns dabei allerdings darauf, dass dass unsere erste extrahierte
Actor URL auch zu dem ersten Namen und der ersten Rolle gehört.

Schauen wir uns allerdings dieses Beispiel an

```yaml
Title: "Mission Impossible"
ActorNames: ["Tom Cruise", "Jon Voight"]
ActorRoles: ["Ethan Hunt"]
ActorUrls:
  ["https://filminfos.de/actor/a32342", "https://filminfos.de/actor/a714241"]
```

so sehene wir, dass wir in diesem Fall nicht mehr zuordnen können welche Rolle
zu welchem Actor gehört. Noch schlimmer sieht es hier aus:

```yaml
Title: "Mission Impossible"
ActorNames: ["", "Tom Cruise", "Jon Voight"]
ActorRoles:
  [
    "Ethan Hunt",
    "<a href='http://temu.com'>Buy TOP Chinese Toy Planes NOW!! <a>",
    "Jim Phelps",
  ]
ActorUrls:
  [
    "https://filminfos.de/actor/a32342",
    "https://google.com",
    "https://filminfos.de/actor/a714241",
  ]
```

Hier haben wir die selbe Anzahl an Ergebnissen aber es ist ersichtlich, dass
eine einfache positionelle Zuordnung trotzdem nicht möglich ist.

Dies ist nicht nur ein theoretisches Problem. Das Fehlen von Daten kann durchaus
beabsichtigt sein, XPaths können zu schlecht sein und Seiten verändern sich ständig.
Was wir brauchen ist ein besseres System, um diese Daten zu zu ordnen:

## Nodes

Schauen wir uns die Beispiel Seite noch einmal an so sehen wir, dass es
bestimmten Bereich gibt, die zu underschiedlichen Dingen gehören.

```html
<!-- Movie Start -->
<html>
  <body>
    <h2>1996 - PG-13 - 1h 50m</h2>
    <h3>
      An American agent, under false suspicion of disloyalty, must discover and
      expose the real spy without the help of his organization.
    </h3>
    <div id="actors">
      <!-- Actor 1 Start -->
      <div class="actor">
        <a href="https://filminfos.de/actor/a32342">Link</a>
        <h2>Tom Cruise</h2>
        <h3>Ethan Hunt</h3>
      </div>
      <!-- Actor 1 End -->
      <!-- Actor 2 Start -->
      <div class="actor">
        <a href="https://filminfos.de/actor/a714241">Link</a>
        <h2>Jon Voight</h2>
        <h3>Jim Phelps</h3>
      </div>
      <!-- Actor 2 End -->
    </div>
  </body>
</html>
<!-- Movie End -->
```

Da der Begriff "Bereiche" etwas ungenau definiert ist, nehmen wir statdessen den
HTML Begriff `Nodes`, der immer genau den Bereich zwischen: `<></>` Klammern
bezeichnet. In diesem Fall wäre der erste Actor Node also:

```html
<div class="actor">
  <a href="https://filminfos.de/actor/a32342">Link</a>
  <h2>Tom Cruise</h2>
  <h3>Ethan Hunt</h3>
</div>
```

Da wir nun mit HTML Nodes arbeiten, können wir auch einfach Xpaths finden, um
diese zu selektieren und relativ zu diesen dann die wirklichen Daten
extrahieren.

Natürlich ist dies nur ein minimal-Beispiel, doch in der Praxis sind Nodes oft
wirklich nicht viel größer, als hier gezeigt. Da wir nur noch einen Bruchteil
der Seite betrachten, wird sowohl die Xpath Generierung, als auch die Extrahierung und
Zuordnung trivial. Alle Daten, die wir extrahieren gehören immer genau zu dem
Node, zu dem sie relativ ausgewertet wurden.

Um Verwechslungen zu vermeiden, wird bei diese Xpaths nun angegeben zu welchem
Node diese relativ sind. Unsern relativen Xpaths sind nun also:

```yaml
Movie.Title: descendant::h1
Movie.Actor: descendant::div[@class='actor']
Actor.Name: child::h3
Actor.Url: child::a
Actor.Role: child::h3
```

Und bilden damit folgende Struktur ab:

```
        [Movie]
           |
    +------+--------+
    |               |
  Title          [Actor]
                    |
         +----------+-----------+
         |          |           |
        Url        Name        Role
```

Es sollte ebenfalls verständlich sein, dass wir beim Extrahieren der Daten
immer ganz Oben anfangen müssen und uns nach unten vorarbeiten müssen, um
sicher zu sein, dass der Node zu dem man relativ ist bereits ausgewertet wurde.

Wer mit Baumdiagrammen und der Unterscheidung zwischen Leafs und Nodes vertraut
ist, dem fällt auf, dass unsere Nodes auch im Baumdiagramm die Nodes
sind. Die Leafs, oder auf deutsch Blätter in diesem Diagramm sind die
wirklichen Daten, die wir Extrahieren. Um zwischen diesen beiden Arten zu
Unterscheiden, nennen wir die Blätter nun "Data Points", da nur diese wirklich
Daten extrahieren.

Wenn wir diese Methode nutzen, um die Daten zu extrahieren, erhalten wir
folgende Daten:

```json
{
  "Movie": [
    {
      "Title": "Mission Impossible",
      "Actor": [
        {
          "Name": "Tom Cruise",
          "Role": "Ethan Hunt",
          "Url": "https://filminfos.de/actor/a32342"
        },
        {
          "Name": "Jon Voight",
          "Role": "Jim Phelps",
          "Url": "https://filminfos.de/actor/a714241"
        }
      ]
    }
  ]
}
```

Es sollte ersichtlich sein, dass das Fehlen von z.B. einer Rolle hier klar
zugeordnet werden kann und dass zusätzliche Werte nur innerhalb eines Nodes
zu falschen Ergebnissen führen können, nichtaber über diese hinaus.

## Idents

Das einzige verbleibende Problem ist die Zuordnung dieser Ergebnisse in eine
Datenbank. Wenn wir überlegen, wie eine passende Datenbank aussieht, könnten
wir uns folgendes Schema überlegen:

```
+---------------+       +------------------+       +--------------+
|     Movie     |       |   Actor2Movie    |       |     Actor    |
+---------------+       +------------------+       +--------------+
| PK:     id    |<------| PK,FK1: movie_id |   /-->| PK:     id   |
| UNIQUE: url   |       | PK,FK2: actor_id |--/    | UNIQUE: url  |
|         title |       |         role     |       |         name |
+---------------+       +------------------+       +--------------+
```

Wir können dieses Schema auch auf eine etwas untypischere Art visualisieren:

```
 [Movie(PK: id, uniq: url)] <------------------------+
         |                                           |
  +------+--------+                                  |
  |      |        |                                  |
 title   url    [Actor(PK: id, uniq: url)] <--- [Actor2Movie]
                   |                                 |
        +----------+                                 |
        |          |                                 |
       url        name                              role
```

Im Vergleich zu unserer Node/Points definition von zuvor erkennt man, dass wir
Nodes Tabellen zuordnen können und die Punkte in den meisten Fällen direkt den
Spalten der Tabelle. Mit diesem Mapping im Kopf, können wir auch einmal
durchgehen, wie das wirkliche Einfügen der zuvor extrahierten Daten abläuft.
Dieser Prozess besteht immer grob aus folgenden Schritten:

1. Starte beim obersten Knoten (dem root node)
2. Wenn unser node eine assoziierte Tabelle hat
    - Finde oder füge eine neue Zeile in die Tabelle ein, die dem einem einmaligen Mapping entspricht
      - Dieses einmalige Mapping besteht aus extrahierten Punkte, und/oder die oder der aktuelle URL zu Spalten in der DB
      - Dieses Mappings nennen wir **Idents**
3. Wenn unser node eine assoziierte Many To Many Tabelle hat
    - Finde oder füge eine neue Zeile in die Many To Many Tabelle ein
      - Nutze dafür die zuvor gefundene Zeile unserer Tabelle und die id unseres parent Nodes
4. Gehe alle unsere points durch und füge diese entweder in unsere, oder die Many To Many Tabelle ein
5. Für alle Nodes, die direkt unter unserem node sind, beginne erneut bei Schritt 2

Da wir als input natürlich kein Diagramm haben, nehmen wir eine Konfiguration
wie diese, die praktisch das selbe wie das Diagramm oben über Nodes & Points aussagt:

```yaml
Movie:
  table: Movie
  idents:
    - url: CURRENT_PAGE_URL
  points:
    - title: "insert node table"
  children_nodes:
    Actor:
      table: Actor
      mtm_table: Actor2Movie
      idents:
        - url: point-url
      points:
        - url: "handled in idents"
        - name: "insert node table"
        - role: "insert node mtm table"
```

```
1. Beginne beim Movie Node
2. Ident definiert dass unsere aktuelle URL zu der url Spalte gehört
   Keine Zeile mit unserer url in der Datenbank gefunden
   Füge neue Spalte ein => movie_id = 1
3. Wir haben keine MtM tabelle
4. Füge Title in die title Spalte ein
5. Beginne erneut bei 2. mit Actor 1 & Actor 2
  ===>
  2. Ident definiert dass unser url point zur url Spalte in der DB gehört
     Keine Zeile mit der url in der Datenbank gefunden
     Füge neue Spalte ein => actor_id = 1
  3. Wir haben eine many2many tabelle. Wir haben actor_id = 1, movie_id = 1
     Keine Zeile mit den Werten in der Datenbank gefunden
     Füge neue Spalte ein => actor_to_movie_id = 1
  4. Füge Name in die name Spalte ein
     Füge Role in die role Spalte der mtm ein
  <===
  ===>
  2. Ident definiert dass unser url point zur url Spalte in der DB gehört
     Keine Zeile mit der url in der Datenbank gefunden
     Füge neue Spalte ein => actor_id = 2
  3. Wir haben eine many2many tabelle. Wir haben actor_id = 2, movie_id = 1
     Keine Zeile mit den Werten in der Datenbank gefunden
     Füge neue Spalte ein => actor_to_movie_id = 2
  4. Füge Name in die name Spalte ein
     Füge Role in die role Spalte der mtm ein
  <===
```

## Automatisierung

Das ganze wirkt auf den ersten Blick ggf. sehr komplex (und das kann es auch
sein). Es bietet allerdings auch viele Möglichkeiten Prozesse zu vereinfachen.

Um Xpaths zu generieren müssen wir die HTML Seite ohnehin annotieren und diesen
Annotationen sinnvolle namen geben. Was also, wenn wir den Prozess ähnlich
durchgehen, wie die Extraktion, also von oben nach unten. Wir nennen unsere
annotation movie und wählen einfach mit einem Klick die ganze Seite aus.
```html
  <body webdb='Movie'>
  ...
```
Sofern wir voneinem engen Annotation=>Node=>Table mapping ausgehen, können wir ohne
weiters zu tun bereits den erforderlichen Node und die Tabelle erstellen.

Das gleiche wiederholen wir mit dem Actor.
```html
<body webdb='Movie'>
...
    <div class="actor" webdb="actor">
        ...
    </div>
...
    <div class="actor" webdb="actor">
        ...
    </div>
...
```

Da dieser nun innerhalb des HTML nodes des Movies ist wissen wir, dass auch der
Actor node unterhalb des Movie nodes sein muss.
Damit kenenn wir auch die Tabellen Relationen und können eine MtM Tabelle erstellen.

Markieren wir nun die Punkte, müssen wir lediglich angeben, ob diese in die
jeweilige MtM, oder die normale tabelle geschrieben werden müssen. Die
assoziation zu points lässt sich daraus ableiten was der erste annotierte HTML
Knoten ist, der darüber liegt.

```html
<div class="actor" webdb="actor">  <<<<<<<<<<<<<<<<<<<<+ name muss zu actor gehören
  <a href="https://filminfos.de/actor/a32342">Link</a> |
  <h2 webdb="name">Tom Cruise</h2>  >>>>>>>>>>>>>>>>>>>+
  <h3>Ethan Hunt</h3>
</div>
```

Mit diesem Verfahren lassen sich Punkte und zugehörige Tabellen Spalten nach
kurzer Annotation der Nodes also eigens durch einen Namen und das Klicken auf
das Element definieren.

Zudem können wir auch Seiten übergreifend agieren. Angenommen wir wollen auf der
Such-Seite die URL der gefundenen filme extrahieren, oder auf der Movie Seite
die Actors beim Crawlen verfolgen. Wir können einfach auswählen, dass wir einen
wichtigen Link annotieren. Sobald wir den Punkt ausgewählt haben, können wir:
- Direkt den richtigen Node finden
- Einen Eintrag für die URL zur Tabelle des Nodes hinzufügen
- Den Link als Ident zum Node hinzufügen
- Den Link als beim crawling weiter zu verfolgendend definieren
- Eine neue leere Seiten Definition erstellen
- Einen obersten Knoten auf dieser Seite erstellen, der die page url nimmt und
  als ident für die gerade erstellte Spalte festlegt

Die einzigen Schritte, die ein Nutzer wirklich noch machen muss sind:
- Namen vergeben
- Auswählen, ob mehrere Elemente Annotiert werden sollen
- Zwischen Node & Point unterscheiden
- Element(e) auswählen

Im folgenden wird nun erklärt, wie die Software wirklich praktisch zu benutzen ist
