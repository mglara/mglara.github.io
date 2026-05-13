# Trazando Syscalls

Simplemente me apetecía jugar un poco con las syscalls y aprovechar para explicar algo más a fondo como funcionan.

## Minitrace

¿Cuál es la idea conceptual que vamos a intentar? Se hace un fork, el hijo se marca a si mismo como trazable con ptrace y luego ejecuta el comando. Con esto hecho el padre entra en un bucle para leer e imprimir las syscalls.

> [!INFO]
> Vamos a ir por partes y entendiendo que está pasando, lo veo la mejor opción.

### Imprimiendo syscall

Vamos a comenzar simplemente imprimiendo syscall cada vez que se haga una llamada al sistema, pero no nos preocupamos de capturar qué llamada es.

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    if (argc < 2) {
        fprintf(stderr, "usage: %s <command> [args...]\n", argv[0]);
        return 1;
    }

    pid_t child = fork();

    if (child == 0) {
        ptrace(PTRACE_TRACEME, 0, NULL, NULL);
        execvp(argv[1], &argv[1]);
        perror("execvp");
        exit(1);
    }

    int status;
    waitpid(child, &status, 0);
    ptrace(PTRACE_SETOPTIONS, child, 0, PTRACE_O_TRACESYSGOOD);

    while (1) {
        ptrace(PTRACE_SYSCALL, child, NULL, NULL);
        waitpid(child, &status, 0);

        if (WIFEXITED(status))
            break;

        if (WIFSTOPPED(status) && WSTOPSIG(status) == (SIGTRAP | 0x80))
            printf("syscall!\n");
    }

    return 0;
}

```

Nos paramos un poco a ver qué estamos haciendo para entenderlo, algunos detalles del código los omito ya que son por hacer una herramienta bonita más que utilidad real para lo que estamos haciendo.

```c
    pid_t child = fork();

    if (child == 0) {
        ptrace(PTRACE_TRACEME, 0, NULL, NULL);
        execvp(argv[1], &argv[1]);
        perror("execvp");
        exit(1);
    }
```

`child` en el padre contendrá el PID del hijo, mientras que en el hijo child vale 0. Entendiendo esto lo que hemos hecho es que si child tiene el valor de 0, es decir, estamos en el proceso hijo, va a llamar a una función llamada ptrace indicando al kernel que está listo para ser depurado.

A partir de este momento el hijo es el `tracee` o proceso traceado y el padre será el `tracer` [manual de ptrace](https://man7.org/linux/man-pages/man2/ptrace.2.html).

Perfecto, pero ¿Y lo demás? Bueno pues tenemos la llamada a execvp obteniendo el argumento y puntero del argumento con el que se inicia nuestro binario `./minitrace cat /etc/hostname` En este caso concreto estaríamos obteniendo `cat` básicamente lo que hacemos es que el proceso hijo que hemos creado lo sustituimos por el argumento, en este caso el binario cat.

Lo demás es simplemente es para capturar el error y salir del programa.

> [!INFO]
> ¡IMPORTANTE! Cuando se usa PTRACE_TRACEME el hijo va a enviar una señal SIGTRAP que el padre puede utilizar para comenzar su función.

¿Por qué utilizar execvp y no execve como mencionan en el manual? Es simple comodidad, de esta forma buscamos en el PATH del sistema sin tener que ejecutar la ruta absoluta.

Continuamos

```c
    int status;
    waitpid(child, &status, 0);
    ptrace(PTRACE_SETOPTIONS, child, 0, PTRACE_O_TRACESYSGOOD);
```

Aquí utilizamos [waitpid](https://man7.org/linux/man-pages/man3/waitpid.3p.html) que está esperando el PID del proceso, el status y las opciones. Básicamente lo que hacemos es capturar ese primer SIGTRAP del kernel para controlar al `tracee`

Y a continuación, volvemos a usar la función de ptrace, pero en este caso para llamar a SETOPTIONS y lo que establecemos es la opción `PTRACE_O_TRACESYSGOOD`. El problema que tenemos ahora mismo es que instrucciones normales y llamadas al sistema ahora mismo van a provocar un SIGTRAP y no vamos a poder identificar qué es una `syscall` y que es otra cosa, por eso marcamos esta opción, a partir de ahora la señal tanto de entrada como de salida de una syscall se marca con 0x80, luego vamos a entender mejor esto.

```c
    while (1) {
        ptrace(PTRACE_SYSCALL, child, NULL, NULL);
        waitpid(child, &status, 0);

        if (WIFEXITED(status))
            break;

        if (WIFSTOPPED(status) && WSTOPSIG(status) == (SIGTRAP | 0x80))
            printf("syscall!\n");
    }
```

Aquí tenemos la joya de la corona que hemos trabajado, nos metemos en un bucle "infinito" y lo primero que tenemos que hacer es reanudar la ejecución del hijo donde le indicamos que se detenga en la proxima entrada o salida de una llamada al sistema. Además, con waitpid nos volvemos a poner a la espera bloqueando al padre hasta que el hijo se detenga.

Aquí ya vienen los dos condicionales, el primero simplemente es para salir del bucle cuando el hijo a terminado. El segundo condicional es el importante, y la razón por la que pusimos la opción `PTRACE_O_TRACESYSGOOD` vamos a desgranarlo.

- WIFSTOPPED que lo sacamos de la librería de sys/wait.h, su status será verdadero si el hijo se ha detenido por una SIGTRAP.
- WSTOPSIG nos ayuda a extraer el número de la señal que causó la parada y aquí es donde lo comparamos...
- SIGTRAP | 0x80 -> Es decir el bit 0x80 y operación OR con la SIGTRAP (0x5) que hacen 0x85(133) y será lo que nos devuelva. Aquí es importante entender que el pipe `|` en `c` es el operador OR bit a bit. Ahora lo vemos visualmente pero en este caso concreto es una suma, pero porque da la casualidad de que no hay bits solapados.

```binary
  5:   00000000 00000000 00000000 00000101
128:   00000000 00000000 00000000 10000000
OR:    00000000 00000000 00000000 10000101  = 133
```

Es decir, el status de WSTOPSIG será igual a 133 cuando se trate de una syscall, porque estamos añadiendo con la opción previa ese bit 0x80.

He añadido de forma temporal un par de variables para enseñarlo.

![Variables sigtrap](/trazando-syscalls/sigtrap_variables.png)

Si entramos en el condicional por ahora simplemente imprimimos syscall por pantalla.

> [!WARNING]
> Recordad que vamos a recibir tanto la syscall de entrada como la de salida

Vamos a ponerlo a prueba, compilamos y a ver qué ocurre...

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace cat /etc/hostname
syscall!
syscall!
syscall!
...
syscall!
demo-ubuntu
syscall!
syscall!
...
syscall!
```

Unas cuantas syscalls después y el hostname voilà tenemos ya la primera parte acabada.

---

### Imprimiendo y mapeando el número de las syscall

Ahora nos interesa saber qué syscalls se están llamando, así que vamos a trabajar sobre ello.

```c
struct syscall_entry {
    long number;
    const char *name;
};

static const struct syscall_entry syscall_table[] = {
    {   0, "read"    },
    {   1, "write"   },
    {   3, "close"   },
    {   5, "fstat"   },
    {   9, "mmap"    },
    {  12, "brk"     },
    {  59, "execve"  },
    { 257, "openat"  },
};

static const char *syscall_name(long nr) {
    for (size_t i = 0; i < sizeof(syscall_table) / sizeof(syscall_table[0]); i++) {
        if (syscall_table[i].number == nr)
            return syscall_table[i].name;
    }
    return NULL;
}
```

Por un lado creamos un `struct` en el que indicamos los tipos para a continuación hacer un array llamado syscall_table a la que consultaremos con cada número de cada syscall.

> [!WARNING]
> Estos números pertenecen a arquitectura x86_64, tenemos un listado en filippo.io/linux-syscall-table/

A continuación tenemos una función simplemente para buscar, le pasamos el número de syscall, recorre la tabla y devuelve el nombre.

```c
if (WIFSTOPPED(status) && WSTOPSIG(status) == (SIGTRAP | 0x80)) {
            struct user_regs_struct regs;
            ptrace(PTRACE_GETREGS, child, NULL, &regs);
            long nr = (long)regs.orig_rax;
            const char *name = syscall_name(nr);
            if (name)
                printf("%s\n", name);
            else
                printf("syscall_%ld\n", nr);
        }
```

Y hay que modificar el segundo condicional que teníamos, ahora en vez de syscall si tenemos el nombre mapeado que nos aparezca.

`user_regs_struct` es una estructura de sys/user.h con el que podemos obtener todos los registros (rax, rbx, rcx, rdx, rsp, rip, etc...) y utilizamos nuevamente ptrace pero ahora para obtener los registros.

Aquí importante en arquitectura x86/64 el número de la syscall se coloca en el registro `orig_rax`.

Vamos a verlo con radare:

```bash
miguel@demo-ubuntu:~/low-level/minitrace$ r2 -d /bin/cat 
[0x7ed619ee6540]> ood /etc/hostname
child received signal 9
File dbg:///usr/bin/cat /etc/hostname reopened in read-write mode
[0x7dec35315540]> dcs
Running child until next syscall
--> SN 0x7dec3531a9cb syscall 12 unknown (0x0 0x7dec3532ca18 0x0)
[0x7dec3531a9cb]> dr
rax = 0xffffffffffffffda
rbx = 0x7dec35316af0
rcx = 0x7dec3531a9cb
rdx = 0x00000000
r8 = 0x00000000
r9 = 0x7dec3532c440
r10 = 0x0000037f
r11 = 0x00000246
r12 = 0x7ffe3494cd50
r13 = 0x00000000
r14 = 0x7dec352f6000
r15 = 0x7dec352f6590
rsi = 0x7dec3532ca18
rdi = 0x00000000
rsp = 0x7ffe3494cc88
rbp = 0x7ffe3494ccc0
rip = 0x7dec3531a9cb
rflags = 0x00000246
orax = 0x0000000c
[0x7dec3531a9cb]> ?vi `dr orax`
12
```

Básicamente lo que estamos haciendo es ejecutar el mismo binario y decirle que nos pare antes de una syscall, aunque ya nos avisa que es el número 12, cómo no nos fiamos lo que hacemos es ver todos los registros y vemos orig_rax `orax = 0x0000000c` que si lo traducimos justo debajo es el mismo número 12.

Y además ese número 12 es la primera syscall, que si vamos a nuestro código se mapea como brk, así que ahora cuando lo ejecutemos fijaos en la primera syscall.

Os lo pongo también de forma visual con el panel de radare, aunque ya os confirmo que no es agradable.

![radare orax](/trazando-syscalls/radare_orax.png)

> [!INFO]
> Como detalle si os fijáis ahora al imprimir una syscall no mapeada le concatenamos el número por si queremos mapearla o buscarla a mano.


Vamos a ejecutarlo a ver qué ocurre

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace cat /etc/hostname
brk
brk
mmap
mmap
syscall_21
syscall_21
openat
openat
...
read
read
write
demo-ubuntu
write
read
read
...
close
close
close
close
syscall_231
```

Vemos cosas interesantes por aquí, para empezar, todas las syscalls están repetidas, esto es debido a que al usar `PTRACE_SYSCALL` el proceso se detiene dos veces por syscall, al entrar y al salir, imprimiendo el mismo nombre en ambas ocasiones, por ahora vamos a dejarlo asi.


### ¿Imprimimos el primer argumento?

Vamos a seguir avanzando un poco, para empezar imprimiendo el primer argumento

```c
if (WIFSTOPPED(status) && WSTOPSIG(status) == (SIGTRAP | 0x80)) {
            struct user_regs_struct regs;
            ptrace(PTRACE_GETREGS, child, NULL, &regs);
            long nr = (long)regs.orig_rax;
            const char *name = syscall_name(nr);
            if (name)
                printf("%s(%llu)\n", name, regs.rdi);
            else
                printf("syscall_%ld(%llu)\n", nr, regs.rdi);
        }
    }
```

Los cambios están en que hemos añadido un unsigned long long `llu` que imprime como número entero el registro `rdi`. ¿Y qué es esto?
Normalmente en las arquitecturas x86/64, los argumentos se colocan en orden en los registros siguiendo el siguiente patrón.

rdi,rsi,rdx,r10,r8,r9.

¿Entonces qué es rdi? El primer argumento que se le pasa a la syscall, ejecutemos a ver qué pasa.

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace cat /etc/hostname
brk(0)
brk(0)
mmap(0)
mmap(0)
syscall_21(131031247156656)
syscall_21(131031247156656)
openat(4294967196)
openat(4294967196)
fstat(3)
fstat(3)
...
read(3)
read(3)
write(1)
demo-ubuntu
write(1)
read(3)
read(3)
...
close(3)
close(3)
close(1)
close(1)
close(2)
close(2)
syscall_231(0)
```

Genial, esto va tomando forma, una de las primeras cosas que observamos es que el primer argumento es el mismo en la entrada y en la salida de la syscall, esto es porque el kernel antes de salir lo restaura al mismo registro.

Si por ejemplo vemos la `syscall_21` tiene un número extremadamente grande vamos a buscar qué es en [la tabla](https://filippo.io/linux-syscall-table/) parece que es access, y el primer argumento es un número larguísimo, probablemente haga referencia a una dirección de memoria. Esto nos hace pensar que a futuro para poder hacer comprobaciones más fidedignas necesitaríamos mejor el hexadecimal, más sencillo y visual.

Vamos a ver esto más de cerca

![radare rdi](/trazando-syscalls/radare_rdi_memory.png)

El primer registro, es decir, el rdi, tiene un hexadecimal concreto, cómo hemos supuesto antes puede ser una dirección de memoria, vamos a seguirla.

![radare rdi](/trazando-syscalls/radare_rdi_memory_2.png)

Parece que tiene todo el sentido del mundo ¿no? está intentando acceder a la librería `ld.so.preload` por mi parte por ahora hemos salido de dudas.

Y obviando parte de los resultados me interesa centrarme en los argumentos con números manejables, tanto el 1 como el 3. Por ejemplo tenemos un read(3), lo que está haciendo es leer el descriptor 3, en este caso el `/etc/hostname`.

Y por otro lado tenemos write(1), MARAVILLOSO, porque enlazamos con este artículo [¿Qué son STDIN, STDOUT, STDERR?](/blog/stdin-stdout-stderr) donde ya vimos esto:

| Columna 1 | Columna 2 | 
|-----------|-----------|
| 0         | STDIN     |
| 1         | STDOUT     |
| 2         | STDERR     |

Es decir, estamos escribiendo en STDOUT, cómo estamos ejecutando el binario vemos la salida por terminal con el contenido de /etc/hostname.

Y luego vemos close(1,2 y 3) porque está cerrando todos los descriptores.


### ¿Y si toman una string? Pues la imprimimos también

Vamos a tomar más información de lo que está ocurriendo en cada syscall.

```c
struct syscall_entry {
    long        number;
    const char *name;
    int         str_arg; /* qué argumento posicional es un string: 0=ninguno, 1=rdi, 2=rsi */
};

static const struct syscall_entry syscall_table[] = {
    {   0, "read",   0 },
    {   1, "write",  2 },
    {   3, "close",  0 },
    {   5, "fstat",  0 },
    {   9, "mmap",   0 },
    {  12, "brk",    0 },
    {  59, "execve", 1 }, 
    { 257, "openat", 2 }, 
};

static void read_child_string(pid_t pid, unsigned long long addr, char *buf, size_t size) {
    struct iovec local  = { .iov_base = buf,          .iov_len = size - 1 };
    struct iovec remote = { .iov_base = (void *)addr, .iov_len = size - 1 };
    ssize_t n = process_vm_readv(pid, &local, 1, &remote, 1, 0);
    if (n < 0) {
        buf[0] = '?';
        buf[1] = '\0';
        return;
    }
    buf[n] = '\0';
}
```
¿Qué es esto? Una función auxiliar para poder leer un texto desde la memoria de un tracee sin necesidad de usar `PTRACE_PEEKDATA`
Para hacerlo necesita usar la llamada al sistema process_vm_readv, esto le permite transferir datos del proceso hijo al padre.

Para esto se usan dos vectores iovec, una estructura definida en `sys/uio.h` se le proporciona una dirección de inicio del buffer y el tamaño en bytes:
- local -> apunta al buffer donde guardaremos la información y con size - 1 se deja espacio para el terminador null.
- remote -> que apunta a donde está la cadena que queremos guardar.
 
> [!DANGER]
> Recordad esta línea `buf[n] = '\0'` ya que añade el null byte al final de los bytes leídos.

Además hemos cambiado la estructura de nuestro mapping para avisar que argumento es el que espera un string.

```c
if (WIFSTOPPED(status) && WSTOPSIG(status) == (SIGTRAP | 0x80)) {
      struct user_regs_struct regs;
      ptrace(PTRACE_GETREGS, child, NULL, &regs);
      long nr = (long)regs.orig_rax;
      const struct syscall_entry *entry = syscall_lookup(nr);

      char str[STR_MAX];
      const char *name = entry ? entry->name : NULL;

      if (!entry) {
          printf("syscall_%ld(%llu)\n", nr, regs.rdi);
      } else if (entry->str_arg == 1) {
          read_child_string(child, regs.rdi, str, sizeof(str));
          printf("%s(\"%s\")\n", name, str);
      } else if (entry->str_arg == 2) {
          read_child_string(child, regs.rsi, str, sizeof(str));
          printf("%s(%d, \"%s\")\n", name, (int)regs.rdi, str);
      } else {
          printf("%s(%llu)\n", name, regs.rdi);
      }
  }
```

Bueno, pues más cambios a nuestro condicional principal, antes estábamos llamando a `syscall_name` ahora necesitamos ir un paso más allá y vamos a llamar a `syscall_lookup` el cual nos devuelve un puntero a toda la entrada.

Nos toca manejar tres posibles casos:
- No conocemos la syscall `!entry`
- El primer argumento es una string y aquí aprovechamos nuestra función `read_child_string()`
- El segundo argumento es una cadena y el primero un entero también usamos `read_child_string()`
- Otros valores o todos enteros

Veamos las diferencias entre el segundo y tercer caso:

```c
#Segundo caso
else if (entry->str_arg == 1) {
  read_child_string(child, regs.rdi, str, sizeof(str));
#Tercer caso
else if (entry->str_arg == 2) {
  read_child_string(child, regs.rsi, str, sizeof(str));
```

Miramos manualmente, si el primer argumento o el segundo argumento es un string, e imprimimos el registro que corresponda, recordamos el orden que se ha mencionado antes, primero rdi, luego rsi.

Vamos a ejecutarlo, os muestro solo las partes interesantes:

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace cat /etc/hostname
brk(0)
brk(0)
...
openat(-100, "/etc/ld.so.cache")
openat(-100, "/etc/ld.so.cache")
...
openat(-100, "/lib/x86_64-linux-gnu/libc.so.6")
openat(-100, "/lib/x86_64-linux-gnu/libc.so.6")
...
openat(-100, "/usr/lib/locale/locale-archive")
openat(-100, "/usr/lib/locale/locale-archive")
...
openat(-100, "/etc/hostname")
openat(-100, "/etc/hostname")
...
syscall_231(0)
```

Vemos que hay varios openat antes siquiera de tocar /etc/hostname, primero consulta la caché de bibliotecas para resolver rutas `ld.so.cache` abre la biblioteca estándar libc `/lib/x86_64-linux-gnu/libc.so.6` y también necesita la de región `/usr/lib/locale/locale-archive`.

Además vemos que todas hacen uso del descriptor `-100` ojo que antes lo veíamos así:

```bash
openat(4294967196)
```

Esto era porque antes imprimiamos `regs.rdi` y ahora si os fijáis había un pequeño cambio e imprimos `(int)regs.rdi` y ese número 4294967196 es el equivalente sin signo a -100. ¿Y ese -100 qué es? Hace referencia a `AT_FDCWD` que básicamente es el directorio de trabajo actual.


### Vamos a distinguir entre entrada y salida

```c
int in_syscall = 0;

    while (1) {
        ptrace(PTRACE_SYSCALL, child, NULL, NULL);
        waitpid(child, &status, 0);

        if (WIFEXITED(status))
            break;

        if (WIFSTOPPED(status) && WSTOPSIG(status) == (SIGTRAP | 0x80)) {
            struct user_regs_struct regs;
            ptrace(PTRACE_GETREGS, child, NULL, &regs);

            if (!in_syscall) {
                long nr = (long)regs.orig_rax;
                const struct syscall_entry *entry = syscall_lookup(nr);
                char str[STR_MAX];

                if (!entry) {
                    printf("syscall_%ld(%llu)", nr, regs.rdi);
                } else if (entry->str_arg == 1) {
                    read_child_string(child, regs.rdi, str, sizeof(str));
                    printf("%s(\"%s\")", entry->name, str);
                } else if (entry->str_arg == 2) {
                    read_child_string(child, regs.rsi, str, sizeof(str));
                    printf("%s(%d, \"%s\")", entry->name, (int)regs.rdi, str);
                } else {
                    printf("%s(%llu)", entry->name, regs.rdi);
                }
            } else {
                long ret = (long)regs.rax;
                if (ret < 0)
                    printf(" = %ld\n", ret);
                else
                    printf(" = %ld\n", ret);
            }

            in_syscall = !in_syscall;
        }
    }
```
Vamos a quitar duplicados por entrada y salida, no es la forma más perfecta, pero nos vale, lo que hacemos es invertir una flag `in_syscall`.

Quitamos el salto de línea y eso nos permite poder añadir un else con más información:

```c
else {
                long ret = (long)regs.rax;
                if (ret < 0)
                    printf(" = %ld\n", ret);
                else
                    printf(" = %ld\n", ret);
            }
```

Básicamente leemos el valor de `rax` con el valor que nos devuelve la syscall, así que tenemos mucha información, descriptores, strings, valor devuelto y sin repeticiones por entrada y salida.

Vamos a lanzarlo:

```bash
miguel@demo-ubuntu:~/minitrace$ ./minitrace cat /etc/hostname
brk(0) = 107374942257152
mmap(0) = 130344951947264
syscall_21(130344952182192) = -2
openat(-100, "/etc/ld.so.cache") = 3
fstat(3) = 0
mmap(0) = 130344951877632
close(3) = 0
openat(-100, "/lib/x86_64-linux-gnu/libc.so.6") = 3
read(3) = 832
syscall_17(3) = 784
fstat(3) = 0
syscall_17(3) = 784
mmap(0) = 130344948858880
mmap(130344949022720) = 130344949022720
mmap(130344950628352) = 130344950628352
mmap(130344950951936) = 130344950951936
mmap(130344950976512) = 130344950976512
close(3) = 0
mmap(0) = 130344951865344
syscall_158(4098) = 0
syscall_218(130344951867920) = 102625
syscall_273(130344951867936) = 0
syscall_334(130344951869536) = 0
syscall_10(130344950951936) = 0
syscall_10(107374131417088) = 0
syscall_10(130344952209408) = 0
syscall_302(0) = 0
syscall_11(130344951877632) = 0
syscall_318(130344950997368) = 8
brk(0) = 107374942257152
brk(107374942392320) = 107374942392320
openat(-100, "/usr/lib/locale/locale-archive") = 3
fstat(3) = 0
mmap(0) = 130344934178816
close(3) = 0
fstat(1) = 0
openat(-100, "/etc/hostname") = 3
fstat(3) = 0
syscall_221(3) = 0
mmap(0) = 130344951726080
read(3) = 31
demo-ubuntu
write(1) = 31
read(3) = 0
syscall_11(130344951726080) = 0
close(3) = 0
close(1) = 0
close(2) = 0
syscall_231(0)
```

¿Todo esto os empieza a sonar a algo? Vamos a probar con un comando más conocido `strace`

```bash
miguel@demo-ubuntu:~/minitrace$  strace cat /etc/hostname
execve("/usr/bin/cat", ["cat", "/etc/hostname"], 0x7ffdc2d6b3e8 /* 52 vars */) = 0
brk(NULL)                               = 0x59f6f8137000
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x79f191157000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No existe el archivo o el directorio)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=67127, ...}) = 0
mmap(NULL, 67127, PROT_READ, MAP_PRIVATE, 3, 0) = 0x79f191146000
close(3)                                = 0
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\220\243\2\0\0\0\0\0"..., 832) = 832
pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 784, 64) = 784
fstat(3, {st_mode=S_IFREG|0755, st_size=2125328, ...}) = 0
pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 784, 64) = 784
mmap(NULL, 2170256, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x79f190e00000
mmap(0x79f190e28000, 1605632, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x28000) = 0x79f190e28000
mmap(0x79f190fb0000, 323584, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1b0000) = 0x79f190fb0000
mmap(0x79f190fff000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1fe000) = 0x79f190fff000
mmap(0x79f191005000, 52624, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x79f191005000
close(3)                                = 0
mmap(NULL, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x79f191143000
arch_prctl(ARCH_SET_FS, 0x79f191143740) = 0
set_tid_address(0x79f191143a10)         = 134637
set_robust_list(0x79f191143a20, 24)     = 0
rseq(0x79f191144060, 0x20, 0, 0x53053053) = 0
mprotect(0x79f190fff000, 16384, PROT_READ) = 0
mprotect(0x59f6cb77a000, 4096, PROT_READ) = 0
mprotect(0x79f191197000, 8192, PROT_READ) = 0
prlimit64(0, RLIMIT_STACK, NULL, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
munmap(0x79f191146000, 67127)           = 0
getrandom("\x68\x0a\xab\x64\xb9\x41\x06\xa0", 8, GRND_NONBLOCK) = 8
brk(NULL)                               = 0x59f6f8137000
brk(0x59f6f8158000)                     = 0x59f6f8158000
openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=14596880, ...}) = 0
mmap(NULL, 14596880, PROT_READ, MAP_PRIVATE, 3, 0) = 0x79f190000000
close(3)                                = 0
fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0x3), ...}) = 0
openat(AT_FDCWD, "/etc/hostname", O_RDONLY) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=31, ...}) = 0
fadvise64(3, 0, 0, POSIX_FADV_SEQUENTIAL) = 0
mmap(NULL, 139264, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x79f191121000
read(3, "demo-ubuntu", 131072) = 31
write(1, "demo-ubuntu", 31demo-ubuntu
) = 31
read(3, "", 131072)                     = 0
munmap(0x79f191121000, 139264)          = 0
close(3)                                = 0
close(1)                                = 0
close(2)                                = 0
exit_group(0)                           = ?
+++ exited with 0 +++
```

Tienen cierto parecido ¿no?, salvando las diferencias, lo nuestro hemos tardado un rato, y strace es una herramienta consolidada.

## Algunos detalles extra

¿No os habéis preguntado porque no he sacado el string del argumento de write? Vamos a verlo:

```c
static const struct syscall_entry syscall_table[] = {
   ...
    {   1, "write",  2 },
  ...
};
```
El string está en el segundo argumento, ¿Qué pasa si hacemos esto? Vamos a ejecutar:

```bash
read(0x3) = 31
write(1, "demo-ubuntu
demo-ubuntu
") = 31
read(0x3) = 0
```

Aquí tenemos un pequeño problema, resulta que hemos unido entrada y salida, pero lo que está ocurriendo es la entrada con el string `write(1, "demo-ubuntu` y antes de devolvernos la salida de la syscall ya se está mostrando por stdout la salida del comando `demo-ubuntu` quedando a continuación la salida en la que nos devuelve `") = 31`

Ojo ¿Somos los únicos que les pasa?

```bash
miguel@demo-ubuntu:~/minitrace$  strace cat /etc/hostname
execve("/usr/bin/cat", ["cat", "/etc/hostname"], 0x7ffdc2d6b3e8 /* 52 vars */) = 0
...
read(3, "demo-ubuntu", 131072) = 31
write(1, "demo-ubuntu", 31demo-ubuntu
) = 31
read(3, "", 131072)                     = 0
...
```

Parece que strace tiene el mismo problema, pero realmente strace lo tiene solucionado, aunque ahora mismo sea invisible para nosotros. Si volvemos a recordar [¿Qué son STDIN, STDOUT, STDERR?](/blog/stdin-stdout-stderr) 

Tenemos stdout, para todo aquello que es "útil" y que probablemente querramos enviar por pipe a otro sitio, mientras que stderr, pese a que mucha gente crea que hace relación a errores, no es así, es también cualquier información extra que pueda proporcionar un programa y que no querríamos enviar de forma normal.

Sabemos que stdout es el descriptor 1, vamos a probar algo:

```bash
miguel@demo-ubuntu:~/minitrace$  strace cat /etc/hostname 1>/dev/null
execve("/usr/bin/cat", ["cat", "/etc/hostname"], 0x7ffdc2d6b3e8 /* 52 vars */) = 0
...
read(3, "demo-ubuntu\n", 131072) = 31
write(1, "demo-ubuntu\n", 31) = 31
read(3, "", 131072)                     = 0
...
```

Al enviar al vacío infinito al stdout con `1>/dev/null` no tenemos ese problema y podemos ver la información mucho mejor por terminal. Por hacer una pequeña mejora podríamos implementar lo mismo en nuestro pequeño código.

```c
fprintf(stderr, ...)
```

Únicamente cambiamos nuestros printf por fprintf para indicar que salimos por stderr.

```bash
miguel@demo-ubuntu:~/minitrace$  ./minitrace cat /etc/hostname 1>/dev/null
...
read(0x3) = 31
write(1, "demo-ubuntu
") = 31
read(0x3) = 0
...
```
Vaya... Nos hemos librado del stdout, pero seguimos con un salto de línea, es porque realmente write hace esto `write(1, "demo...\n", 31)` de hecho se puede ver en la salida de strace `"demo-ubuntu\n"` 

De todas formas no nos preocupa, no estamos intentando hacer un strace desde cero 100% funcional, únicamente entender qué está ocurriendo bajo el capó, existen multitud de formas para solucionar este problema.

Yo creo que por hoy ha sido suficiente, nos vemos en la próxima.
