---
sidebar_position: 2
---

# Análisis inicial

- [Análisis inicial](#análisis-inicial)
  - [Probar en local](#probar-en-local)
  - [Tecnologías](#tecnologías)
  - [Páginas](#páginas)
    - [Tablas](#tablas)
    - [Vistas tipo diagrama](#vistas-tipo-diagrama)
    - [Vistas formulario / wizard](#vistas-formulario--wizard)


## Probar en local

PARA PROBAR EL REPO EN LOCAL:

2 - entrar a url
http://localhost:3000/workflows?instance_url={URL_ENTORNO}&token={TOKEN_ENTORNO}

ejemplo:
http://localhost:3000/workflows?instance_url=https://qa.flows.ninja&token=eyJhbGciOiJIUzI1NiJ9.eyJ1c2VybmFtZSI6Im1vZGVsc2FkbWluIiwic3VwZXIiOnRydWUsImVtYWlsIjoiZG1vcm9uQGFpcmZsb3dzLmNvbSIsImFncmVlbWVudEFjY2VwdGVkIjp0cnVlLCJpc3N1ZWRBdCI6MTc1MjgzMTMwNjMzOX0.-hyizImoX87PUShksFb1UCyNLfxKp6M7NLQnaUP-D2M

(el usuario debe tener permiso en la tabla Process, *dmoron* no vale)

Lo primero que se carga es el dashboard de workflows (`app/page.tsx`)


---
## Tecnologías 
- Framework: Next.js app-router, React 18, TypeScript, Tailwind + shadcn/Radix primitives for UI.
- React-Flow-driven para editar los workflows (components/workflow/*).
- Data layer: wrappers pequeñitos que hacen llamadas fetch.
- Styling: Todo está con Tailwind, nada de Material.
- Build: PostCSS, Tailwind, ts-config tuned for Next.


## Páginas 
### Tablas

| página & componente raíz | categoría | dependencias |
| --- | --- | --- |
| **lista de workflows**<br/>`app/page` → `process-chart` | molecules | process-form · page-container · data-table · process-columns |
| | ui | button · separator · dialog |
| | services | createProcess · fetchProcessListWithoutLimit |
| | util | cookies |
| **lista de tareas**<br/>`app/task/page` → `task-chart` | molecules | page-container · data-table · task-columns |
| | ui | separator |
| | services | assignUserTask · fetchUserTask · fetchUserGroupsTask |
| | util | cookies |
| **lista de instancias de un workflow**<br/>`app/instance/page` → `process-instance-chart` | molecules | page-container · data-table-instance · process-instance-columns |
| | ui | button · separator |
| | services | fetchProcessInstance · fetchProcessDetail |
| | util | cookies |

### Vistas tipo diagrama

| página & componente raíz | categoría | dependencias |
| --- | --- | --- |
| **vista de un workflow (proceso)**<br/>`app/process/page` → `workflow-chart` | molecules | xyflow · activity-node · and-node · end-node |
| | services | fetchData · updateProcessDetail |
| | util | cookies · generateUniqueId · toast |
| **vista de una instancia de un workflow (proceso)**<br/>`app/instance/detail/page` → `detail-process-instance-chart` | molecules | xyflow · activity-node · and-node · or-node · xor-node · function-node · ia-agent-node · end-node · start-node · custom-readonly-edge |
| | ui | button |
| | services | fetchProcessInstanceData · getProcessInstance |
| | util | cookies · getLayoutedElements |

### Vistas formulario / wizard

| página & componente raíz | categoría | dependencias |
| --- | --- | --- |
| **lanzador de instancia**<br/>`app/instance/launch/page` → `variable-form` | molecules | data-table · variable-instance-columns |
| | ui | button · card · form · input · modal · checkbox · datetimePicker |
| | services | createFullProcessInstance · fetchProcessDetail · fechVariableData · createVariableData · deleteVariableData · updateVariableData |
| | util | cookies · customDateFormater · toast |


