+++
date = '2026-05-19T09:00:00+02:00'
draft = false
title = 'frfix : un démon de correction orthographique du français au niveau système pour Linux'
+++
**tl;dr :** les navigateurs web, les applications de messagerie et les IDE embarquent chacun leur propre correcteur orthographique, sauf votre terminal, votre application de notes protégée par mot de passe et la moitié des outils de niche que vous utilisez vraiment tous les jours. `frfix` est un petit démon Python qui lit les frappes clavier via `evdev`, les passe à travers `hunspell` et `grammalecte`, et réécrit silencieusement les fautes de français partout sous Linux/X11. Aucune extension, aucune astuce de presse-papiers, aucun moteur de méthode de saisie. Source : [github.com/lemmebee/frfix](https://github.com/lemmebee/frfix).

---
## Table des matières
- [Le problème : la correction orthographique s'arrête à la frontière de l'application](#le-probleme-la-correction-orthographique-sarrete-a-la-frontiere-de-lapplication)
- [L'idée : une correction qui agit au lieu de souligner](#lidee-une-correction-qui-agit-au-lieu-de-souligner)
- [Ce qu'il corrige](#ce-quil-corrige)
- [Comment ça fonctionne](#comment-ca-fonctionne)
  - [1. Lire les frappes clavier avec `evdev`](#1-lire-les-frappes-clavier-avec-evdev)
  - [2. Traduire les keycodes via X11/xkb](#2-traduire-les-keycodes-via-x11xkb)
  - [3. Construire des mots et des phrases](#3-construire-des-mots-et-des-phrases)
  - [4. Corriger avec hunspell + grammalecte](#4-corriger-avec-hunspell--grammalecte)
  - [5. Injecter la correction avec `xdotool`](#5-injecter-la-correction-avec-xdotool)
- [Pourquoi X11 uniquement (et pas Wayland)](#pourquoi-x11-uniquement-et-pas-wayland)
- [Sécurité : les secrets que vous tapez doivent le rester](#securite-les-secrets-que-vous-tapez-doivent-le-rester)
- [Installation et lancement](#installation-et-lancement)
- [Configuration](#configuration)
- [Limites et compromis assumés](#limites-et-compromis-assumes)
- [La suite](#la-suite)

## Le problème : la correction orthographique s'arrête à la frontière de l'application

Écrire en français sous Linux, c'est une accumulation de petites irritations. Firefox corrige automatiquement, l'éditeur de texte de GNOME souligne, Slack en attrape la moitié. Puis vous basculez vers un terminal, une application Tauri, un outil Electron qui a désactivé la correction orthographique, une session SSH, un formulaire React personnalisé avec `spellcheck="false"`, ou un widget GTK que personne n'a pris la peine de câbler, et vous revoilà à taper `francais` et `cest` en faisant comme si personne n'avait remarqué.

Chaque application réinvente la correction orthographique. La plupart le font mal. Beaucoup s'en passent complètement. Résultat : vous traînez la même charge mentale du « pense aux accents » dans chaque champ de texte du système, et votre mémoire musculaire perd quand même.

Je voulais que la correction orthographique soit une propriété du **clavier**, pas de l'**application**.

## L'idée : une correction qui agit au lieu de souligner

`frfix` est un démon d'arrière-plan. Il fait trois choses en boucle :

1. Il surveille les frappes clavier à l'échelle du système via `/dev/input`.
2. Il maintient un petit tampon de mot/phrase.
3. Quand un mot se termine (espace, ponctuation, Entrée), il vérifie le mot/la phrase avec hunspell et grammalecte. Si c'est fautif, il le réécrit sur place : retour arrière × N, puis frappe de la correction.

Il n'y a aucune interface avec laquelle interagir. Il n'y a aucune intégration applicative. Il ne sait pas quelle fenêtre a le focus, au-delà d'une vérification d'exclusion. Il reste simplement là, vous regarde taper, et corrige les fautes à l'instant même où le mot se termine, dans le terminal, dans `vim`, dans une application Electron qui a désactivé la correction, dans `xterm`, partout.

Voyez-le comme un correcteur orthographique uniquement français, à l'échelle du système, qui *agit* au lieu de souligner.

## Ce qu'il corrige

Les quatre catégories qui couvrent environ 95 % des fautes de frappe françaises du quotidien :

- **Accents manquants** : `francais` → `français`, `realite` → `réalité`, `ecran` → `écran`, `etre` → `être`.
- **Élisions** : `cest` → `c'est`, `jai` → `j'ai`, `lhomme` → `l'homme`, `cetait` → `c'était`, `quil` → `qu'il`. La classe de faute la plus agaçante quand on vient d'un layout anglais où l'apostrophe est mal placée.
- **Inversions AZERTY/QWERTY** : `vrqi` → `vrai`, `fqire` → `faire`, `voulqis` → `voulais`. Le grand classique du « j'ai oublié quel layout était actif ».
- **Grammaire en fin de phrase** : `tu peut` → `tu peux`, `je veut` → `je veux`, les petits accords simples. Pris en charge par grammalecte une fois la phrase terminée.

Il y a aussi une **annulation** : `Ctrl+Z` immédiatement après une correction la révoque. Un petit tampon circulaire garde en mémoire les dernières corrections pour que vous puissiez revenir en arrière si frfix se trompe.

Le système est **conservateur par conception** : si un mot est déjà un mot français valide, rien ne se passe. Les faux positifs sont l'ennemi. Un correcteur qui réécrit ce que vous avez tapé volontairement est pire que pas de correcteur du tout.

## Comment ça fonctionne

```
                ┌──────────┐    keysym +    ┌──────────┐
   /dev/input ─▶│  evdev   │───keycode ────▶│ KeyTrans │
                │ capture  │  (read-only)   │  X11/xkb │
                └──────────┘                └────┬─────┘
                                                 │ char
                                                 ▼
                              ┌───────────────────────────────┐
                              │ TextBuffer (word + sentence)  │
                              └────────────┬──────────────────┘
                                           │ word / sentence
                                           ▼
                              ┌───────────────────────────────┐
                              │ FrenchCorrector               │
                              │  • hunspell (spelling)        │
                              │  • grammalecte (grammar)      │
                              │  • user dictionary            │
                              └────────────┬──────────────────┘
                                           │ replacement
                                           ▼
                              ┌───────────────────────────────┐
                              │ Injector (xdotool)            │
                              │  backspace × N + type fix     │
                              └───────────────────────────────┘
```

### 1. Lire les frappes clavier avec `evdev`

Linux expose les périphériques de saisie sous `/dev/input/event*`. Avec l'appartenance au groupe `input`, un processus peut ouvrir ces fichiers en lecture seule et recevoir chaque événement clavier que le noyau voit, indépendamment du serveur X, du compositeur ou de l'application ayant le focus. `python-evdev` enveloppe cela dans un itérateur asynchrone.

Point crucial : il s'agit d'une **capture passive**. Il n'y a aucun keyboard grab. Toutes les autres applications reçoivent chaque touche normalement. frfix est un observateur silencieux, pas un intermédiaire.

### 2. Traduire les keycodes via X11/xkb

`evdev` vous donne des keycodes Linux bruts (`KEY_A`, `KEY_SEMICOLON`, …). Ceux-ci sont inutiles à eux seuls : la même touche physique produit `q` en AZERTY et `a` en QWERTY. Pour savoir quel caractère a réellement été tapé, frfix interroge le **layout xkb actif** via X11 et résout keycode → keysym → caractère à travers la table du layout.

C'est ce qui lui permet de gérer correctement un utilisateur qui bascule entre QWERTY et AZERTY en cours de session : la même `KEY_Q` produit `q` ou `a` selon le layout que xkb déclare actif à l'instant présent.

Par défaut, frfix ne s'exécute que lorsque le layout courant est `fr`. Au démarrage, il bascule vers `fr` et mémorise votre layout précédent ; à la sortie, il le restaure. (Vous pouvez outrepasser ce comportement avec `--no-layout-check` ou `--force-layout`.)

### 3. Construire des mots et des phrases

Un `TextBuffer` conserve deux éléments d'état :

- Le **mot courant**, qui grandit caractère par caractère.
- La **phrase courante**, qui s'accumule à travers les frontières de mots.

Les frontières de mots (espace, tabulation, ponctuation, Entrée) déclenchent les vérifications orthographiques. Les frontières de phrases (`.`, `!`, `?`) déclenchent les vérifications grammaticales. Le retour arrière raccourcit le mot courant. Le tampon est réinitialisé lors des changements de focus et lors des corrections injectées (pour que frfix n'essaie pas de « corriger » récursivement sa propre correction).

### 4. Corriger avec hunspell + grammalecte

Deux couches de vérification, dans cet ordre :

- **`hunspell` + `hunspell-fr-comprehensive`** pour les vérifications au niveau du mot. Si le mot est du français valide, ne rien faire. Sinon, générer des candidats avec le moteur de suggestion de hunspell, plus des règles ciblées pour les cas fréquents (accent manquant sur un radical connu, apostrophe manquante après `c`/`j`/`l`/`qu`/`d`/`n`/`s`/`t`, substitution AZERTY/QWERTY).
- **`grammalecte`** (optionnel, via GObject Introspection) pour la grammaire au niveau de la phrase. Il attrape les petits accords de verbe que hunspell ne peut pas voir parce que chaque mot pris isolément est valide.

Un dictionnaire utilisateur situé à `~/.config/frfix/dictionary.txt` est consulté en premier. Tout ce qui s'y trouve est traité comme un mot français valide et n'est jamais corrigé. Mettez-y les noms propres, les noms de marques et le jargon.

### 5. Injecter la correction avec `xdotool`

Quand une correction se déclenche, frfix synthétise des événements de saisie via `xdotool` :

1. `xdotool key BackSpace` × `len(original)` pour supprimer ce que l'utilisateur a tapé.
2. `xdotool type --` *remplacement* pour taper la correction.

`xdotool` fonctionne uniquement sous X11. Il parle XTEST au serveur X. L'injection atterrit dans la fenêtre qui a actuellement le focus clavier, exactement comme si l'utilisateur l'avait tapée. L'application qui reçoit les touches ne peut pas faire la différence entre un humain et `xdotool`, ce qui constitue à la fois la force et le point faible de cette conception.

## Pourquoi X11 uniquement (et pas Wayland)

Deux couches de la conception requièrent des capacités que Wayland retire délibérément :

- **Capture globale des frappes.** Sous Wayland, seule l'application ayant le focus reçoit les événements clavier. Il n'existe aucun équivalent à la lecture de `/dev/input` au niveau du protocole (vous pouvez toujours le faire au niveau du périphérique, mais la traduction du layout casse, car il n'y a pas d'équivalent à la requête `xkb`). C'est une fonctionnalité de sécurité, pas un bug.
- **Injection globale des frappes.** Wayland n'a aucun équivalent à XTEST. `xdotool` ne fonctionne pas. Des outils comme `ydotool` fonctionnent via `/dev/uinput`, mais la plupart des compositeurs les empêchent d'injecter dans les surfaces ayant le focus des autres applications.

`frfix` s'exécute donc **uniquement sous X11**. Si vous êtes sur un bureau GNOME-sur-Wayland, basculez votre session vers Xorg à l'écran de connexion (l'icône d'engrenage sur l'invite de connexion).

Ce n'est pas un état temporaire. Le chemin le plus propre vers la prise en charge de Wayland est un portail. Un portail de bureau XDG pour « la correction de saisie à l'échelle du système » n'existe pas encore, et ne devrait probablement pas exister, pour les mêmes raisons qui font que Wayland n'expose pas de primitives de keylogging au départ. Pour l'instant : X11.

## Sécurité : les secrets que vous tapez doivent le rester

Un démon qui lit chaque touche que vous tapez est, par construction, un keylogger avec quelques étapes en plus. Trois règles gouvernent la conception :

1. **Rien n'est jamais persisté.** Les frappes, les mots et les phrases vivent en mémoire dans un petit tampon circulaire, et seules les dernières corrections sont conservées (pour l'annulation `Ctrl+Z`). Rien n'est écrit sur le disque. Rien n'est envoyé sur le réseau. Il n'y a aucune télémétrie, aucun analytics, aucune « statistique d'usage anonyme ».
2. **Exclusions par défaut.** La configuration par défaut exclut les gestionnaires de mots de passe (`keepassxc`, `1password`, `bitwarden`) par classe d'application, ainsi que toute fenêtre dont le titre correspond à `password`, `mot de passe` ou `sudo`. Quand la fenêtre au focus correspond à une exclusion, frfix ne met même pas les frappes en tampon ; elles passent sans être touchées.
3. **Exclusions extensibles par l'utilisateur.** Si vous tapez des secrets dans une application inhabituelle, ajoutez-la à `exclusions.apps` ou `exclusions.window_titles` dans la configuration. La vérification d'exclusion a lieu *avant* que quoi que ce soit ne soit mis en tampon ou comparé au dictionnaire.

Ce modèle de confiance est le prix de cette conception. Si vous ne pouvez pas accepter un démon avec un accès en lecture à `/dev/input`, ne l'exécutez pas. C'est le même modèle de confiance que tout outil d'enregistrement d'écran, tout enregistreur de macros et tout utilitaire d'accessibilité sous Linux, mais cela mérite d'être dit clairement.

## Installation et lancement

```bash
git clone https://github.com/lemmebee/frfix.git
cd frfix
bash setup.sh
```

`setup.sh` installe les paquets système (`hunspell`, `gobject-introspection`, l'outillage venv de Python, `xdotool`), ajoute votre utilisateur au groupe `input`, crée un environnement virtuel et installe une unité systemd utilisateur. Si vous venez d'être ajouté à `input`, déconnectez-vous puis reconnectez-vous avant de lancer.

Ensuite :

```bash
# foreground, interactive
.venv/bin/frfix

# verbose: see keystrokes, candidate words, correction decisions
.venv/bin/frfix --debug

# as a systemd user service, auto-start on login
systemctl --user enable --now frfix
journalctl --user -u frfix -f
```

Le flag `--debug` est la chose la plus utile quand quelque chose ne fonctionne pas. Il affiche chaque frappe reçue, chaque frontière de mot détectée et chaque décision de correction. Si des mots apparaissent mais que les corrections ne se déclenchent jamais, c'est que hunspell n'est pas installé ou que la vérification de layout rejette votre layout courant. Si aucune frappe n'apparaît du tout, c'est que vous n'êtes pas dans le groupe `input` (ou que vous ne vous êtes pas déconnecté puis reconnecté depuis l'ajout).

## Configuration

La configuration se trouve à `~/.config/frfix/frfix.toml`, créée automatiquement au premier lancement :

```toml
[general]
enabled = true

[corrections]
spelling = true     # word-level: accents, typos, elisions
grammar  = true     # sentence-level grammalecte rules

[overlay]
enabled       = true
duration_ms   = 1500
bg_color      = "#1a1a2e"
text_color    = "#e0e0e0"
highlight_color = "#4ecca3"

[exclusions]
apps          = ["keepassxc", "1password", "bitwarden"]
window_titles = ["password", "mot de passe", "sudo"]
```

Un petit overlay GTK (toast) affiche brièvement la paire original → corrigé pour que vous *appreniez* vraiment ce que vous avez mal tapé, au lieu que frfix lisse silencieusement la même faute pour la 500e fois. Désactivez-le en fixant `[overlay].enabled = false`.

Le dictionnaire utilisateur (`~/.config/frfix/dictionary.txt`) est à raison d'un mot par ligne. Tout ce qui s'y trouve est traité comme un mot français valide. Utilisez-le pour les noms propres, les noms de marques, le jargon, la terminologie interne à l'entreprise.

## Limites et compromis assumés

- **X11 uniquement.** Déjà abordé. Wayland n'est structurellement pas pris en charge.
- **Français uniquement.** Toute la couche de correction est bâtie autour de hunspell-fr et de grammalecte. La prise en charge multilingue impliquerait de détecter la langue de chaque mot, un problème difficile et explicitement hors périmètre.
- **L'injection via xdotool est sensible au timing.** Certaines applications (surtout Electron et certains multiplexeurs de terminal) avalent ou réordonnent occasionnellement les événements injectés. Quand ça tourne mal, vous obtenez un mot à moitié corrigé ; `Ctrl+Z` l'annule.
- **Aucune compréhension sémantique.** Les règles de grammalecte sont bonnes mais pas parfaites. Les phrases avec des constructions rares peuvent déclencher des faux positifs. Ajoutez des corrections à votre dictionnaire utilisateur ; si une règle de grammaire particulière se déclenche à tort de façon répétée, désactivez `[corrections].grammar`.
- **Qualité alpha.** Le correcteur est conservateur à dessein. Il reste des classes de fautes qu'il n'attrape pas encore, en particulier les élisions complexes (`s'il` vs `si il`) et les mots composés.

Le but n'est pas d'être parfait. Le but est de corriger les 95 % de fautes triviales que *tout* francophone qui tape fait *chaque* jour, pour que le budget cognitif aille au contenu réel.

## La suite

Les éléments que j'ai le plus envie d'étendre :

- **Meilleur scoring des candidats.** Actuellement, la première suggestion plausible de hunspell l'emporte. Un scorer pondéré par la fréquence sur un corpus français ferait mieux.
- **Comportement par application.** Désactiver la grammaire dans les terminaux (où vous tapez des commandes, pas de la prose), activer un mode strict dans les éditeurs.
- **Statistiques.** Locales uniquement. Quelles classes de fautes l'utilisateur fait-il le plus ? Strictement sur opt-in, ne quitte jamais la machine.

Si vous écrivez en français tous les jours sous Linux et que l'incohérence de la correction orthographique d'une application à l'autre vous agace autant qu'elle m'agace, essayez-le. Les PR sont les bienvenues.

Source : [github.com/lemmebee/frfix](https://github.com/lemmebee/frfix). MIT.
