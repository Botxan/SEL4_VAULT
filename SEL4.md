- [Whitepaper](https://sel4.systems/About/seL4-whitepaper.pdf)
- [FAQ](https://docs.sel4.systems/projects/sel4/frequently-asked-questions.html)
- [Supported platforms](https://docs.sel4.systems/Hardware/)
- [SEL4 on Raspberry PI 3](https://research.csiro.au/tsblog/sel4-raspberry-pi-3/)

##### ¿Cuál es el tamaño de seL4?
Depende de la arquitectura del procesador. En el caso del [[Raspberry PI 4]], la arquitectura es Arm de v8 (64 bits) =>  16-18k SLOC.

- La verificación formal del 64-bit Arm (Arm v8 AArch64) está en progreso.

##### ¿Qué dispositivos soporta seL4?
Como cualquier otro microkernel, ejecuta todos los controladores en *user mode*, por tanto no es asunto del kernel. Las excepciones son:
- Controlador del temporizador, que seL4 necesita para hacer cumplir los tiempos de espera.
- Controlador de interrupción en el que seL4 media.
- Cuando se compila con la depuración activada, ell kernel también contiene un controlador de serie.

##### ¿DMA en seL4?
Las pruebas formales asumen que el DMA está desactivado, ya que los dispositivos controlados por el DMA puede sobreescribir la memoria de la máquina, incluidos los datos y código del kernel.

Puedes seguir utilizando los dispositivos DMA de forma segura, pero tienes que asegurarte por separado de que se comportan bien, es decir, que no sobrescriben el código del kernel o las estructuras de datos, y sólo escriben en los marcos que se les asignan de acuerdo con la política del sistema. En la práctica, esto significa que hay que confiar en los controladores y el hardware de los dispositivos DMA.

##### ¿Soporta seL4 multicore?
Sí, en x64 y en arm v7 (32-bit) y v8 (64-bit), pero su verificación está en progreso.
El kernel multicore usa un [big-lock](https://en.wikipedia.org/wiki/Giant_lock) approach, es decir, que hay un bloqueo cuando los threads entran al espacio del kernel, y desbloqueo cuando salen. El caso más típico es el de las *system calls*. En este modelo, los hilos del espacio del usuario pueden ejecutarse de forma concurrente entre los procesadores (o cores) disponibles, pero no más de un hilo puede ejecutarse al mismo tiempo en el espacio del kernel. Eliminando la concurrrencia del espacio del kernel, este no necesita implementar soporte para sistemas SMP, lo cual reduce la complejidad pero evidentemente también la eficiencia. Los sistemas operativos de hoy en día utilizan [bloqueo de grano fino](https://en.wikipedia.org/wiki/Lock_(computer_science)#Granularity).

##### ¿Para que tipo de aplicaciones está pensado seL4?
Al ser un microkernel de propósito general, así que la respuesta es para cualquier tipo de aplicación. Los principales objetivos son los sistemas integrados con requisitos de seguridad o fiabilidad, pero eso no es exclusivo.

##### ¿Se puede ejecutar Linux encima de seL4?
Sí, como máquina virtual, pero sólo en ARMv7 de momento de forma estable.
Para dar soporte a las máquinas virtuales, seL4 se ejecuta como un hipervisor y reenvía los eventos de virtualización a un monitor de máquina virtual (VMM) que realiza las emulaciones necesarias.


### Capabilities
- Para manejar permisos
- Estructura: referncia (puntero) a objeto + permisos de acceso.
- Todas las operaciones son autorizadas por una *capability*.
- Funcionamiento resumido:
1. Cuando se realiza una operación en un objeto (como por ejemplo enviar un mensaje a un *endpoint* IPC), la *capability* para el objeto es pasada al kernel.
2. El kernel comprueba que la *capabilty* reune los accesos suficientes para ejecutar la opeación. Si es así, se ejecuta sin necesidad de más preguntas.
- Por seguridad, están almacenadas en memoria kernel (también conocida como *Cnodes*), y el modo usuario puede referenciarlas a traves de referencias a ubicaciones *Cspace*, es decir, el lugar de memoria donde se guardan los *Cnodes*.

##### ¿Cómo maneja el usermode el acceso al espacio de memoria de kernel de forma segura?
La memoria libre se llama ***Untyped***.

El kernel pone al espacio del usuario al control de los recursos del sistema entregando toda la memoria *Untyped* tras el arranque del primer proceso de usuario (llamado ***root task***), depositando las respectivas *capabilities* en el *Cspace* del *root task*. La tarea raíz puede entonces implementar sus políticas de gestión de recursos, por ejemplo, partiendo el sistema en dominios de seguridad y entregando a cada dominio un subconjunto disjunto de memoria no tipificada.

Cuando una *system call* requiere de metadatos del kernel, como TCB de un hilo al crearlo, el espacio de usuario debe de proveer de este espacio de memoria explícitamente al kernel. Esto se consigue "reescribiendo" parte de la memoria *Untyped* en el tipo de objeto del kernel correspondiente. Por ejemplo, para la creación de hilos, el espacio del usuario debe reescribir memoria Untyped en *TCB Objects*. Al hacerlo, la memoria se convierte en memoria del kernel,en el sentido de que sólo el kernel puede leer o escribir en ese espacio. Aun así, el espacio de usuario puede revocar ese espacio de memoria, lo que implícitamente destruiría los objetos (e.g., threads) representados por el objeto.

Los únicos objetos directamente accesibles por el espacio del usuario son los *Frame Objects*. Estos se pueden mapear a un *Address Space Object* (en esencia a una tabla de páginas),  después de lo cual el espacio del usuario puede escribir en la memoria física representada por los *Frame Objects*.

##### ¿Cómo se comuncian los hilos?
Para mensajes cortos (unos pocos cientos de bytes) ---> IPC.
Para mensajes más largos: búferes de memoria compartida (sincronizados mediante *Notifications*).

##### ¿Cómo funciona el message-passing?
Sel4 utiliza "IPC sincronizado". Esto significa que el mensaje es intercambiado cuando tanto el emisor como el receptor están preparados. Si ambos se están ejecutando en el mismo core, significa que uno de los does se bloqueará hasta que el otro invoque la operación de IPC.

En el4, el IPC se realiza via *Endpoint Objects*. Un *Endpoint* se puede entender como un buzón por el que el emisor y el receptor intercambian mensajes por medio de un *handshake*. Cualquiera que tenga la *capability* de *Send* puede enviar mensajes por el *Endpoint*, y cualquiera que tenga la de *Receive* puede recibir mensajes. Esto significa que puede haber cualquier cantidad de emisores y receptores. En particular, un mensaje específico solamente es enviado a un receptor (el primero en la cola), sin importar cuantos hilos estén intentando recibir desde el *Endpoint*.

El *message broadcast* se implementa por encima de seL4.

##### ¿Por qué las operaciones *send-only* no devuelven una indicación de éxito?
Algunas IPC *system calls* como "seL4_Send()" o "seL_NBSend()" tienen la *capability send-only*, es decir, transferencia de datos en una única dirección. Un código de retorno implicaría un canal de retorno, y eso violaría las garantías de flujo de información ofrecidas por seL4, permitiendo flujo de información que no está explícitamente autorizado por una *capability*.

##### ¿Qué son las *Notifications*?

Un *Notification Object* es un pequeño array de semáforos binarios, con las mismas operaciones *Signal* y *Wait*. Debido a la naturaleza binaria, multiples *Signals* pueden ser perdidas si no están intercaladas por *Waits*.

Utilizar *Signal* en una notificación requiere la *cap Send* en el "Notification object". La *cap* tiene una "insignia". que consiste en un patrón de bits definido por el creador de la *cap* (y normalmente el propietario del *Notification Object*). La operación Signal realiza una operación bitwise OR en la matríz de bits del objeto *Notificación*. La opción *Wait* bloquea hasta que la matriz es distinto de cero, y después devuelve la cadena de bits y ceros fuera del array.

Las *Notifications* también pueden ser "Encuestadas", que es como un *Wait*, excepto porque la operacíon no es bloqueante, y en cambio devuelve cero inmediatamente, incluso si la cadena de bits *Notification* es cero.