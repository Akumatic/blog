---
title: Windows 11 mit lokalem Konto einrichten
date: 2024-04-10
description: |
    Das Setup von Windows 11 verhindert, die Einrichtung ohne eine aktive Internetverbindung fortzusetzen. Mit einem einfachen Trick kann man dies umgehen und weiterhin einen lokalen Account anlegen.
categories: 
    - os
---

Schon seit Windows 10 strebt Microsoft an, dass der Rechner mit einem Microsoft-Account benutzt wird. Während des Setups konnte man kein lokales Konto einrichten, wenn man mit dem Internet verbunden war. Um die Option für das lokale Konto angezeigt zu bekomen, musste man also offline bleiben.

Im aktuellen Setup von Windows 11 scheint dies nichts mehr zu bringen. Um im Setup fortfahren zu können, fordert Microsoft eine Internetverbindung.

![Ohne Internetverbindung kann das Setup nicht fortgesetzt werden.](img/missing_link.de.png)

Auch wenn es nicht mehr so trivial wie früher ist, lässt sich diese Anforderung immer noch umgehen. Hierfür kann im Setup mit der Tastenkombination <kbd>STRG</kbd> + <kbd>F10</kbd> eine Konsole geöffnet werden. Dort gibt man folgenden Befehl ein und bestätigt ihn:

```
OOBE\BYPASSNRO
```

Anschließend startet das System neu. Folgt man dem Assistenten wieder bis zum vorherigen Schritt, ist nun die Option "Ich habe kein Internet" verfügbar:

![Nach dem Neustart ist eine neue Funktion sichtbar, um auch ohne Internet forzufahren.](img/available_link.de.png)

Dadurch kann man mit der "eingeschränkter Einrichtung" fortfahren und ein lokales Konto anlegen.
