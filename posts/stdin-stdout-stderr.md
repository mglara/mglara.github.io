# ¿Qué son STDIN, STDOUT, STDERR?

Cuando hablamos de estos conceptos, nos referimos a los flujos de datos estándar que gestionan la entrada y salida de información en los programas.

## STDIN

**STDIN**, o entrada estándar, es el flujo por donde el programa recibe datos, que por defecto provienen del teclado. Sin embargo, podemos redirigir esta entrada desde un archivo u otras fuentes.

Por ejemplo, si creamos un archivo llamado comandos con el siguiente contenido:

```bash
echo "echo hola" > comandos
echo "netstat -tonap" >> comandos
echo "exit" >> comandos
```

Podemos lanzar una shell utilizando el contenido del archivo como STDIN:

```bash
$ sh < ./comandos

hola
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name     Timer
tcp        0      0 0.0.0.0:139             0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 0.0.0.0:445             0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0    392 192.168.0.22:22         192.168.0.11:54945      ESTABLISHED -                    on (0,04/0/0)
tcp6       0      0 :::139                  :::*                    LISTEN      -                    off (0.00/0/0)
tcp6       0      0 :::22                   :::*                    LISTEN      -                    off (0.00/0/0)
tcp6       0      0 :::445                  :::*                    LISTEN      -                    off (0.00/0/0)
```

Como podéis ver, hemos logrado que el programa obtenga el STDIN directamente del archivo que creamos.

## STDOUT

STDOUT, o salida estándar, es donde se envían los resultados de un programa. Por defecto, esta salida se muestra en la consola, pero también puede ser redirigida a otros destinos. Si habéis trabajado con Linux, probablemente hayáis hecho redirecciones incluso sin daros cuenta de lo que estabáis haciendo.

Por ejemplo, si queremos hacer un ping y guardar los resultados en un archivo, podemos usar:

```bash
$ ping -c 3 127.0.0.1 1> resultado
$ cat resultado

PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.035 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.037 ms
64 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.022 ms

--- 127.0.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2037ms
rtt min/avg/max/mdev = 0.022/0.031/0.037/0.006 ms
```

Para redirigir el flujo de STDOUT, comúnmente utilizamos `1>` o simplemente `>`.

## STDERR

**STDERR**, o error estándar, es el flujo que maneja los mensajes de error de un programa. Por defecto, estos mensajes también se envían a la consola, pero pueden ser redirigidos a otros destinos.

> [!WARNING]
> Aunque por defecto es la salida de error estándar mucho ojo con esto, porque realmente se utiliza para más cosas, mensajes de diagnóstico, estado, advertencias. Muchos binarios juegan con esto para poder mantener limpio un pipe | o una redirección >


Para redirigir STDERR, usamos 2>. Por ejemplo:

```bash
$ cat noexisto 2> resultado
$ cat resultado

cat: noexisto: No existe el fichero o el directorio
```

De hecho podríamos redirigir ambas salidas, tanto STDOUT como STDERR a distintos lugares:

```bash
./miprograma.sh 1> resultadosSTDIN 2> resultadosSTDOUT

# O incluso al mismo lugar
./miprograma.sh > resultadosSTDIN 2>&1
```

En este último caso, `2>&1` significa que se está redireccionando el STDERR a resultadoSTDIN, lo que estás diciendo es que quieres que el STDERR `2>`se redirija al descriptor de archivo `1` y por eso se utiliza el operador `&` delante.

| Descriptor    | Salida              |
| ---------: | :------------------- |
| 0 | STDIN                 |
| 1        | STDOUT |
| 2        | STDERR |

Siempre me ha llamado la atención que existen programas basados en consola especialmente en Linux donde mandan al STDERR la información del propio programa, es decir, esos mensajes que no son por defecto errores como tal. De tal forma que si haces una redirección de STDOUT no te llevas esa información a la salida y la mantiene más limpia.

De hecho podemos crear nuestro propio flujo de trabajo en código que nos va a permitir hacer algo así, pongamos como ejemplo que yo tengo un programa que va a realizar un scan de puertos, pero además de esto me va a dar cierta información, puedes llamarla información DEBUG o simplemente para saber cómo va avanzando el programa.

```bash
#!/bin/bash

# Manejo de la señal de interrupción (Ctrl+C)
trap ctrl_c INT

# Función que se ejecuta al recibir una interrupción
function ctrl_c() {
    echo -e "\n\n[*] CANCELANDO... [*]\n" >&2
    exit 0
}

# Verifica que se haya proporcionado un argumento (IP o dominio)
if [ -z "$1" ]; then
    echo "Uso: $0 <IP o dominio>" >&2
    exit 1
fi

# Información sobre el inicio del escaneo
echo "[*] Iniciando escaneo de puertos en $1... [*]" >&2

# Escaneo de puertos
for port in $(seq 0 65535); do
    echo "[*] Escaneando puerto $port... [*]" >&2
    timeout 1 bash -c "echo '' < /dev/tcp/$1/$port" 2>/dev/null && echo "$port;UP" &
done

# Espera a que todos los procesos de escaneo terminen
wait

# Información sobre la finalización del escaneo
echo "[*] Escaneo completado. [*]" >&2
```

Vamos a revisar el output del código anterior:

```bash
$ ./scan.sh 127.0.0.1
[*] Iniciando escaneo de puertos en 127.0.0.1... [*]
[*] Escaneando puerto 0... [*]
[*] Escaneando puerto 1... [*]
[*] Escaneando puerto 2... [*]
[*] Escaneando puerto 3... [*]
...
[*] Escaneando puerto 50... [*]
22;UP
```

Da demasiada información, probablemente lo suyo sería que vaya sobreescribiendo el mensaje anterior para no ensuciar tanto el output, pero recordad que estamos con motivos de demo 😜.

Si ahora probamos a lanzar el scan de la siguiente forma:

```bash
$ ./scan.sh 127.0.0.1 > puertos 2> debug
```

Y hacemos un cat tanto a puertos como a debug:

```bash
$ cat puertos     

22;UP

139;UP

445;UP

$ cat debug  
[*] Iniciando escaneo de puertos en 127.0.0.1... [*]
[*] Escaneando puerto 0... [*]
[*] Escaneando puerto 1... [*]
[*] Escaneando puerto 2... [*]
[*] Escaneando puerto 3... [*]
...
```

Efectivamente ya os habéis dado cuenta, al haber seleccionado lo que queríamos que fuese la salida estándar y que mensajes forman parte de la salida de error podemos mantener limpio el output sin necesidad de generar una sección de almacenamiento de datos en fichero en nuestro pequeño script. Cómo os digo esta técnica es usada en muchos programas de consola.

## Conclusión

Entender cómo funcionan los flujos de datos estándar facilita el uso de comandos en diversas situaciones, especialmente al manejar errores. Este artículo solo rasca la superficie; hay mucho más por descubrir. De hecho os recomiendo que os paséis por [Redireccionamiento a partir de FD](/blog/redireccionamiento-stdin-stdout-stderr) para profundizar un poco más en estos conceptos.
