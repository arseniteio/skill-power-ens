# skill-power-ens

Skill para agentes que instala OSCompliance ENS y Power ENS en Wazuh 4.14.7 a 4.14.9, audita máquinas Linux frente al ENS y las lleva al cumplimiento solo mediante configuración. Las instrucciones están en [SKILL.md](SKILL.md), en inglés. La skill sigue la documentación de [oscompliance-ens](https://github.com/arseniteio/oscompliance-ens) y [power-ens](https://github.com/arseniteio/power-ens).

El agente instala con el `install.sh` de la release, o con el paquete verificado cuando no se permite `curl | bash`. Solo cambia la configuración de Power ENS, las exclusiones por control con motivo y los ficheros de configuración del agente y del grupo que nombra la documentación. Nunca modifica scripts ni sistemas sin permiso, y registra el motivo de cada cambio.

## Claude Code

Clona el repositorio en una carpeta llamada `power-ens` dentro del directorio de skills. Para todos los proyectos del usuario:

```bash
git clone https://github.com/arseniteio/skill-power-ens.git ~/.claude/skills/power-ens
```

Para un solo proyecto, desde su raíz:

```bash
git clone https://github.com/arseniteio/skill-power-ens.git .claude/skills/power-ens
```

Abre una sesión nueva. Claude Code carga la skill cuando una petición encaja con su descripción. Para cargarla de forma explícita, escribe `/power-ens`. Actualízala con `git -C ~/.claude/skills/power-ens pull`.

## OpenCode

OpenCode lee los directorios de Claude Code anteriores, así que un solo clon sirve para las dos herramientas. Para tenerla en los directorios de OpenCode, usa la ruta global o la del proyecto:

```bash
git clone https://github.com/arseniteio/skill-power-ens.git ~/.config/opencode/skills/power-ens
```

```bash
git clone https://github.com/arseniteio/skill-power-ens.git .opencode/skills/power-ens
```

El modelo carga la skill con su herramienta `skill` cuando una petición encaja con la descripción. Para cargarla de forma explícita, escribe `@power-ens` en el prompt. Si `opencode.json` restringe las skills con el permiso `skill`, permite el recurso `power-ens`.

## Uso

Da al agente una shell de root o sudo en el servidor de Wazuh y en las máquinas que se auditan, y pídele que despliegue Power ENS. Primero pregunta el tipo de despliegue, las máquinas, la categoría ENS y la ventana de reinicio. Exporta las tres contraseñas de la guía de instalación en la shell antes de empezar; el agente nunca las pide en la conversación.

## Licencia

AGPL-3.0-only. Consulta [LICENSE](LICENSE).
