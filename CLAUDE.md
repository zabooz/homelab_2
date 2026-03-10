# Project Guide

Homelab Wiki - Dokumentation für die gesamte Homelab-Infrastruktur, Services und Netzwerk-Konzepte. Wird als Wiki.js Wiki betrieben und via Git synchronisiert.

## Konventionen

- Sprache: Deutsch (mit englischen Fachbegriffen)
- Alle .md Dateien haben Wiki.js YAML Frontmatter (title, description, published, date, tags, editor, dateCreated)
- Tags: bilingual (deutsch + englisch), kommagetrennt
- Wiki.js Links müssen absolute Pfade mit `/en/` Locale-Prefix verwenden, ohne `.md` Extension (z.B. `(/en/konzepte/routing)`, NICHT `(routing)` oder `(../konzepte/routing.md)`)
- Diagramme als ASCII-Art in Code-Blöcken
- `home.md` ist der Index aller Seiten — bei neuen Docs dort verlinken
- Details zu Hosts, IPs, Services stehen in den Docs selbst (siehe `infrastructure/NETWORK_OVERVIEW.md`)
