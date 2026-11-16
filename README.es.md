# Skills de OSCompliance ENS y Power ENS

Skills para agentes que despliegan [OSCompliance ENS](https://github.com/arseniteio/oscompliance-ens) y [Power ENS](https://github.com/arseniteio/power-ens) sobre Wazuh y llevan las máquinas Linux al cumplimiento del ENS solo mediante configuración. Funcionan con Claude Code y OpenCode.

## Cómo funciona

Pide a tu agente que despliegue Power ENS. Primero pregunta lo que necesita: la tarea, los hosts de Wazuh y cómo llegar a ellos, las máquinas que se auditan, la categoría ENS y la ventana de reinicio. Después identifica por SSH el rol y la versión de cada host de Wazuh, presenta un plan y espera tu confirmación. Ejecuta el instalador de la release una vez por host, en orden: el manager maestro, los workers, los dashboards y, por último, las máquinas auditadas. Comprueba cada host y termina con un informe.

Después pídele que suba el cumplimiento. Lee los veredictos, separa los fallos reales del sistema de la configuración que falta o es incorrecta, y propone valores de parámetros o exclusiones con motivo. Solo aplica lo que confirmas y registra el porqué de cada cambio. Nunca modifica scripts ni sistemas sin tu permiso, y nunca cambia un valor solo para poner un veredicto en verde.

Las contraseñas nunca pasan por el agente. El instalador las pide en el manager maestro: escríbelas en tu propio terminal cuando el agente te dé el comando, o expórtalas antes en esa sesión. El agente nunca las pide en la conversación.

## Instalación

### Claude Code

```
/plugin install power-ens --marketplace arseniteio/skill-power-ens
```

En versiones de Claude Code anteriores a la 2.1.275, añade antes el marketplace:

```
/plugin marketplace add arseniteio/skill-power-ens
/plugin install power-ens@arseniteio
```

Actualiza con `/plugin marketplace update arseniteio`.

### OpenCode

Clona el repositorio en el directorio de skills de OpenCode. OpenCode encuentra las dos skills dentro.

```bash
git clone https://github.com/arseniteio/skill-power-ens.git ~/.config/opencode/skills/power-ens
```

Para un solo proyecto, clónalo en `.opencode/skills/power-ens` en la raíz del proyecto. Actualiza con `git -C ~/.config/opencode/skills/power-ens pull`. Si `opencode.json` restringe las skills con el permiso `skill`, permite `power-ens-*`.

### Comprobación

Abre una sesión nueva y pregunta: «¿Qué skills de Power ENS tienes?». El agente lista power-ens-deploy y power-ens-compliance.

## Skills

- power-ens-deploy instala, actualiza o desinstala OSCompliance ENS y Power ENS en Wazuh 4.14.7 a 4.14.9, todo en uno, en Docker o distribuido, y añade máquinas a la auditoría.
- power-ens-compliance lee los veredictos y declara parámetros de control y exclusiones por control con motivo.

Las skills se cargan solas cuando una petición encaja. Para cargar una de forma explícita, usa `/power-ens:power-ens-deploy` en Claude Code o `@power-ens-deploy` en OpenCode.

## Licencia

AGPL-3.0-only. Consulta [LICENSE](LICENSE).
