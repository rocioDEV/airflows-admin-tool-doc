---
sidebar_position: 3
---

# Flujos

Discovery de flujos implementados actualmente en el frontal

## SignIn, Dashboard y refresh

![Dependencies Diagram](./img/signin.png)

Se extraen los siguienres requisitos funcionales:

- al cargar la aplicación en cualquier ruta, comprobamos si tenemos el modelo en el contexto
    - si tenemos modelo, renderizamos la ruta
    - si no lo tenemos, miramos a ver si tenemos guardado un token
        - si tenemos token, recuperamos el modelo del back y lo guardamos en el contexto
        - si no tenemos nada, vamos al login

        (TBC: añadir redirección IAM y seteo idioma del usuario)