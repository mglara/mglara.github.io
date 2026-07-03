# Balanceando conexiones sin tocar la aplicación

Hoy voy a hablar de un tema un poco diferente al de los últimos posts, por variar un poco y porque por necesidad tuve que montar algo concreto que me hizo pensar que es interesante conocer esta parte de los sistemas Linux.

## IPtables

¿Sabes qué es iptables? ¿Alguna vez has jugado un poco con ello? Es una herramienta de línea de comandos que permite gestionar el firewall de Linux. Puedes interceptar, filtrar y modificar los datos que pasen por el equipo de múltiples formas.

Realmente no es el firewall en si, sino que es la herramienta que te permite interactur con NetFilter en el Kernel.

No quiero ponerme ahora a escribir una guía paso a paso de que es una tabla, una cadena o una regla, que de eso hay mucho por internet. Sólo decir que el uso más típico es para bloquear algunas conexiones en los bastionados o redirigir puertos específicos.

¿Y sabes que es NFtables?

## NFtables

Básicamente es el sucesor de iptables, unifica todas las herramientas que antes se requerían cuando querías gestionar de forma adecuada firewalls complejos en una sola. La sintaxis es más cómoda y pese a lo que podríamos pensar tiene un mayor rendimiento que IPtables, de hecho ¿Por qué no hacemos la comprobación un día? Pero hoy no que voy a hablar de algo distinto.

## Objetivo

Estoy hablando de todo esto porque me vi en la necesidad de hacer un filtrado de tráfico en un Linux, actuando como un router y además "balanceando" conexiones entre distintos túneles VPN para conseguir una anonimización adecuada.

> [!DANGER]
> DISCLAIMER: Aviso que lo que voy a explicar es prototipo y no se ha hecho un modelado de amenazas adecuado, no es productivo 100% y de hecho se que tiene algunos problemas base.

Incluso aún con este Disclaimer considero que es interesante para poder aprender un poquito más sobre el firewall de Linux y las cosas que podemos hacer con él.

## Comencemos a trabajar

Vamos a plantear cada paso de lo que necesitamos hacer, ¿Cuál es el objetivo que tenemos?

Vamos a tener varios túneles y necesitamos que cuando los clientes se conecten a nuestro "router" se divida el tráfico entre estos túneles de forma aleatoria sin cortar comunicaciones largas, esto último es importante tenerlo en cuenta para no ir paquete a paquete sino flujo a flujo.

Para hacer esto ¿Qué necesitamos? Pues que el paquete entre, se categorice de alguna forma y se envíe por un túnel aleatorio.

Una de las grandes diferencias entre nftables e iptables es que las cadenas en iptables son cerradas, mientras que en nftables tú decides, las nombras y usas donde tu quieras hacerlo.

Vamos a comenzar con una cadena que llamaremos prerouting, es un nombre descriptivo porque es la que usaremos nada más entren los paquetes en la tarjeta de red.

```
chain prerouting {

}
```

¿A continuación?

```
chain prerouting {
    type filter hook prerouting priority mangle; policy accept;
    ...
```

Estamos diciendo que será un filtro `type filter` y que se ejecute nada más entren los paquetes a la tarjeta de red `hook prerouting` justo antes de que el kernel elija ruta.
Aparte es importante el tema de las prioridades, recordemos que cuando hablamos de firewalls la prioridad marca qué ocurre con el paquete antes y qué ocurre después, si por ejemplo haces un drop de un paquete olvídate de poder seguir trabajando con él más adelante.
En este caso estamos hablando de nada más entrar, vamos a darle una prioridad muy temprana `priority mangle` que internamente es -150, así nos aseguramos de que toque el paquete antes que nadie y por supuesto después de hacer lo que queramos hacer con él le dejamos seguir pasando `policy accept`.

```
chain prerouting {
    ...
    ip saddr != @client_net return
    ...
```

Ahora estamos diciendo comprueba las direcciones de origen `ip saddr` y si no pertenece al conjunto `!= @client_net` entonces dejamos de procesar el paquete `return`.

### ¿Qué es un conjunto?

Un "conjunto" o "set" en nftables es una colección que vive en el kernel. Te permite no tener que repetir cierta información una y otra vez, además admite rangos y CIDR no sólamente IPs sueltas, en este caso usamos:

```
set client_net {
    type ipv4_addr
    flags interval
    elements = { 10.8.0.0/24 }
}
set internal_nets {
    type ipv4_addr
    flags interval
    elements = { 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 }
}
```

Es decir, un conjunto cliente_net que viene de un rango 10.8.0.0/24 y un conjunto internal_nets con los típicos rangos internos. En iptables por ejemplos las cadenas son cerradas, aquí podemos definir conjuntos y referenciarlo en todas las reglas que necesitemos.

```
chain prerouting {
    ...
    ct mark != 0 meta mark set ct mark return
    ...
```

Hay otras cosas que necesitamos, recordemos que no queremos trabajar paquete a paquete, mataría comunicaciones largas, por lo que tenemos que mirar el flujo, y para ello tenemos que mirar el estado actual de la conexión `ct` en este caso lo que estamos mirando es si ya tiene una marca y si la tiene, es decir, es distinta de cero `!= 0` copiamos esa marca al paquete que estamos estudiando.

**ct** Hace referencia al connection tracking, el kernel tiene que mantener siempre una tabla de conexiones, sino cada comunicación sería algo nuevo.

```
chain prerouting {
    ...
    ip daddr @internal_nets return
    ...
```

También otro punto de manejo importante, en este caso queremos únicamente trabajar sobre IPs externas, por eso ponemos una regla que dice si la ip destino `ip daddr` está dentro del conjunto de redes internas `@internal_nets` no hacemos nada más con ella.

```
chain prerouting {
    ...
    jump egress_pick
    ...
```

Esta línea directamente salta a una cadena que habrá que definir después llamada `egress_pick`. Aquí solo deberían llegar flujos nuevos, ya que hemos filtrado las que van a redes internas, las que no están permitidos o flujos que ya están marcados previamente.

```
chain prerouting {
    ...
    ct mark set meta mark
    ...
```

Al volver de egress_pick ya guardamos la marca permanentemente en el registro de conexión `ct mark`, el que antes mirábamos si ya estaba.
Ojo, esto ahora mismo solo tiene sentido cuando veamos qué estamos haciendo en egress_pick, ya que es donde vamos a poner una marca en el paquete, de hecho esta marca solo sirve para algo porque se usará para elegir tabla de rutas.

## Cadena egress_pick

Creo que esta cadena es una de las más importantes, porque es la que permitirá luego la redirección de flujos por distintos túneles. Además aunque ponga un ejemplo en el sistema final se rellena dinámicamente en función de los túneles "sanos" que tenemos.

```
chain egress_pick {
    meta mark set numgen random mod 3 map { 0 : 1, 1 : 2, 2 : 3 }
}
```

Primero ponemos una marca al paquete `meta mark set` y generamos un número aleatorio entre 0 y 2 `numgen random mod 3` en este caso concreto tenemos solo 3 túneles, pero en mi resultado final la idea era una veintena de túneles distintos, por eso se genera de forma dinámica con los "túneles sanos".

Una vez hemos generado un número aleatorio se hace el mapeo, es decir traducimos ese número a la marca de uno de los túneles `map { 0 : 1, 1 : 2, 2 : 3 }`

Claro... Para que esto funcione luego en producción tiene que ser dinámico de verdad, por lo que la forma que hice fue un borrado de la regla anterior, es decir, si tenemos 19 túneles sanos, se detecta, se borra y se rehace esta regla con sólo 19 túneles, para ello uso:

```
flush chain inet egress egress_pick
```

Se borra la regla vieja `flush` y se rehace. Porque además el `map` sólo va a contener túneles que han pasado un chequeo de salud.

## ¿Marcas vacías?

Realmente si nos ponemos a pensarlo no hemos hecho nada importante, es decir, hemos utilizado el firewall para marcar paquetes, pero una marca por si misma es algo vacío no hace nada...

Aquí viene otra parte interesante, la marca permite decir a Linux que use varias tablas de rutas y pueda decidir qué tabla consulta cada paquete según la marca que tiene establecida.

Repasemos un segundo, hemos puesto una marca aleatoria en un paquete `egress_pick` lo hemos grabado en la tabla de conexiones `ct mark set meta mark` impidiendo que cada paquete se modifique.

Ahora nos falta que los paquetes con marca X miren la tabla de rutas Y.

Cuando se levanta cada uno de los túneles, además de levantar una interfaz, que en este caso estoy usando WireGuard, se monta una tabla de rutas y una regla que lo asocie a una marca.

> [!WARNING]
> Si se hace así no debemos mantener en Wireguard sus propias rutas, debemos establecer `Table = off` porque si no lo gestionamos a mano nos volvemos locos.

Entonces por cada túnel vamos a añadir de forma dinámica, importante la palabra **dinámica** ya que en el servicio productivo monté unos servicios que hacen este trabajo mirando en todo momento si los túneles están "sanos". Me voy por las ramas, pero vamos a añadir de forma dinámica esto:

```
ip route replace default dev wgt01 table 101
ip rule add fwmark 1 table 101 priority 1001

ip route replace default dev wgt02 table 102
ip rule add fwmark 2 table 102 priority 1002

ip route replace default dev wgt03 table 103
ip rule add fwmark 3 table 103 priority 1003
...
```

Es decir, remplazamos la salida por defecto de cada una de las interfaces por una tabla concreta `replace default dev wgtXX table 1XX`.
Y además añadimos una regla que establece cada marca `add fwmark X` por una tabla `table 1XX` y la prioridad `priority 10XX`.
Esas `fwmark` son las que se mapean en el `egress_pick` en `map { 0 : 1, 1 : 2, 2 : 3 }`

> [!TIP]
> Cuando hablamos de meta mark (nftables) y fwmark hablamos exactamente de lo mismo, la marca de firewall del paquete pero vista por dos herramientas distintas.

En nuestro sistema ya montado si lanzamos el comando:

```
ip rule show
0:      from all lookup local
1001:   from all fwmark 0x1 lookup 101
1002:   from all fwmark 0x2 lookup 102
1003:   from all fwmark 0x3 lookup 103
32766:  from all lookup main
32767:  from all lookup default
```

Vemos todo esto que estamos diseñando, la marca asociada a cada una de las tablas que permitirá los distintos enrutamientos, y ¿Qué enrutamientos son?

```
ip route show table 101
default dev wgt01 scope link 
```

Sumando ambas sintaxis está diciendo para toda marca de tipo 0x1 mira la tabla 101 `from all fwmark 0x1 lookup 101` y todo lo que vaya por la tabla de enrutamiento 101 sale por la interfaz wgt01 `default dev wgt01`.

Pero podemos pedir aún más información al kernel, que es el tipo listo:

```
ip route get 1.1.1.1 mark 2
1.1.1.1 dev wgt02 table 102 src XX.XX.XX.XX mark 2 uid 1000
```

¿Por donde saldría un paquete dirigido a 1.1.1.1 que lleve la marca 2? `ip route get 1.1.1.1 mark 2` saldría por la interfaz wgt02 `dev wgt02` que corresponde a la tabla 102 `table 102`.

Además de esto el interfaz de cada túnel quedaría algo así, para quitar el DNS y forzar el enrutado:

```
[Interface]
Table = off
Address = 10.x.x.x
PrivateKey = ...

[Peer]
PublicKey = ...
Endpoint = ...
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

**REPASEMOS**

Por cada túnel se levanta una interfaz, cada interfaz se lleva a una tabla y se asocia una `fwmark` a cada interfaz.
Entra un paquete, se comprueba que esté permitido, que vaya al exterior y que no esté marcado dentro de un flujo previo. Si no está marcado se marca y se mapea con las `fwmark` que son las que controlan por qué interfaz saldrá.
Esta marca se mete en la tabla de conexiones para que no cambie cada paquete y sea por flujo y voilà lo tenemos.

En este punto es donde es interesante conocer herramientas de troubleshooting y que nos permitan ver estas cosas de forma fácil y rápida.

Por ejemplo, si yo monto el sistema y hago un `conntrack -L` o `conntrack -E` me encuentro con algo así:

```
tcp      6 118 TIME_WAIT src=10.8.0.99 dst=1.1.1.1 sport=36370 dport=443 src=1.1.1.1 dst=10.66.0.22 sport=443 dport=36370 [ASSURED] mark=2 use=1
```

Esto aparece tras hacer `curl -s https://1.1.1.1/cdn-cgi/trace` vamos a estudiar la salida entonces. Pone que ahora mismo se ha hecho una petición desde la ip `10.8.0.99` con destino a la `1.1.1.1` al puerto `443`claro es un curl https, y después sin embargo tenemos que el src `1.1.1.1` con dst `10.66.0.22`, claro... porque se ha hecho NAT, que lo explico un poco más adelante, por lo que el destino al que va a responder es otro, de hecho ni siquiera es el destino real que está viendo el sistema `1.1.1.1` que estará viendo una IP pública a la que corresponderá, esto es lo que nos va a llegar a nosotros para luego poder hacer la traducción.

Realmente solo he puesto una línea pero si hacemos `conntrack -E` y lanzamos por ejemplo un `curl ifconfig.me`

```
[NEW] udp      17 30 src=10.8.0.99 dst=1.1.1.1 sport=58317 dport=53 [UNREPLIED] src=1.1.1.1 dst=10.66.0.11 sport=53 dport=58317 mark=1
    [NEW] udp      17 30 src=10.8.0.99 dst=1.1.1.1 sport=59648 dport=53 [UNREPLIED] src=1.1.1.1 dst=10.66.0.11 sport=53 dport=59648 mark=1
 [UPDATE] udp      17 30 src=10.8.0.99 dst=1.1.1.1 sport=58317 dport=53 src=1.1.1.1 dst=10.66.0.11 sport=53 dport=58317 mark=1
 [UPDATE] udp      17 30 src=10.8.0.99 dst=1.1.1.1 sport=59648 dport=53 src=1.1.1.1 dst=10.66.0.11 sport=53 dport=59648 mark=1
    [NEW] tcp      6 120 SYN_SENT src=10.8.0.99 dst=34.160.111.145 sport=34290 dport=80 [UNREPLIED] src=34.160.111.145 dst=10.66.0.22 sport=80 dport=34290 mark=2
 [UPDATE] tcp      6 60 SYN_RECV src=10.8.0.99 dst=34.160.111.145 sport=34290 dport=80 src=34.160.111.145 dst=10.66.0.22 sport=80 dport=34290 mark=2
 [UPDATE] tcp      6 432000 ESTABLISHED src=10.8.0.99 dst=34.160.111.145 sport=34290 dport=80 src=34.160.111.145 dst=10.66.0.22 sport=80 dport=34290 [ASSURED] mark=2
 [UPDATE] tcp      6 120 FIN_WAIT src=10.8.0.99 dst=34.160.111.145 sport=34290 dport=80 src=34.160.111.145 dst=10.66.0.22 sport=80 dport=34290 [ASSURED] mark=2
 [UPDATE] tcp      6 30 LAST_ACK src=10.8.0.99 dst=34.160.111.145 sport=34290 dport=80 src=34.160.111.145 dst=10.66.0.22 sport=80 dport=34290 [ASSURED] mark=2
 [UPDATE] tcp      6 120 TIME_WAIT src=10.8.0.99 dst=34.160.111.145 sport=34290 dport=80 src=34.160.111.145 dst=10.66.0.22 sport=80 dport=34290 [ASSURED] mark=2
```

Estamos viendo tanto la conexión UDP por puerto 53 para pedir la IP del dominio `ifconfig.me` como luego todo el flujo de comunicación con su marca durante todo el flujo.

También a la hora de trabajar con nft nos interesa utilizar el comando `nft monitor trace` pero para ello tenemos que marcarlo:

```
# Marcamos los paquetes de nuestro cliente para poder seguirlos
nft insert rule inet egress prerouting ip saddr 10.8.0.99 meta nftrace set 1
nft monitor trace

# Primer paquete del flujo (el SYN): es nuevo, hay que elegir tunel
trace id 5b35cea7 inet egress prerouting packet: iif "ens18" ip saddr 10.8.0.99 ip daddr 34.160.111.145 ip protocol tcp tcp dport 80 tcp flags == syn ip ttl 64
trace id 5b35cea7 inet egress prerouting rule jump egress_pick (verdict jump egress_pick)
trace id 5b35cea7 inet egress egress_pick rule meta mark set numgen random mod 3 map { 0 : 0x00000001, 1 : 0x00000002, 2 : 0x00000003 } (verdict continue)
trace id 5b35cea7 inet egress prerouting rule ct mark set meta mark (verdict continue)
trace id 5b35cea7 inet egress prerouting policy accept meta mark 0x00000003
trace id 5b35cea7 inet egress forward packet: iif "ens18" oif "wgt03" ip saddr 10.8.0.99 ip daddr 34.160.111.145 ip protocol tcp tcp dport 80 tcp flags == syn ip ttl 63
trace id 5b35cea7 inet egress forward rule ip saddr @client_net oifname "wgt*" accept (verdict accept)
trace id 5b35cea7 inet egress postrouting rule oifname "wgt*" masquerade (verdict accept)

# Un paquete posterior del MISMO flujo (un ACK): ya viene marcado
trace id ae5021ef inet egress prerouting packet: iif "ens18" ip saddr 10.8.0.99 ip daddr 34.160.111.145 ip protocol tcp tcp dport 80 tcp flags == ack ip ttl 64
trace id ae5021ef inet egress prerouting rule ct mark != 0x00000000 meta mark set ct mark return (verdict return)
trace id ae5021ef inet egress prerouting policy accept meta mark 0x00000003
trace id ae5021ef inet egress forward rule ct state established,related accept (verdict accept)

# ... el resto de paquetes del flujo repiten este segundo camino porque la marca ya está fijada
```

Cada `trace id` es el viaje de UN paquete por el firewall, regla a regla, con el veredicto de cada una. He recortado la salida a dos paquetes representativos y le he quitado el ruido (MACs, ids, puertos de origen, ventana...) para que se entienda.

Mirad el primer paquete, el `SYN` que abre la conexión: llega a `prerouting`, no tiene marca previa, así que cae en el `jump egress_pick`. Ahí el `numgen` le asigna `meta mark 0x00000003`, la guardamos en la conexión con `ct mark set meta mark` y el paquete sale del prerouting ya con su marca. Justo después el kernel decide la ruta, y por eso al entrar en `forward` ya aparece `oif "wgt03"` ya que la marca 3 lo ha mandado por el túnel wgt03. Lo acepta el kill-switch y en `postrouting` se le hace el `masquerade`.

El segundo paquete es un `ACK` de esa misma conexión, y aquí se ve cómo funcionamos por flujo y no por paquete, como ya tiene marca, entra por la regla `ct mark != 0 ... return`, copia la marca guardada y sale del prerouting sin volver a pasar por `egress_pick`. En forward lo acepta directamente por `ct state established,related`.

Importante si lo estás haciendo en tu sistema necesitas borras la regla, mira a ver cual es el handle con `nft -a list chain inet egress prerouting` y luego `nft delete rule inet egress prerouting handle N`,


## Cadena forward

```
chain forward {
    type filter hook forward priority filter; policy drop;
    ...
```

Creamos una nueva cadena, realmente lo que va a hacer es analizar los paquetes que atraviesan el equipo pero que no tienen como objetivo el propio "router"

Aquí también marcamos que sea un filtro `type filter` y la engancha al forward, por eso le pusimos además ese nombre para que se entienda correctamente `hook forward`. El kernel lo que va a hacer es ¿Es para este equipo? No, pues entonces va a forward y postrouting, Si, pues entonces va a INPUT.

No le ponemos tanta prioridad como las anteriores, para ello usamos `priority filter` y ojo porque si un paquete llega al final sin que ninguna regla lo acepte entonces lo dropeamos, nos deshacemos de él `policy drop`.

> [!TIP]
> A cualquier regla puedes añadirle un counter, y nftables te va a decir cuántos paquetes la han pisado 'ct state invalid counter drop' podría permitir contar todos los estados inválidos y luego consultarlo con 'nft list chain inet egress forward'. La verdad que esto os lo estoy contando porque necesité hacerlo por algunas meteduras de pata iniciales que tuve y necesité hacer un poco de troubleshooting.

```
chain forward {
    ...
    ct state established,related accept
    ...
```

Entonces miramos el estado dentro de la tabla de conexiones `ct state established` si hay una conexión existente o si no es la misma pero está relacionada `related` que puede ocurrir en cierto tipo de conexiones.

En caso de que esté establecido o relacionado pues aceptamos `accept`. Recordemos que teníamos un denegado por defecto en la cadena, es importante tener en cuenta qué aceptamos entonces.

```
chain forward {
    ...
    ct state invalid drop
    ...
```

Importante, miramos la tabla de conexiones y buscamos inválido para dropearlo, realmente no es necesario hacer esto, porque al ser un estado inválido y no haberlo permitido se haría igual el drop, pero al mismo tiempo es muy importante.

Un estado inválido puede hacer referencia a paquetes malformados, fuera de secuencia o cualquier aspecto "raro" del paquete.

Con esto, si a futuro se añaden otras reglas, como por ejemplo aceptar icmp, aceptaríamos también los estados inválidos y queremos evitarlo, disminuimos error humano.

Por otro lado podría mejorar la eficiencia, ya que hacemos drop y se dejan de comprobar el resto de reglas, en esta situación con lo poco que tenemos no va a afectar drásticamente pero quizás si en un futuro.

```
chain forward {
    ...
    ip saddr @client_net oifname "wgt*" accept
    ...
```

Esto es importante, ya que va a tener relación con la forma en la que queremos que trabaje, si la dirección origen está en el conjunto de redes que permitíamos usar este "router" `ip saddr @client_net` y la interfaz por la que va a salir es cualquiera que comience por wgt `oifname` lo aceptamos `accept`.

Muy importante, estamos viendo el nombre de la interfaz de output, de esta forma evitamos que si algo se ha escapado y va a salir por donde no debe pueda hacerlo, gracias al `drop` inicial.


## Postrouting

Ya no queremos seguir en el punto de filtrado, entramos en mundo NAT.

```
chain postrouting {
    type nat hook postrouting priority srcnat; policy accept;

    oifname "wgt*" masquerade
}
```

Aquí al ser cortito lo pongo entero del tirón, primero especificamos que es de tipo NAT `type nat`, es decir, no está pensada para filtrar sino para hacer modificaciones de dirección y lo enganchamos a las reglas de postrouting `hook postrouting`. Estas reglas se ejecutan cuando el Kernel ya ha decidido por donde saldrá el paquete.

> [!TIP]
> Para muchos no hará falta que lo explique pero por si a alguien NAT le suena a chino intento hacer explicación simple. Si tu te comunicas desde una IP interna 10.8.0.12 a internet, cuando quiera volver el paquete no va a saber donde está 10.8.0.12, un problema grande... Por lo que se hace un NAT, es decir se cambia por una IP que SI que conoce, normalmente el router de casa lo cambiará por tu IP pública, permitiendo que sepa volver. ¿Cómo creéis que ocurriría si tiene un ordenador detrás de varios routers? Gracias a esto funciona internet.

La prioridad que usamos es srcnat `priority srcnat` todas estas prioridades realmente son simbólicas y hacen referencia a un número concreto, por ejemplo, en este caso es aproximadamente 100 mientras que en mangle es -150.

Cuando aquí decimos que aceptamos `policy accept` es porque por definición su función no es dropear paquetes, sino hacer modificaciones.

Y esta parte es importante `oifname "wgt*" masquerade` si la interfaz de salida es uno de los wgt entonces se hace un `masquerade`que quiere decir justo lo que hemos mencionado antes con lo de NAT. De esta forma lo que haremos será traducir la IP de la que viene por la IP de salida que está utilizando la interfaz wgt por la que sale.
Después sabrá devolver la respuesta porque el kernel guarda en su tabla de conexiones la traducción.

La diferencia entre usar `masquerade`o usar `snat` es que con snat tendrías que escribir la IP concreta, necesitamos algo dinámico.

Es decir:

El paquete entra -> PREROUTING (mangle, -150) Aquí marcamos -> Decisión de ruta por kernel, aquí mira `ip rule` y el `fwmark` para elegir la tabla -> FORWARD (filter) Donde pondremos el kill-switch -> POSTROUTING (nat, srcnat) donde añadiremos la máscara para enrutar.

## IPv6

Realmente todas las reglas anteriores estaban dentro de la tabla inicial, ahora mismo es tabla de IPv6 y tenemos una política importante, todo lo que sea `IPv6` en `FORWARD` lo ignoramos `DROP`.

De todas formas esto también lo vamos a modificar a nivel de sistema con sysctl donde añadimos:

```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
```

Estas reglas de sistema deshabilitan ipv6 directamente a nivel de kernel.


## Configuración del sistema (sin esto nada enruta)

Todo lo anterior asume que el equipo está configurado como router. Eso no viene "de fábrica"; hay que tocar el kernel vía `sysctl`. Lo importante:

```
net.ipv4.ip_forward = 1
```
Hay que habilitar si o si el reenvío de paquetes, sin esto no hay FORWARD.

```
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.default.rp_filter = 2
```
El rp_filter debe colocarse en modo loose (2), ya que al tener varias tablas de rutas la ruta de vuelta no es simétrica, sino nos tira los paquetes.

Hay que tener en cuenta que cuando se hace un enrutado mediante marcas el tráfico entra por una interfaz y vuelve por otra.

```
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
```
No se permiten redirecciones ICMP ni reescrituras de rutas.

```
net.netfilter.nf_conntrack_max = 1048576
```
Y un tema importante de rendimiento, necesitamos aumentar el límite de la tabla de conexiones, vamos a necesitar el track de muchos flujos y no queremos quedarnos cortos.

## DNS Leak, cuidadito

Aquí hay algo que muchísima gente pasa por alto y que tira por tierra toda la anonimización: el DNS.

Imagina que has montado todo lo anterior, el tráfico de tus clientes sale precioso y aleatorio por una veintena de túneles... pero el cliente, para saber a qué IP conectarse, primero **resuelve el dominio**. Y si esa resolución la hace contra el DNS de siempre, da igual todo lo demás se tiene la lista completa de a qué dominios estás entrando, con tu IP real. Esto se llama **DNS leak** ya que el tráfico va por un lado y las preguntas DNS por otro.

Para evitarlo, en el "router" levantamos nuestro propio "resolutor" `unbound` y obligamos a que las consultas pasen por él.

Lo tenemos escuchando en el puerto 53 y rechaza a cualquier que no sea localhost o un cliente aceptado:

```
access-control: 127.0.0.0/8 allow
access-control: 10.8.0.0/24 allow     
access-control: 0.0.0.0/0   refuse
```

Y en el firewall hay una parte que no he mencionado porque tenía también otros detalles de bastionado que no venían a cuento, sólo se abre el 53 al tráfico que viene de la red de clientes aceptados.

Pero claro... Esto no sirve de nada si luego la consulta que hace el router no van cifradas. Para esto usamos DNS sobre TLS en la configuración de unbound como os enseño a continuación:

```
forward-zone:
    name: "."
    forward-tls-upstream: yes
    forward-addr: ...
```

Además le decimos a unbound que no filtre nombres internos:

```
private-address: 10.0.0.0/8
private-address: 172.16.0.0/12
private-address: 192.168.0.0/16
```

Sumado a `qname-minimisation` (preguntar lo mínimo en cada salto), `hide-identity` y `hide-version`, el resolutor cuenta lo justo.
Es cierto que ahora mismo no estamos forzando la salida por este resolutor, porque además todavía tendríamos que forzar que las consultas DNS vayan por allí y que luego se enruten por los túneles, cómo en este caso las propias consultas de los cliente se van a enrutar no es necesario.

De hecho en nuestra configuración inicial lo que de verdad evita el leak no es unbound sino el kill-switch que tenemos en el FORWARD.

## NOTA IMPORTANTE

Cómo he dicho en el DISCLAIMER inicial, no estoy contando algo 100% productivo, de hecho no os he pasado las configuraciones automatizadas para montarlo. Tened en cuenta que si utilizáis tal y cómo hemos visto para anonimato este sistema vais a correlar todas vuestras IPs de las VPNs que utilicéis a un mismo origen, ya que aunque estamos filtrando por flujo no lo estamos haciendo por navegación, es decir, distintas peticiones a la misma página saldrán desde distintas IPs pero con características similares que permitirían su correlación.

En el sistema en el que se va a usar eso ya se maneja por las conexiones finales de salida con otros proveedores, por lo que no es un problema real, pero usando únicamente esto si que sería un problema. De todas formas tenéis la información suficiente para poder seguir trabajando en solucionar ese problema y convertir esta información en un sistema de anonimización adecuado.

Por otro lado lo ideal sería también decidir si el propio "router" se va a autoanonimizar, algo que en este escenario tampoco se ha hecho, todas las conexiones del propio "router" salen por su IP pública.

De todas formas aunque podría explicarlo no voy a hacerlo, sólo voy a comentar que utilizaría `jhash` para solucionar ese problema de identidad correlacionada. Aunque sigue teniendo un problemilla... Pero esto para otro momento, o para vuestras noches de insomnio.