# skillful

Marketplace de Claude Code con una suite de skills reutilizables en múltiples proyectos.

## Estructura

```
.claude-plugin/
  marketplace.json        # Catálogo del marketplace
plugins/
  skillful/
    .claude-plugin/
      plugin.json         # Manifiesto del plugin
    skills/
      <nombre>/SKILL.md   # Una carpeta por skill
```

## Instalación

Desde una copia local:

```
/plugin marketplace add /ruta/a/skillful
/plugin install skillful@skillful
```

Desde GitHub (después de publicar el repo):

```
/plugin marketplace add <usuario>/skillful
/plugin install skillful@skillful
```

## Añadir una skill

1. Crea `plugins/skillful/skills/<nombre>/SKILL.md` con frontmatter `name` y `description`.
2. Sube la versión en `plugins/skillful/.claude-plugin/plugin.json`.
3. Ejecuta `claude plugin validate .` antes de hacer commit.

## Añadir otro plugin

Crea `plugins/<plugin>/` con su propio `.claude-plugin/plugin.json` y regístralo en `.claude-plugin/marketplace.json`.
