# Xpaths

Der Xpaths Tab zeigt alle Xpaths der aktuellen Seite an. Xpaths geben an, wie
das HTML DOM navigiert werden muss, um die Daten wirklich zu extrahieren.

<img style="width: 70%; display: block; margin: 0 auto;" src="chapter_xpaths_1.png"/>

Ein Klick auf `Generate Xpaths` sendet die aktuell annotierte Seite an den
Server und aktuallisiert die Liste nach dem Abschluss mit den neuen Werten.

Bei Bedarf können auch Parameter der XPath Generierung angepasst werden. Diese
Einstellungen sind:

- **K**: Die Menge an Xpaths, die Generiert werden. Dies beeinflusst ebenfalls
  die Qualität der Xpaths
- **Complex XPaths**: Erzeugt Xpaths, die rekursiv andere Xpaths enthalten. Ein
Beispiel wäre: `descendant::div[descendant::h1]`

Beise Einstellungen beeinflussen nicht nur die Qualität, sondern auch die Dauer
der Generierung.

# Xpaths auswählen

In manchen Fällen kann es dazu kommen, dass der am besten bewertete XPath zu
spezifisch ist, also z.B. den gesuchten Titel eines Films enthällt:

```c
descendant::h1::[text()='Mission Impossible'] // Das Element mit genau diesem Text
```

oder sich zu sehr auf die absolute Position des Elements bezieht:

```c
descendant::li[8] // Das 8. Listen element auf der Seite
```

Um sich manuell den besten XPath auszuwählen, kann man in der Liste der
generierten XPaths mit dem `Set` Knopf einen Xpath auswählen, der besser
funktioniert. Dieser wird daraufhin an die erste Stelle gesetzt.
