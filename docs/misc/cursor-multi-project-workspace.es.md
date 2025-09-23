## gestionar múltiples proyectos en un solo workspace de Cursor

Tu workspace contiene varios proyectos, por ejemplo:
- `/home/rderandom/dev/airflows-uno`
- `/home/rderandom/dev/airflows-admin-tool`

Cursor trata cada carpeta de primer nivel como un proyecto separado mientras indexa todo el workspace conjuntamente.

- **explorador**: Cada proyecto aparece como carpeta raíz. Las acciones de fichero se limitan a esa carpeta.

- **búsqueda y apertura rápida**: Global por defecto en todas las carpetas. Filtra por prefijo de carpeta en la consulta (p.ej., `airflows-admin-tool src/components Button`).

- **git**: Cada carpeta mantiene su propio repo. La vista de control de código muestra múltiples repos; haz stage/commit/push por repositorio.

- **terminal**: Las terminales nuevas arrancan en una carpeta (típicamente la primera que abriste). Abre una terminal en la carpeta objetivo o haz `cd` a ella. Prefiere rutas absolutas para evitar ambigüedades.

- **tareas/scripts**: Los scripts de paquete se ejecutan por proyecto. Lánzalos desde una terminal en la carpeta correcta, o pasa una ruta explícita. Ejemplos:

```bash
pnpm -C /home/rderandom/dev/airflows-admin-tool test:nowatch
npm run --prefix /home/rderandom/dev/airflows-admin-tool test:nowatch
```

- **linters/typecheck/servidores de lenguaje**: Se ejecutan por proyecto usando las configs más cercanas (p.ej., `tsconfig.json`, `package.json`, ESLint). El panel de Problemas agrega diagnósticos de todas las carpetas; filtra por carpeta si es necesario.

- **configuraciones de depuración/ejecución**: Mantén configs por proyecto (p.ej., `.vscode/launch.json` dentro de cada proyecto). Inicia la configuración que corresponde al proyecto activo.

- **contexto y comandos de IA**: La IA usa el fichero activo y su proyecto como contexto principal, pero puede leer todo el workspace. Al pedir que edite/ejecute algo, especifica el proyecto o la ruta absoluta para evitar ambigüedades.

- **ignores y rendimiento**: Usa un `.cursorignore` a nivel de workspace y ignores por proyecto para excluir rutas pesadas o irrelevantes y mejorar el rendimiento de búsqueda/IA.

Consejo: Si las tareas se sienten lentas o los contextos chocan, abre cada repositorio en su propia ventana de Cursor para aislar.
