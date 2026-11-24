# RadioSense — lecteur de musique intelligent

Projet de l'épreuve finale du cours 420-314 (Cégep Marie-Victorin, automne 2026).

RadioSense est un lecteur de musique autonome sur ESP32 (MicroPython). Il joue des mélodies enregistrées dans sa mémoire flash (format RTTTL) sur un buzzer passif, affiche le titre et le volume sur un LCD 16x2, se met en veille quand personne n'est présent (capteur PIR) et passe en mode NUIT dans une pièce sombre (photorésistance).

Tous les composants viennent de la liste des composants disponibles du cours, sauf l'ESP32 DevKit V1.

## Membres

- Madhav Ramdin
- Jade Mendoza

## Documentation

- [Livrable 1 — Choix de projet et plan de projet](docs/livrable1.md)
- Version Word : [RadioSense_Livrable1_Complet-v3.docx](RadioSense_Livrable1_Complet-v3.docx)

## Structure

```
docs/    documentation des livrables
src/     code MicroPython (à venir)
tests/   tests des modules (à venir)
```
