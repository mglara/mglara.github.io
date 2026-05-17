# Mini-Sandbox

Es un placer que me acompañes un día más, lo que vamos a tratar hoy realmente tiene de base el siguiente artículo [Trazando Syscalls](/blog/trazando-syscalls) te recomiendo que te pases por ahí antes de comenzar.

## Introducción

Recordando donde nos habíamos quedado, estábamos justo con un mini strace funcionando, todavía con muchas lagunas y no se podría convertir en un programa productivo y adecuado, pero suficiente para seguir toqueteando un poco a Linux.

El plan de hoy es hacer un mini entorno sandbox, quizás llamarlo sandbox es venirnos muy arriba, pero lo que vamos a hacer es que cualquier programa pudiese ejecutarse a través de nuestro binario, pero controlemos las acciones que va a realizar.

Cómo te digo, no te esperes algo productivo y que vas a utilizar, y de hecho la forma en la que vamos a hacerlo ya no es la actual, pero es igualmente importante entender esta base, así que vamos con ello.

## Fase 0, matamos el proceso

¿Qué vamos a hacer? Definir una syscall prohibida sin hacer grandes cambios en nuestro código, recuerda que puedes ver cómo quedó el código en [Trazando Syscalls](/blog/trazando-syscalls).

```c
#include <stdbool.h>

struct syscall_entry {
    long        number;
    const char *name;
    int         str_arg;
    bool        blacklist; /*Un booleano 0 no está blacklist y 1 está blacklist*/
};

static const struct syscall_entry syscall_table[] = {
    {   0, "read",   0, 0 },
    {   1, "write",  2, 0 },
    {   3, "close",  0, 0 },
    {   5, "fstat",  0, 0 },
    {   9, "mmap",   0, 0 },
    {  12, "brk",    0, 0 },
    {  59, "execve", 1, 1 }, 
    { 257, "openat", 2, 0 }, 
};
```

En primer lugar, la forma más rápida que se me ha ocurrido sin hacer grandes cambios en el código, que no tiene porque ser la mejor, es la que estás viendo. Añadimos a nuestra estructura un booleano, no necesitamos más, si está en 0 pasa, si está en 1 está blacklisted la syscall.

He blacklisteado para esta prueba execve, ¿Por qué me preguntas?, ¿Por qué no? Te respondo. Ahora en serio, execve permite ejecutar, en ciertos entornos o para ciertas cosas quizás es justamente lo que queremos evitar. 

> [!DANGER]
> OJO no puedes hacer esto en todo el sistema que te lo cargarías.

```c
if (!entry) {
    fprintf(stderr, "syscall_%ld(0x%llx)", nr, regs.rdi);
} else if (entry->blacklist == 1) {
    kill(child, SIGKILL);
    fprintf(stderr, "%s(\"%s\")", entry->name, str);
    fprintf(stderr, "\nBLOCKED\n");
    return 0;
```

Y sólo tenemos que añadir un nuevo else if, este va a comprobar el valor de blacklist y si está en True, entonces va a matar directamente el proceso `kill(child, SIGKILL)`, a lo bestia, no es la opción más adecuada, pero es efectiva, y `return 0` lo usamos para que el padre también cierre directamente, veamos el resultado:

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace env ls
brk(0x0) = 94961438253056
mmap(0x0) = 133098360590336
syscall_21(0x790d608465b0) = -2
...
syscall_302(0x0) = 0
syscall_11(0x790d607fb000) = 0
syscall_318(0x790d6060a178) = 8
brk(0x0) = 94961438253056
brk(0x565deecd7000) = 94961438388224
openat(-100, "/usr/lib/locale/locale-archive") = 3
fstat(0x3) = 0
mmap(0x0) = 133098341662720
close(0x3) = 0
execve("/usr/lib/locale/locale-archive")
BLOCKED
```
Parece que funciona, me vale por ahora...

## Fase 1, Fallo silencioso

Vamos a complicarlo un poco más, no nos vale con bloquear como antes. En este caso lo que queremos hacer es seleccionar nosotros el error que queremos que devuelva.

```c
#include <errno.h>
```
Recuerda incluir la librería de errores, así podemos llamarlos directamente.

```c
...
int in_syscall = 0;
bool sandbox_blocked = 0;

  while (1) {
...
```
Justo antes del while vamos a añadir por comodidad un booleano que nos dice cuando hemos decidido bloquear la syscall, ahora verás su uso.

```c
if (!entry) {
    fprintf(stderr, "syscall_%ld(0x%llx)", nr, regs.rdi);
} else if (entry->blacklist == 1) { 
    fprintf(stderr, "%s(0x%llx)", entry->name, regs.rdi);
    fprintf(stderr, "BLOCKED");
    regs.orig_rax = -1; 
    ptrace(PTRACE_SETREGS, child, 0, &regs);
    sandbox_blocked = 1;   
```
He cambiado algunas cosas, para empezar no me quiero complicar y no vamos a imprimir los argumentos basándonos en nuestro array. Por otro lado si que quiero que me avise con `BLOCKED` y ahora la primera parte interesante, antes matábamos directamente el proceso, ahora solo vamos a engañarle, lo explico:

`reg.orig_rax = -1` seguido de `ptrace(PTRACE_SETREGS, child, 0, &regs);` lo que está haciendo es cambiar el rax, es decir, estamos modificando la llamada por un syscall inválido antes de lo que vamos a hacer después. ¿Por qué? Porque de esta forma lo que se consigue es que si por ejemplo lo que estamos bloqueando es la escritura no llegue a escribir antes de darle el código de error que queremos.

Recapitulemos unos pasos atrás, las syscalls tenían su entrada, llegan al kernel, hacen su acción, y pasan a la salida, por eso en [Trazando Syscalls](/blog/trazando-syscalls) al principio teníamos los outputs duplicados. Quiere decir que si bloqueamos algo que realice acciones o escritura simplemente tocando la salida aunque podamos "para el output" la acción se ha realizado. Precisamente el enviarlo a una syscall inválida es nuestra salvaguarda contra este hecho.

Y por supuesto establecemos nuestro booleano de bloqueo a 1, para que lo sepa la salida de la syscall.

```c
if (sandbox_blocked) {
    regs.rax = -EACCES;                
    ptrace(PTRACE_SETREGS, child, 0, &regs);
    sandbox_blocked = 0;
}
```

Y nos queda enviar nuestro error de salida, en este caso es tan sencillo como modificar el registro rax con el error que queramos, te pongo ahora output con `-EACCES` y con `-EPERM` para que se vea la diferencia.

**Output con -EACCES**

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace cat /etc/passwd
brk(0x0) = 104202677182464
mmap(0x0) = 125737236832256
writev(0x725b7b59b5b0) = -2
openat(0xffffff9c)BLOCKED = -38
openat(0xffffff9c)BLOCKED = -38
newfstatat(0xffffff9c) = -2
openat(0xffffff9c)BLOCKED = -38
newfstatat(0xffffff9c) = -2
openat(0xffffff9c)BLOCKED = -38
newfstatat(0xffffff9c) = 0
openat(0xffffff9c)BLOCKED = -38
newfstatat(0xffffff9c) = -2
openat(0xffffff9c)BLOCKED = -38
newfstatat(0xffffff9c) = -2
openat(0xffffff9c)BLOCKED = -38
newfstatat(0xffffff9c) = 0
openat(0xffffff9c)BLOCKED = -38
newfstatat(0xffffff9c) = -2
openat(0xffffff9c)BLOCKED = -38
newfstatat(0xffffff9c) = -2
openat(0xffffff9c)BLOCKED = -38
newfstatat(0xffffff9c) = 0
openat(0xffffff9c)BLOCKED = -38
newfstatat(0xffffff9c) = -2
openat(0xffffff9c)BLOCKED = -38
newfstatat(0xffffff9c) = -2
openat(0xffffff9c)BLOCKED = -38
newfstatat(0xffffff9c) = 0
access(0x2)cat: error while loading shared libraries: libc.so.6: cannot open shared object file: Permission denied
 = 104
exit_group(0x7f)
```


**Output con -EPERM**

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace cat /etc/passwd
brk(0x0) = 99751971921920
mmap(0x0) = 133053564239872
writev(0x7902f27255b0) = -2
openat(0xffffff9c)BLOCKED = -38
openat(0xffffff9c)BLOCKED = -38
newfstatat(0xffffff9c) = -2
openat(0xffffff9c)BLOCKED = -38
newfstatat(0xffffff9c) = -2
openat(0xffffff9c)BLOCKED = -38
newfstatat(0xffffff9c) = 0
access(0x2)cat: error while loading shared libraries: libc.so.6: cannot open shared object file: Operation not permitted
 = 110
exit_group(0x7f)
```

Se puede ver como en uno nos dice `Permission denied` mientras que en otro dice `Operation nor permitted` Pero además una de las cosas más llamativas es que las llamadas son más o menos dependiendo del error. 

Piensa que desde el proceso padre hemos engañado al proceso hijo devolviendo un código de error inventado, que el hijo se ha tragado como si fuese un veredicto del kernel mintiendo en este caso al pobre cat por el canal que más confía, esto ha provocado que haga una acción u otra dependiendo del error que recibe, debido al manejo de errores en su código.

Hagamos una cosa, vamos a realizar una ligera modificación manual previa a nuestro if en la entrada de la syscall:

```c
read_child_string(child, regs.rsi, str, sizeof(str));
fprintf(stderr, "%s(%d, \"%s\")", entry->name, (int)regs.rdi, str);
```

Y lo lanzamos de nuevo:

**Output con -EACCES**

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace cat /etc/passwd
brk(0x0) = 106029665501184
mmap(0x0) = 133949423562752
writev(0x79d387d2c5b0) = -2
openat(-100, "/etc/ld.so.cache")BLOCKED = -38
openat(-100, "/lib/x86_64-linux-gnu/glibc-hwcaps/x86-64-v3/libc.so.6")BLOCKED = -38
newfstatat(0xffffff9c) = -2
openat(-100, "/lib/x86_64-linux-gnu/glibc-hwcaps/x86-64-v2/libc.so.6")BLOCKED = -38
newfstatat(0xffffff9c) = -2
openat(-100, "/lib/x86_64-linux-gnu/libc.so.6")BLOCKED = -38
newfstatat(0xffffff9c) = 0
openat(-100, "/usr/lib/x86_64-linux-gnu/glibc-hwcaps/x86-64-v3/libc.so.6")BLOCKED = -38
newfstatat(0xffffff9c) = -2
openat(-100, "/usr/lib/x86_64-linux-gnu/glibc-hwcaps/x86-64-v2/libc.so.6")BLOCKED = -38
newfstatat(0xffffff9c) = -2
openat(-100, "/usr/lib/x86_64-linux-gnu/libc.so.6")BLOCKED = -38
newfstatat(0xffffff9c) = 0
openat(-100, "/lib/glibc-hwcaps/x86-64-v3/libc.so.6")BLOCKED = -38
newfstatat(0xffffff9c) = -2
openat(-100, "/lib/glibc-hwcaps/x86-64-v2/libc.so.6")BLOCKED = -38
newfstatat(0xffffff9c) = -2
openat(-100, "/lib/libc.so.6")BLOCKED = -38
newfstatat(0xffffff9c) = 0
openat(-100, "/usr/lib/glibc-hwcaps/x86-64-v3/libc.so.6")BLOCKED = -38
newfstatat(0xffffff9c) = -2
openat(-100, "/usr/lib/glibc-hwcaps/x86-64-v2/libc.so.6")BLOCKED = -38
newfstatat(0xffffff9c) = -2
openat(-100, "/usr/lib/libc.so.6")BLOCKED = -38
newfstatat(0xffffff9c) = 0
access(0x2)cat: error while loading shared libraries: libc.so.6: cannot open shared object file: Permission denied
 = 104
exit_group(0x7f)
```

**Output con -EPERM**

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace cat /etc/passwd
brk(0x0) = 95755764318208
mmap(0x0) = 132824277864448
writev(0x78cd8fe9e5b0) = -2
openat(-100, "/etc/ld.so.cache")BLOCKED = -38
openat(-100, "/lib/x86_64-linux-gnu/glibc-hwcaps/x86-64-v3/libc.so.6")BLOCKED = -38
newfstatat(0xffffff9c) = -2
openat(-100, "/lib/x86_64-linux-gnu/glibc-hwcaps/x86-64-v2/libc.so.6")BLOCKED = -38
newfstatat(0xffffff9c) = -2
openat(-100, "/lib/x86_64-linux-gnu/libc.so.6")BLOCKED = -38
newfstatat(0xffffff9c) = 0
access(0x2)cat: error while loading shared libraries: libc.so.6: cannot open shared object file: Operation not permitted
 = 110
exit_group(0x7f)
```

Cuando se usa `-EPERM` está diciendo, si no me permites esta operación no gasto más tiempo, mientras que con -EACCESS dice, no hay problema si no puedo leer /lib/x86_64-linux-gnu/libc.so.6 lo sigo buscando en otros directorios a ver si doy con uno que si sepa leer. Esto puede ser muy interesante para entender el manejo de errores de binarios e incluso aprovecharte de ellos, ¿Qué pasa entonces si veo que tengo permisos de escritura sobre alguno de estos directorios en los que busca y ese binario se lanza con SUID o una tarea cron con otro usuario? 😈

> [!WARNING]
> ¿Te has dado cuenta que estamos imprimiendo -38 como error?

Realmente -38 es `-ENOSYS` que quiere decir Function not implemented. Esto es porque estamos capturando el valor de retorno antes de sobreescribirlo:

```c
else {
    /* salida: rax tiene el valor de retorno */
    long ret = (long)regs.rax;
    if (sandbox_blocked) {
        regs.rax = -EPERM;                
        ptrace(PTRACE_SETREGS, child, 0, &regs);
        sandbox_blocked = 0;
    }
    if (ret < 0)
        fprintf(stderr, " = %ld\n", ret);
    else
        fprintf(stderr, " = %ld\n", ret);
}
```

Cuando imprimimos al final es el error real que nos devolvió el kernel con nuestra modificación en la entrada, la solución es fácil, pero era interesante para ver que realmente el kernel nos devolvió un retorno real.

De todas formas ya tenemos una primera moraleja con esta fase, y es que establecer una política de bloqueo de esta forma es muy sensible, no hemos permitido siquiera que llegase a lo que nos interesaba de este cat, porque de forma previa al `openat` que se podía esperar al leer /etc/passwd ha tenido que leer una librería que tampoco hemos permitido, y por esto la protección de un sistema a estos niveles es tan sensible, y un error puede provocar una caída completa de un sistema.


## Fase 2, ¿Y si queremos políticas más concretas?

Ahora mismo lo que hemos hecho es, te dejamos pasar o no, pero ¿Y si quiero dejar pasar para ciertas cosas? Ya hemos visto que nos cargamos toda la lógica del binario de la otra forma. Así que vamos a intentar bloquear ciertos paths, por ejemplo permitir `openat` sólo en ciertas situaciones.

> [!WARNING]
> Antes de que me regañes se que no estoy haciendo el código más limpio del mundo, pero tampoco es el objetivo, vamos a quedarnos con el aprendizaje detrás del código usado como herramienta.

```c
#include <string.h>
```

Lo vamos a necesitar para el strcmp que nos permite comparar strings.

```c
struct blacklist_path {
    const char *name;
};

static const struct blacklist_path blacklist_path_table[] = {
    { "/etc/passwd" },
    { NULL }
};
```

He dejado una estructura creada únicamente por si luego queremos añadir algo más, es un por si acaso. Y si, lo sé, soy vago, he metido un NULL cómo último valor del array a modo centinela, así no me preocupo de comprobar cuando acaba.

```c
bool is_blacklisted(const char *path) {
    for (const struct blacklist_path *p = blacklist_path_table; p->name != NULL; p++) {
        if (strcmp(path, p->name) == 0){
            return 1;
            }
    }
    return 0;
}
```

Esta ya si es la función que va a comparar el string que introducimos con el array anterior. No hay mucho que decir, en cuanto vea NULL deja de iterar, mientras tanto compara uno a uno con los paths que hayamos metido en nuestra blacklist y devuelve un booleano, está o no está.

```c
if (!entry) {
    fprintf(stderr, "syscall_%ld(0x%llx)", nr, regs.rdi);
} else if (entry->blacklist == 1) {
    read_child_string(child, regs.rsi, str, sizeof(str));
    if (is_blacklisted(str)) {
        fprintf(stderr, "%s(%d, \"%s\")", entry->name, (int)regs.rdi, str);
        fprintf(stderr, "BLOCKED");
        regs.rax = -1; 
        ptrace(PTRACE_SETREGS, child, 0, &regs);
        sandbox_blocked = 1;                  // flag para la salida
    }
    else {
        read_child_string(child, regs.rsi, str, sizeof(str));
        fprintf(stderr, "%s(%d, \"%s\")", entry->name, (int)regs.rdi, str);
    }
```

Y, por supuesto, esto lo hemos introducido en nuestro condicional, cómo digo el código está bastante sucio, porque lo estamos centrando únicamente en la prueba del path. Por esa razón, estamos leyendo el registro rsi directamente sin hacer más comprobaciones, ya que cómo sabíamos previamente, es donde vamos a ver el argumento del openat que se va a realizar. Envíamos esta información a nuestra función y ahora sólo bloqueamos en caso de que el path concreto esté en blacklist. Vamos a ponerlo a prueba:

```bash
miguel@demo-ubuntu:~/minitrace$  ./minitrace cat /etc/passwd
brk(0x0) = 107928751235072
...
close(0x3) = 0
openat(-100, "/lib/x86_64-linux-gnu/libc.so.6") = 3
read(0x3) = 832
...
...
openat(-100, "/etc/passwd")BLOCKED = 3
write(2, "cat: ")cat:  = 5
write(2, "/etc/passwd�V{")/etc/passwd = 11
openat(-100, "/usr/share/locale/locale.alias") = 4
fstat(0x4) = 0
read(0x4) = 2996
read(0x4) = 0
close(0x4) = 0
openat(-100, "/usr/share/locale/es_ES.UTF-8/LC_MESSAGES/libc.mo") = -2
openat(-100, "/usr/share/locale/es_ES.utf8/LC_MESSAGES/libc.mo") = -2
openat(-100, "/usr/share/locale/es_ES/LC_MESSAGES/libc.mo") = -2
openat(-100, "/usr/share/locale/es.UTF-8/LC_MESSAGES/libc.mo") = -2
openat(-100, "/usr/share/locale/es.utf8/LC_MESSAGES/libc.mo") = -2
openat(-100, "/usr/share/locale/es/LC_MESSAGES/libc.mo") = -2
openat(-100, "/usr/share/locale-langpack/es_ES.UTF-8/LC_MESSAGES/libc.mo") = -2
openat(-100, "/usr/share/locale-langpack/es_ES.utf8/LC_MESSAGES/libc.mo") = -2
openat(-100, "/usr/share/locale-langpack/es_ES/LC_MESSAGES/libc.mo") = -2
openat(-100, "/usr/share/locale-langpack/es.UTF-8/LC_MESSAGES/libc.mo") = -2
openat(-100, "/usr/share/locale-langpack/es.utf8/LC_MESSAGES/libc.mo") = -2
openat(-100, "/usr/share/locale-langpack/es/LC_MESSAGES/libc.mo") = 4
...
write(2, ": Permiso denegado"): Permiso denegado = 18
write(2, "
TF-8")
 = 1
close(0x1) = 0
close(0x2) = 0
exit_group(0x1)
```

En esta ocasión si que estamos viendo muchos openat que no están siendo bloqueados, únicamente el que intenta abrir /etc/passwd devolviendo de output `Permiso denegado`. Probemos con otro archivo.

```bash
miguel@demo-ubuntu:~/minitrace$  ./minitrace cat /tmp/test.txt
brk(0x0) = 100429855199232
...
openat(-100, "/tmp/test.txt") = 3
fstat(0x3) = 0
syscall_221(0x3) = 0
mmap(0x0) = 126859302477824
read(0x3) = 21
write(1, "Hello!!!!!
Hola!!!!!
")Hello!!!!!
Hola!!!!!
 = 21
...
exit_group(0x0)
```

Sin embargo, si intentamos leer otro archivo funciona perfectamente.

> [!DANGER]
> Pero... En todo esto tenemos un problemón enorme... ¿Te suena el TOCTOU?

## TOCTOU

Y ahora viene el gran problema, ¿De verdad hemos cortado la comunicación gracias a nuestro pequeño binario? Paramos la llamada al kernel sin problema únicamente cuando está en paths prohibidos, pero aquí es donde la condición de carrera o TOCTOU nos mata...

Te voy a enseñar un output:

```bash
miguel@demo-ubuntu:~/minitrace$  ./minitrace ../exploit 2>/dev/null
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sroot:x:0:0:root:/root:/bin/bash
```

He mandado el descriptor 2 al vacío existencial para limpiar el output y que se pueda observar un gran problema... Si tenemos cortado el acceso a /etc/passwd a través de nuestro binario ¿Cómo hemos conseguido bypassearlo?

```c
#include <fcntl.h>
#include <pthread.h>
#include <string.h>
#include <unistd.h>

static char path[256] = "/tmp/loquesea.txt";

void *flipper(void *arg) {
    for (;;) {
        memcpy(path, "/tmp/innocent.txt", 18);
        usleep(1);
        memcpy(path, "/etc/passwd", 12);
        usleep(1);
    }
    return NULL;
}

int main() {
    pthread_t t;
    pthread_create(&t, NULL, flipper, NULL);
    for (int i = 0; i < 10000; i++) {
        int fd = openat(AT_FDCWD, path, O_RDONLY);
        if (fd >= 0) {
            char buf[128];
            ssize_t n = read(fd, buf, sizeof(buf));
            if (n > 0) write(1, buf, n);
            close(fd);
        }
    }
    return 0;
}
```

Voy a explicarlo por partes:

```c
static char path[256] = "/tmp/loquesea.txt";
```

Es un buffer que cualquier hilo va a poder leer y modificar sin problema.

```c
void *flipper(void *arg) {
    for (;;) {
        memcpy(path, "/tmp/loquesea.txt", 18);
        usleep(1);
        memcpy(path, "/etc/passwd", 12);
        usleep(1);
    }
    return NULL;
}

int main() {
    pthread_t t;
    pthread_create(&t, NULL, flipper, NULL);
```
Esto es un bucle infinito, lo que va a hacer es copiar /tmp/loquesea.txt al buffer path, suspender durante 1 microsegundo y a continuación copiar /etc/passwd al buffer path.

Y luego una vez inicia el programa en main creamos un hilo que va a ejecutar el flipper de forma continua hasta el fin del programa.

```c
 for (int i = 0; i < 10000; i++) {
        int fd = openat(AT_FDCWD, path, O_RDONLY);
        if (fd >= 0) {
            char buf[128];
            ssize_t n = read(fd, buf, sizeof(buf));
            if (n > 0) write(1, buf, n);
            close(fd);
        }
    }
    return 0;
}
```

Y aquí se realizan 10000 iteraciones en las que se va a intentar hacer un openat directo al buffer path, que va cambiando entre uno legítimo al que se tiene acceso y uno que no se tiene acceso /etc/passwd.

A continuación, simplemente, si tiene éxito pues declara un buffer local `char buf[128];`, lee el archivo `ssize_t n = read(fd, buf, sizeof(buf));` y lo escribe en la salida estándar `if (n > 0) write(1, buf, n);`.

En resumen, lo que está haciendo es intentar leer continuamente un path al que no tiene acceso (si es ejecutado con nuestro binario que podríamos haber integrado en la shell del sistema). Lo que realmente pasa aquí es que aprovecha un pequeño momento en nuestro código entre que se hace el check de la string del path y el momento en que el kernel realmente lo lee. Y aquí está el enorme problema de intentar hacer estas cosas con ptrace, la ventana que existe entre que obtenemos los registros, leemos la memoria del hijo, tomamos una decisión y hacemos la syscall es de microsegundos en realidad, pero es más que suficiente para encontrar el hueco donde entrar.

Y sinceramente, tampoco hace falta complicarse tanto con código, aunque es interesante para seguir aprendiendo, desde una terminal se puede conseguir el mismo resultado. Por un lado tendríamos una terminal tal que así:

```lang:filename.ext
while true; do 
ln -sf /tmp/innocent.txt /tmp/toctou 
ln -sf /etc/passwd /tmp/toctou 
done
```

Hacemos un bucle infinito que va a modificar un enlace simbólico hacia un archivo que podemos leer y otro que no podemos leer.


```bash
while true; do 
./minitrace cat /tmp/toctou 2>/dev/null
done
```

Después lanzaríamos otro bucle que continuamente intente hacer el cat y tendríamos el resultado.

> [!DANGER]
> Pero la cosa va a peor...

Hay una forma aún más simple de hacerlo, y es simplemente...

```lang:filename.ext
ln -sf /etc/passwd /tmp/toctou
./minitrace cat /tmp/toctou
```

Pues vaya... tanto esfuerzo y lo estamos rompiendo nada más que con mirarlo un poco fuerte... Imagina esto, con mucho más código, en formas mucho más complejas, a niveles mucho más profundos, y entenderás porque aparecen continuamente CVEs nuevos en códigos muy antiguos.

Hay que pensar que estamos operando sobre user space, específicamente sobre lo que el proceso le dice al kernel. Por debajo están ocurriendo un montón de cosas, normalizar, seguir symlinks, contexto, namespaces, descriptores... De hecho hay muchas más formas de saltarse esta protección, todas basadas en el mismo principio. Y si, podríamos intentar hacer más comprobaciones en nuestro código, salvaguardas... Pero la razón por la que quería mostrar el TOCTOU es precisamente porque hagamos lo que hagamos siempre vamos a tener ese problema encima.

¿Cuál es el veredicto? Que todo mecanismo de seguridad que separe una decisión por un proceso ajeno va a tener este problema fundamental. Y esto se puede traducir a muchas cosas, validación por sidecar, comprobaciones en userspace y kernel, validaciones en cliente y servidor...

Por lo que sabemos que en este caso concreto necesitamos trabajar si o si sobre el kernel y utilizar otras opciones, pero ya mejor otro día que es tarde.





