Los pasos seguidos en este documento para el montaje de seL4 se realiza bajo la distribución Arch Linux.

# Construir dependencias
1. Instalar la herramienta [repo](https://source.android.com/setup/develop#installing-repo) de Google.
`$ pacman -S repo`
Verificar la correcta instalación:
`$ repo version`
Si `repo launcher version` informa 2.15 o un número superior, quiere decir que el número de versión es correcto y que la instalación es adecuada.

