# CLAUDE.md — RadioSense

Lecteur de musique autonome sur **ESP32 DevKit V1 (WROOM-32) en MicroPython**.
Projet de l'épreuve finale du cours 420-314, Cégep Marie-Victorin, automne 2026.
Équipe : Madhav Ramdin et Jade Mendoza. Enseignant : Maxime.

RadioSense joue des **mélodies RTTTL** stockées dans la mémoire flash de l'ESP32 sur un **buzzer
passif** (PWM), affiche titre, piste, volume et état sur un LCD 16x2, se met en veille sans
présence (PIR) et passe en mode NUIT dans une pièce sombre (photorésistance).

## Contrainte matérielle principale

**Seuls les composants de la liste du cours sont permis** (`Composants_disponibles.xlsx`, dans le
dossier OneDrive du cours 420-314), sauf l'ESP32 DevKit V1. Cette liste n'a ni ampli I2S, ni
lecteur microSD, ni haut-parleur : c'est pourquoi le son passe par le buzzer passif et que les
mélodies sont dans la flash. Ne pas proposer de composant hors liste sans le signaler clairement.

Composants utilisés (nom exact dans la liste) : LCD Module, Passive Buzzer, push button 2 pin (x3),
Analog Rotation Sensor, Photocell Sensor, PIR Motion Sensor, White LED Module (x2), Power Bank.

## Langue et style

- Documentation, commentaires, messages de commit et textes du LCD : **en français**.
- Identifiants de code (variables, fonctions, modules) : en français sans accents
  (ex. `etat`, `volume_max`, `lire_ldr()`), ou en anglais, mais de façon cohérente dans tout le dépôt.
- Le LCD HD44780 n'affiche pas les accents ni les symboles ▶/❚❚ sans caractères personnalisés :
  passer par `custom_char()` ou écrire les messages sans accents.

## Sources de vérité

- `docs/livrable1.md` : plan de projet (diagrammes Mermaid, broches, états, risques). **Référence principale.**
- `RadioSense_Livrable1_Complet-v3.docx` : version Word à remettre (même contenu, mise en page).
  La v2 (ancienne conception avec microSD et ampli I2S) est périmée.
- Toute modification du câblage ou du comportement doit être reportée dans `docs/livrable1.md`
  et dans le Word du livrable en cours.

## Structure du dépôt

```
docs/    documentation des livrables (Markdown)
src/     code MicroPython copié sur l'ESP32 (main.py, boot.py, modules)
tests/   petits scripts de test d'un module à la fois (Livrable 2)
```

Organisation prévue de `src/` (un module par périphérique, à créer) :
`main.py` (boucle principale + machine à états), `config.py` (broches et constantes),
`lcd.py`, `melodie.py` (analyse RTTTL + lecteur de notes non bloquant + volume),
`capteurs.py` (PIR, photorésistance, potentiomètre), `boutons.py` (anti-rebond),
`melodies/` (fichiers `.txt` RTTTL copiés dans la flash).

## Attribution des broches (ne pas changer sans mettre à jour la doc)

| GPIO | Composant | Type | Remarque |
|---|---|---|---|
| 21 / 22 | LCD Module SDA / SCL | I2C | PCF8574, adresse 0x27 (vérifier avec `i2c.scan()`, parfois 0x3F) |
| 25 | Passive Buzzer | PWM | `freq` = note, `duty` = volume |
| 32 / 33 / 13 | Boutons PRÉC. / LECTURE-PAUSE / SUIV. | Entrée | `Pin.PULL_UP`, actif à LOW |
| 34 | Analog Rotation Sensor (volume) | ADC1 | `ADC.ATTN_11DB`, entrée seulement |
| 35 | Photocell Sensor | ADC1 | `ADC.ATTN_11DB`, entrée seulement ; sens de variation à calibrer |
| 14 | PIR Motion Sensor | Entrée | Sortie ≈ 3,3 V |
| 17 | DEL rythme (White LED Module) | PWM | Résistance sur le module |
| 16 | DEL d'état (White LED Module) | Sortie | OK sur WROOM, pas sur WROVER |

Règles : seulement l'ADC1 (l'ADC2 est inutilisable avec le Wi-Fi) ; GPIO 34–39 sans pull-up
interne ; ne pas utiliser GPIO 0, 2, 5, 12, 15 (broches de démarrage) ; jamais plus de 3,3 V sur
une entrée. Alimentation : LCD et PIR en 5 V (VIN), photorésistance et potentiomètre en 3,3 V.

## Machine à états

États : `DEMARRAGE`, `LECTURE`, `PAUSE`, `VEILLE`, `ERREUR`, plus le **modificateur NUIT**
(actif en LECTURE et PAUSE, ce n'est pas un état séparé).

- DEMARRAGE → PAUSE si le LCD répond et qu'au moins une mélodie est valide ; sinon → ERREUR.
- PAUSE ⇄ LECTURE : bouton LECTURE/PAUSE.
- LECTURE : PRÉC./SUIV. ou fin de mélodie → mélodie précédente/suivante (reste en LECTURE).
- LECTURE ou PAUSE → VEILLE après 60 s sans présence ; retour à l'état d'avant dès qu'une présence est détectée.
- LECTURE → ERREUR si plus aucune mélodie n'est lisible ; ERREUR → DEMARRAGE toutes les 3 s ou sur un bouton.
- NUIT : photorésistance sous le seuil bas → volume plafonné à 30 % et DEL tamisée ; au-dessus du seuil haut → normal (hystérésis).

| État | LCD | DEL d'état |
|---|---|---|
| DEMARRAGE | « Chargement… » | éteinte |
| LECTURE / PAUSE | titre, n° de piste, volume, ▶ / ❚❚ | éteinte |
| VEILLE | rétroéclairage coupé | clignote à 1 Hz |
| ERREUR | « Aucune mélodie » (si le LCD répond) | allumée fixe |

## Contraintes de code (exigences du cours)

- **Code non bloquant obligatoire** : pas de `time.sleep()` dans la boucle principale.
  Utiliser `time.ticks_ms()` et `time.ticks_diff()` pour tous les délais : durée de chaque note,
  veille 60 s, clignotement 1 Hz, nouvel essai 3 s, anti-rebond, rafraîchissement LCD.
- Lecteur de mélodies : la boucle vérifie si la note courante est finie ; si oui, elle règle
  `buzzer.freq(note)` et `buzzer.duty_u16(volume)` pour la suivante (silence = `duty_u16(0)`).
- Format RTTTL : `Titre:d=4,o=5,b=120:c,e,g,...`. Analyser chaque fichier au chargement en une
  liste `(frequence_hz, duree_ms)` ; un fichier invalide (`ValueError`, `OSError`) est sauté,
  jamais de plantage. Utiliser des airs du domaine public ou composés par l'équipe.
- Volume : rapport cyclique PWM (max 50 %) avec courbe de correction, selon le potentiomètre,
  plafonné à 30 % du volume en mode NUIT. La DEL rythme suit le volume de la note courante.
- Capteurs : moyenne de plusieurs lectures ADC + hystérésis (photorésistance), anti-rebond
  logiciel des boutons, temporisation de stabilisation du PIR au démarrage (~60 s de chauffe).
- Démarrage autonome : `main.py` se lance seul à la mise sous tension, sans ordinateur.
- Les constantes (broches, seuils, délais) vont dans `config.py`, pas en dur dans le code.

## Travailler avec la carte

Les outils ne sont pas encore fixés dans le dépôt. Par défaut, utiliser `mpremote` :

```sh
mpremote connect auto fs cp -r src/. :     # copier le code (et melodies/) sur l'ESP32
mpremote connect auto run tests/test_lcd.py # lancer un test sans le copier
mpremote connect auto repl                  # console série
```

Il n'y a pas de tests automatisés sur ordinateur : les scripts de `tests/` se lancent sur la carte
et vérifient un module à la fois (LCD, buzzer, boutons, capteurs). Le code ne peut pas être exécuté
ici sans l'ESP32 branché ; le dire clairement plutôt que d'affirmer qu'il fonctionne.

## Échéancier

| Date | Étape |
|---|---|
| Semaine du 23 nov. 2026 | Livrable 1 : plan de projet, emprunt du matériel |
| Semaine du 30 nov. | Livrable 2 : chaque module testé seul |
| Semaine du 7 déc. | Livrable 3 : intégration, machine à états, maquette |
| 10 déc. | Remise finale (code, documentation, maquette) |
| Semaine du 14 déc. | Examen oral |

## Répartition

- **Madhav** : buzzer, lecteur de mélodies RTTTL, liste de pistes, volume PWM, DEL rythme, DEL d'état, état ERREUR.
- **Jade** : LCD, boutons, potentiomètre, machine à états, anti-rebond, PIR (veille), photorésistance (NUIT).
- **Ensemble** : intégration, documentation, oral.

## Points à corriger

- Le dépôt distant s'appelle encore `Esp-32-alarme` (ancien projet ChronoAlarme), alors que le
  dossier local et le lien du Word disent `Esp-32-Radio`.
