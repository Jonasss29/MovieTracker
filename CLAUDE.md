# MovieTracker

Grundlage: `docs/PLAN.md` (Entscheidungen, Phasenplan) und `docs/SECURITY-CONCEPT.md` (Sicherheitskonzept). Das ursprüngliche Briefing liegt nur lokal und ist nicht Teil des Repos.
Projektdokumente auf Deutsch, Bezeichner englisch im Code-Format. Code, Kommentare, Commits: Englisch.
Diese Datei ist vorläufig und wird in Phase 2 durch die vollständige Fassung ersetzt.

## Git
Commits nur auf Anfrage. Gearbeitet wird auf einem Branch je Schritt, zurück nach `main` per Pull Request mit grüner CI und „Rebase and merge“ (s. `docs/PLAN.md`, Zeile „Branches“). Kein direkter Push auf `main`.

## Privatsphäre
Keine persönlichen Daten im Repo: keine Namen, keine Benutzerpfade, keine echten E-Mail-Adressen (Commit-Autor = GitHub-noreply-Adresse). Persönliche Hinweise gehören in `CLAUDE.local.md` (nicht im Repo).
