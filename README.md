# Pipeline DevOps – Infraestructura VoIP (Asterisk)

## Estrategia de ramificación: GitFlow adaptado a infraestructura

Elegimos **GitFlow** como modelo de ramificación para este proyecto porque el
repositorio no gestiona código de aplicación tradicional, sino **configuración
de infraestructura como código (IaC)**: archivos `pjsip.conf` y
`extensions.conf` de un servidor Asterisk en producción.

En este contexto, un cambio mal aplicado (un puerto SIP mal configurado, una
extensión de marcado rota) puede dejar sin servicio de telefonía a una
organización completa. GitFlow da la separación de responsabilidades
necesaria para ese escenario:

- **`main`**: refleja el estado de configuración actualmente desplegado en el
  servidor de producción. Nunca se edita directo.
- **`develop`**: rama de integración donde se validan los cambios antes de
  pasar a producción (equivalente a un entorno de staging/pre-prod).
- **`feature/<nombre>`**: cambios planificados y no urgentes (ej. agregar una
  nueva extensión, un nuevo anexo). Se ramifican desde `develop` y vuelven a
  `develop` vía Pull Request.
- **`hotfix/<nombre>`**: correcciones urgentes sobre configuración ya en
  producción (ej. un puerto SIP mal definido que corta el servicio). Se
  ramifican desde `main` y se mergean tanto a `main` como a `develop`.

Se descartó **trunk-based development** porque ese modelo asume ciclos de
integración muy rápidos y tolerancia alta a arreglar errores directo sobre la
rama principal — algo riesgoso cuando la "rama principal" representa un
sistema de telefonía en producción, donde un error no revisado puede
significar caída de servicio real, no solo un bug de software.

## Estructura de ramas del proyecto

| Rama | Origen | Destino merge | Propósito |
|---|---|---|---|
| `main` | — | — | Estado en producción |
| `develop` | `main` | `main` | Integración/staging |
| `feature/anexo-100` | `develop` | `develop` | Nuevo anexo interno |
| `feature/dialplan-basico` | `develop` | `develop` | Plan de marcado básico |
| `hotfix/puerto-sip` | `main` | `main` + `develop` | Corrección urgente puerto SIP |

## CI/CD

Se configuró un workflow de GitHub Actions (`.github/workflows/ci.yml`) que
se ejecuta automáticamente en:
- `push` a `develop`
- `pull_request` hacia `main`

El workflow valida que los archivos `pjsip.conf` y `extensions.conf` existan
y que su sintaxis de secciones (`[nombre]`) esté correctamente balanceada,
como primera línea de defensa antes de fusionar cambios a producción.

## Guía de buenas prácticas

### Naming de ramas

| Prefijo | Uso | Ejemplo |
|---|---|---|
| `feature/` | Nueva funcionalidad o cambio planificado, no urgente | `feature/anexo-100` |
| `hotfix/` | Corrección urgente sobre producción (`main`) | `hotfix/puerto-sip` |
| `release/` | (opcional, si se usa) preparación de una versión antes de pasar a `main` | `release/1.1.0` |

Reglas:
- Todo en minúsculas, separado por guiones (`kebab-case`).
- El nombre después del prefijo debe describir el cambio, no la persona ni la fecha (`feature/anexo-100`, no `feature/juan-cambio1`).
- No se trabaja directo sobre `main` ni `develop`: todo cambio pasa por una rama con prefijo y un Pull Request.

### Mensajes de commit

Se usa el formato de **Conventional Commits**:

```
<tipo>(<alcance opcional>): <descripción corta en modo imperativo>
```

Tipos usados en este proyecto:
- `feat`: nueva funcionalidad (ej. `feat(pjsip): agrega anexo interno 100`)
- `fix`: corrección de un error (ej. `fix(pjsip): corrige puerto de transporte UDP`)
- `docs`: cambios de documentación (README, wiki)
- `ci`: cambios en pipelines/workflows de automatización
- `chore`: tareas de mantenimiento sin impacto en configuración funcional

Reglas:
- Descripción corta (idealmente <72 caracteres), en presente/imperativo: "agrega", "corrige", no "agregado" ni "agregando".
- Un commit = un cambio lógico. No mezclar un `feat` con un `fix` en el mismo commit.
- El cuerpo del commit (opcional, líneas siguientes) se usa para explicar el *porqué* del cambio, no solo el *qué*.

### Estructura de carpetas

```
asterisk-devops-ep1/
├── pjsip.conf              # Configuración de endpoints, auth y transporte SIP
├── extensions.conf         # Plan de marcado (dialplan)
├── .github/
│   └── workflows/
│       └── ci.yml          # Pipeline de validación automática
└── README.md                # Documentación y guía de buenas prácticas
```

A futuro, si el proyecto crece, se recomienda separar por entorno
(`environments/prod/`, `environments/staging/`) para evitar mezclar
configuración de distintos entornos en los mismos archivos.

### Control de versiones y flujo de revisión

- Ningún cambio se sube directo a `main` o `develop`: siempre vía Pull Request, aunque el equipo sea pequeño o el trabajo individual.
- Todo PR debe describir brevemente **qué cambia** y **por qué**.
- Los `hotfix/*` se mergean primero a `main` (para restablecer producción cuanto antes) y luego se sincronizan a `develop`, evitando que el fix se pierda en el siguiente release.
- Antes de mergear, se revisa que el workflow de CI (`ci.yml`) haya pasado en verde.
- Conflictos de merge se resuelven revisando manualmente ambas versiones del bloque en conflicto, priorizando la versión que refleje el estado correcto y vigente de la configuración (nunca aceptar "ambos cambios" sin revisar, ya que en archivos `.conf` eso puede duplicar secciones y romper el servicio).


## Conclusiones

Lo que más me costó entender al principio fue el conflicto de merge, principalmente porque no tenía tan claro qué hacía Git cuando dos ramas modificaban las mismas líneas de un archivo. Al principio pensaba que Git simplemente iba a elegir automáticamente cuál versión dejar, pero entendí que cuando no puede determinar cuál cambio conservar, tengo que resolver el conflicto manualmente y después hacer el commit correspondiente. También me costó un poco entender la estructura del YAML de GitHub Actions, especialmente la relación entre los workflows, los jobs y los steps, porque la sintaxis es bastante específica y un error pequeño puede hacer que el proceso falle.

Sobre el flujo de ramas, ahora entiendo mejor la lógica de separar main y develop. Antes pensaba que era innecesario tener varias ramas si finalmente todo terminaba en main, pero ahora veo que sirve para mantener una versión estable mientras se desarrollan y prueban cambios en otra rama. Esto permite reducir el riesgo de subir directamente a producción algo que todavía no ha sido probado.

También cambió mi forma de ver los Pull Requests. Antes los asociaba principalmente a equipos grandes donde varias personas tienen que revisar el código, pero ahora entiendo que también tienen utilidad cuando uno trabaja solo. Un PR permite tener un punto de control antes de integrar los cambios, revisar lo que se hizo, dejar evidencia y mantener un historial más ordenado. En un contexto laboral, además, facilita que otra persona pueda revisar los cambios antes de aplicarlos.

Relacionándolo con mi trabajo como sysadmin, creo que este flujo tiene bastante aplicación real. En administración de sistemas muchas veces trabajamos con configuraciones que pueden afectar servidores, redes o servicios completos. Por ejemplo, si tengo configuraciones de Ansible, archivos YAML de Docker, configuraciones de servicios o scripts para automatizar tareas, podría mantenerlas versionadas en Git. En vez de modificar directamente la configuración que está funcionando, podría crear una rama, realizar los cambios, probarlos y después hacer un Pull Request para revisarlos antes de integrarlos.

También veo una conexión importante con la infraestructura como código (IaC). Si las configuraciones de infraestructura están almacenadas en un repositorio, Git permite saber quién hizo un cambio, cuándo lo hizo y qué se modificó. Además, GitHub Actions puede utilizarse para automatizar validaciones, pruebas o incluso procesos de despliegue.

En conclusión, el encargo me ayudó a entender que Git y GitHub no son solamente herramientas para programadores. En el área de conectividad y redes también pueden ser muy útiles para administrar configuraciones, automatizar tareas y reducir errores humanos. Especialmente trabajando como sysadmin, creo que acostumbrarse a trabajar con ramas, PRs, revisiones y automatización puede hacer que los cambios de infraestructura sean mucho más controlados, trazables y seguros.
