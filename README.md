# Documentation officielle du projet ROK4

Sources du site de documentation et du générateur de site statique associé (Mkdocs).

## Démarrage rapide

### Guide de rédaction

[Cette page](https://squidfunk.github.io/mkdocs-material/reference/formatting/) donne des indications sur les capacités et spécificités de l'implémentation de la syntaxe Markdown.

### Prérequis

- Python >= 3.9

### Installation

Après avoir cloné ou téléchargé le dépôt, installer les prérequis :

```bash
python -m pip install -U -r requirements.txt
```

### Générer le site

```bash
mkdocs build
```

Avec une tolérance 0 aux erreurs :

```bash
mkdocs build --strict
```

### Servir le site en local

```bash
mkdocs serve
```

Avec rechargement à chaud sélectif :

```bash
mkdocs serve --dirtyreload
```

Le site est accessible en local à l'adresse suivante : <http://localhost:8000/>.
Quand un contenu est modifié, le site est automatiquement rechargé.
