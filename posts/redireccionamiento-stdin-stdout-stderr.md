# Redirección de FD

Conocemos los descriptores de Linux así que vamos a juguetear un poco con ellos en caliente.

| Columna 1 | Columna 2 | 
|-----------|-----------|
| 0         | STDIN     |
| 1         | STDOUT     |
| 2         | STDERR     |

En Linux todo es un archivo, y la comunicación con el exterior la hacen a través de descriptores de fichero, vamos a verlo en terminal partiendo de un proceso con `PID = 213408`.

Todo es un archivo como he comentado, y toda la información que necesitamos ahora mismo de ese proceso vive en `/proc/PID` concretamente nos interesa ahora mismo `/proc/PID/fd` porque es donde están los file descriptors.

```bash
miguel@demo-ubuntu:~/tmp/std$ ls -lah /proc/213408/fd
total 0
dr-x------ 2 miguel miguel  4 may 12 17:05 .
dr-xr-xr-x 9 miguel miguel  0 may 12 17:05 ..
lrwx------ 1 miguel miguel 64 may 12 17:05 0 -> /dev/pts/15
lrwx------ 1 miguel miguel 64 may 12 17:08 1 -> /dev/pts/15
lrwx------ 1 miguel miguel 64 may 12 17:08 2 -> /dev/pts/15
lr-x------ 1 miguel miguel 64 may 12 17:08 255 -> /home/miguel/tmp/std/test.sh
```

Nos olvidamos por un momento del 255, pero podemos ver que 0, 1 y 2, es decir, STDIN, STDOUT y STDERR tienen una especie de enlace simbólico a `/dev/pts/15` ¿Qué es esto? Podemos ir a cualquier terminal y hacer:

```bash
miguel@demo-ubuntu:~/tmp/std$ tty
/dev/pts/19
```

aaamigo, ¿Todo es un archivo no? ya hemos encontrado que quiere decir entonces, parece ser que este proceso está lanzado en una terminal con tty 15, o al menos todo lo relacionado con él se relaciona con dicha terminal.

Es posible redireccionar hacia donde se dirige stdin, stdout y stderr. Pero para hacerlo no podemos directamente cambiar el enlace simbólico sino que tenemos que hacer uso de mecanismos avanzados.

Si ya te has perdido y no sabes de lo que hablamos te recomiendo leer <a href="/stdin-stdout-stderr">el post acerca de stdin-stdout-stderr</a>

Cómo he comentado no podemos simplemente cambiar el enlace simbólico a otra terminal:

```bash
miguel@demo-ubuntu:~/tmp/std$ ln -sf /proc/213408/fd/1 /dev/pts/17
ln: fallo al crear el enlace simbólico '/dev/pts/17': Permiso denegado
miguel@demo-ubuntu:~/tmp/std$ sudo ln -sf /proc/213408/fd/1 /dev/pts/17
ln: fallo al crear el enlace simbólico '/dev/pts/17': Operación no permitida
```

Pero vamos a hacerlo de otra forma...

---

## Redireccionando con GDB

Con GDB podemos hacerlo de la siguiente forma, en primer lugar tenemos que saber el PID del proceso del cual queremos cambiar su redirección, como ejemplo el siguiente script:

```bash
#!/bin/bash
echo "This is the pid: $$"
echo "This is stdout:"; sudo ls -l /proc/$$/fd/0;
n=0; while true; do ((n++)); echo $n; sleep 1; done
```

Este script simplemente nos va a mostrar su PID, la entrada estándar y luego entra en un bucle infinito contando números cada segundo, de hecho el proceso que estábamos mirando antes es justo este script.

Para poder modificar la salida de este script cogemos ese PID y abrimos gdb de la siguiente forma:

```bash
# abrimos gdb
miguel@demo-ubuntu:~/tmp/std$ sudo gdb

# Primero nos conectamos al proceso, una vez hacemos esto se para el proceso y nos permite ejecutar código en su espacio de direcciones. 
attach PID

# A continuación tenemos que abrir la consola destino en modo lectura y escritura
call open("/dev/pts/17", 66, 0666)

# Al hacerlo obtenemos un número que será al que llamemos a continuación para duplicarlo en el destino como primer número, el segundo número ponemos el 1 haciendo referencia a la salida es decir stdout
call dup2(3, 1)

# Por último dejar que siga funcionando y cerrar
detach
quit
```

Lo vemos en un pequeño vídeo

<localvideo src="/redireccionamiento-stdin-stdout-stderr/TerminalChange2.mp4" />

Sabiendo que esto es posible podríamos montar nuestro propio programita que haga esto mismo, o algo similar pero de forma automatizada, pero hoy no va a ser ese día, eso si, vamos a hacer algo un pelín más currado antes de acabar, no os dejo a medias. Luego ya la idea que tenga cada uno con lo que voy a enseñar no necesito saberla 😈

---

## Mirror de terminal

Entendiendo como funciona y aplicando algunos conceptos extra podríamos hacer un mirror, es decir, poder ver la información de lo que ocurre en otra terminal, en este caso vamos a hacerlo de un proceso único, y verla en tiempo real en nuestra propia terminal. ¿Quizás dos personas conectadas con el mismo usuario a la misma máquina?...

Aquí no nos vale únicamente con el dup tal cual, porque al hacer el dup perdemos la referencia y no nos permite enviar a dos terminal al mismo tiempo, el STDOUT es único. Tampoco sería viable crear una pipe, no porque no pudiese ser una opción teórica sino porque requeriríamos un puntero de memoria reservado previamente. Vamos a darle una vuelta más...

¿Y si usamos un FIFO? Un FIFO es una pipe pero con su propio nombre, otorgar un nombre ya sabemos que puede dar superpoderes. Un pipe normal solo puede usarse entre procesos relacionados, mientras que un FIFO persiste en el sistema y se puede utilizar por otros procesos. Su creación es fácil:

```bash
miguel@demo-ubuntu:~/tmp/std$ mkfifo /tmp/mirror
miguel@demo-ubuntu:~/tmp/std$ ls -lh /tmp/mirror
prw-rw-r-- 1 miguel miguel 0 may 12 19:37 /tmp/mirror
```

Perfecto, ahora teniendo esto nos vamos a GDB:

```bash
(gdb) call open("/tmp/mirror", 1, 0)
$4 = 4
(gdb) call dup2(4, 1)  
```

En realidad os estoy engañando, he tenido un problema intermedio...

Cuando hacemos `call open("/tmp/mirror_pipe", 1, 0)` el 1 hace referencia a solo escritura, pero al abrir un FIFO solo para escritura se bloquea la llamada hasta que otro proceso abra el FIFO como lectura. Un fix rápido es tener ya algo escuchando, abrirlo como lectura y escritura o si como yo lo teniais ya lanzado, podéis hacer un cat a /tmp/mirror y solucionado.

Y nos falta la última parte, utilizar ese FIFO para mostrar en dos terminales la información, para ello podemos utilizar tee, un binario que precisamente nos permite esto:

```bash
cat /tmp/mirror | tee /dev/pts/15 /dev/pts/17
```

Os muestro el vídeo acortando un poco el proceso que ya lo tenéis explicado:

<localvideo src="/redireccionamiento-stdin-stdout-stderr/Mirrorpipe.mp4" />

Aparte de la multitud de usos que puede tener esta información bien utilizada, simplemente estamos aprendiendo un poco más de los entresijos del maravilloso mundo de linux.
