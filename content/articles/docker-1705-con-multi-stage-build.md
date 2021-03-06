Title: Docker 17.05 con multi-stage build
Slug: docker-1705-con-multi-stage-build
Date: 2017-05-22 10:00
Category: Operaciones
Tags: docker, dockerfile, multistage, build



Hacía tiempo que esperaba ansiosamente la nueva versión de **docker**. La raíz de tanta expectación son las mejoras que la versión candidata anunciaba, especialmente el nuevo modelo de *build*. Se ha modificado los *Dockerfile* para que puedan generar varias imágenes en un solo fichero, algunas de ellas partiendo de otras.

La gran mejora de la versión consiste en poder especificar varias veces el *keyword* **FROM**, para poder crear imágenes varias en el mismo fichero. Para ello, hay dos mejoras sustanciales en la sintaxis de los *keywords* **FROM** y **COPY**.

La idea es que puedes crear imágenes intermedias, para que luego algunas otras hereden de ella e incluso se puedan pasar ficheros entre sí. Eso se consigue con dos cambios nuevos:

* El *keyword* **FROM** puede incluir un *keywork* **AS**, que va a permitir heredar a las demás imágenes a partir de este *keyword*.
* El *keyword* **COPY** puede llevar un *flag* **--from** que permite copiar ficheros entre las imágenes creadas por el mismo *Dockerfile*.

Un ejemplo rápido:

```bash
gerard@aldebaran:~/docker/ejemplo$ cat Dockerfile 
FROM alpine:3.5 AS base
CMD ["cat", "/greeting"]

FROM base AS victim
RUN echo gold > /stealme
RUN echo hello > /greeting

FROM base
COPY --from=victim /stealme /
RUN echo bye > /greeting
gerard@aldebaran:~/docker/ejemplo$ 
```

Construimos la imagen como de costumbre:

```bash
gerard@aldebaran:~/docker/ejemplo$ docker build -t ejemplo .
Sending build context to Docker daemon  2.048kB
Step 1/8 : FROM alpine:3.5 AS base
 ---> 4a415e366388
Step 2/8 : CMD cat /greeting
 ---> Running in c27e95810829
 ---> 36e7f6d769bd
Removing intermediate container c27e95810829
Step 3/8 : FROM base AS victim
 ---> 36e7f6d769bd
Step 4/8 : RUN echo gold > /stealme
 ---> Running in 1ea9e0668bb7
 ---> e0dc8c579221
Removing intermediate container 1ea9e0668bb7
Step 5/8 : RUN echo hello > /greeting
 ---> Running in e0da2940311e
 ---> b2f3845ad8ff
Removing intermediate container e0da2940311e
Step 6/8 : FROM base
 ---> 36e7f6d769bd
Step 7/8 : COPY --from=victim /stealme /
 ---> 918c21e97a6b
Removing intermediate container 4846e3cc50a7
Step 8/8 : RUN echo bye > /greeting
 ---> Running in e3ad8598d14d
 ---> e451f40afe17
Removing intermediate container e3ad8598d14d
Successfully built e451f40afe17
Successfully tagged ejemplo:latest
gerard@aldebaran:~/docker/ejemplo$ 
```

Podemos ver que los pasos 3 y 6 parten ambos de la imagen *base*, mientras que la imagen final, copia un fichero de la imagen *victim*, de la que no hereda siquiera.

```bash
gerard@aldebaran:~/docker/ejemplo$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
ejemplo             latest              e451f40afe17        About a minute ago   3.99MB
<none>              <none>              b2f3845ad8ff        About a minute ago   3.99MB
alpine              3.5                 4a415e366388        2 months ago         3.99MB
gerard@aldebaran:~/docker/ejemplo$ docker run --rm b2f3845ad8ff
hello
gerard@aldebaran:~/docker/ejemplo$ docker run --rm ejemplo
bye
gerard@aldebaran:~/docker/ejemplo$ 
```

Otro detalle curioso es que el *tag* solo se puede dar a la imagen del último **FROM** del *Dockerfile*. Hay que tener en cuenta que si una imagen falla al construir, no se sigue con las demás.

## Posibles usos

### Pipelines de test

Cuando una imagen falla, el resto de imágenes no se construyen. Podemos aprovechar este punto para crear la imagen de test, ejecutando los tests necesarios. De esta forma, si los tests fallan, el *build* falla y no se genera la imagen posterior de *runtime*.

Supongamos este *Dockerfile*:

```bash
gerard@aldebaran:~/docker/pipeline$ cat Dockerfile 
FROM alpine:3.5 AS base
RUN apk add --no-cache python2
COPY script.py /

FROM base
COPY tests.py /
RUN python tests.py

FROM base
CMD ["python", "script.py"]
gerard@aldebaran:~/docker/pipeline$ 
```

Supongamos el el *script* de test falla; para ello he puesto un *script* que devuelve siempre un código de retorno "1".

```bash
gerard@aldebaran:~/docker/pipeline$ docker build -t release .
Sending build context to Docker daemon  4.096kB
...  
Step 6/8 : RUN python tests.py
 ---> Running in 4572bed9ffb5
Tests FAILED
The command '/bin/sh -c python tests.py' returned a non-zero code: 1
gerard@aldebaran:~/docker/pipeline$ 
```

No se genera ningún *tag*, porque uno de los pasos ha fallado. De esta forma, no tenemos *release* porque tenerla no sirve de nada: está rota.

Veamos ahora lo que pasa si el test tiene éxito:

```bash
gerard@aldebaran:~/docker/pipeline$ docker build -t release .
Sending build context to Docker daemon  4.096kB
...  
Step 6/8 : RUN python tests.py
 ---> Running in 5a8908410927
Tests OK
 ---> 2cd640cc2122
Removing intermediate container 5a8908410927
Step 7/8 : FROM base
 ---> 9a6cafb2fd49
Step 8/8 : CMD python script.py
 ---> Running in bad18c8f4f27
 ---> 6221630d5ac3
Removing intermediate container bad18c8f4f27
Successfully built 6221630d5ac3
Successfully tagged release:latest
gerard@aldebaran:~/docker/pipeline$ 
```

Y en este caso tenemos una release, con una imagen de *runtime*, sin los tests, y con la directiva **CMD** lista para ser usada.

### Reducción de capas

A todos nos ha pasado que copiamos unos ficheros en nuestras imágenes, y tras aplicarles modificaciones de permisos y de usuarios, esos ficheros ocupan el doble o más. Es inevitable. La idea es poder utilizar una imagen grande para adecuar nuestros fichero y luego copiarlos a una imagen en su forma final.

Supongamos el siguiente ejemplo, donde *bigfile* es un fichero de 100mb con permisos 644 y que pertenece a mi usuario:

```bash
gerard@aldebaran:~/docker/bigfiles$ cat Dockerfile 
FROM busybox AS builder
COPY bigfile /
RUN chmod 777 /bigfile && \
    chown nobody:nogroup /bigfile

FROM busybox
COPY --from=builder /bigfile /
CMD ["ls", "-lh", "/"]
gerard@aldebaran:~/docker/bigfiles$ 
```

Construimos la imagen:

```bash
gerard@aldebaran:~/docker/bigfiles$ docker build -t final .
Sending build context to Docker daemon  104.9MB
Step 1/6 : FROM busybox AS builder
latest: Pulling from library/busybox
7520415ce762: Pull complete 
Digest: sha256:32f093055929dbc23dec4d03e09dfe971f5973a9ca5cf059cbfb644c206aa83f
Status: Downloaded newer image for busybox:latest
 ---> 00f017a8c2a6
Step 2/6 : COPY bigfile /
 ---> 0a27ff02824b
Removing intermediate container 56007f1c8e9d
Step 3/6 : RUN chmod 777 /bigfile &&     chown nobody:nogroup /bigfile
 ---> Running in b8ff92d84014
 ---> bf3600cb8967
Removing intermediate container b8ff92d84014
Step 4/6 : FROM busybox
 ---> 00f017a8c2a6
Step 5/6 : COPY --from=builder /bigfile /
 ---> 57b8aa7c2aeb
Removing intermediate container e2ec30bfd912
Step 6/6 : CMD ls -lh /
 ---> Running in a212d95bb65f
 ---> c672e1ad3fcb
Removing intermediate container a212d95bb65f
Successfully built c672e1ad3fcb
Successfully tagged final:latest
gerard@aldebaran:~/docker/bigfiles$ 
```

El resultado es sorprendente: Nos ahorramos el duplicado del fichero *bigfile*, y este conserva los permisos que le habíamos indicado, pero **el usuario sigue siendo root**.

```bash
gerard@aldebaran:~/docker/bigfiles$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
final               latest              c672e1ad3fcb        2 minutes ago       106MB
<none>              <none>              bf3600cb8967        2 minutes ago       211MB
busybox             latest              00f017a8c2a6        2 months ago        1.11MB
gerard@aldebaran:~/docker/bigfiles$ docker history bf3600cb8967
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
bf3600cb8967        4 minutes ago       |0 /bin/sh -c chmod 777 /bigfile &&     ch...   105MB               
0a27ff02824b        4 minutes ago       /bin/sh -c #(nop) COPY file:777847f3f03c68...   105MB               
00f017a8c2a6        2 months ago        /bin/sh -c #(nop)  CMD ["sh"]                   0B                  
<missing>           2 months ago        /bin/sh -c #(nop) ADD file:c9ecd8ff00c653f...   1.11MB              
gerard@aldebaran:~/docker/bigfiles$ docker history final
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
c672e1ad3fcb        4 minutes ago       /bin/sh -c #(nop)  CMD ["ls" "-lh" "/"]         0B                  
57b8aa7c2aeb        4 minutes ago       /bin/sh -c #(nop) COPY file:5e392ba9fe1d0a...   105MB               
00f017a8c2a6        2 months ago        /bin/sh -c #(nop)  CMD ["sh"]                   0B                  
<missing>           2 months ago        /bin/sh -c #(nop) ADD file:c9ecd8ff00c653f...   1.11MB              
gerard@aldebaran:~/docker/bigfiles$ docker run --rm final | grep bigfile
-rwxrwxrwx    1 root     root      100.0M May  9 13:53 bigfile
gerard@aldebaran:~/docker/bigfiles$ 
```

Esto limita mucho el uso de esta solución.

### Build container pattern

Construir nuestros artefactos en la misma imagen que los va a ejecutar es una guarrada. El truco era tener una imagen para construir el artefacto, y se pasaba a una imagen de *runtime* mediante un uso inteligente de volúmenes. Con el nuevo **COPY --from** no necesitamos el paso intermedio de volúmenes, y se puede hacer todo con un solo paso.

Supongamos que queremos crear un programa en C, compilarlo y tener una imagen que se limite a ejecutarlo:

```bash
gerard@aldebaran:~/docker/test$ cat hello.c 
#include <stdio.h>
#include <stdlib.h>

int main() {
	printf("Hello world!\n");
	exit(0);
}
gerard@aldebaran:~/docker/test$ cat Dockerfile 
FROM alpine:3.5 AS builder
RUN apk add --no-cache gcc musl-dev
COPY hello.c /
RUN gcc -static -o /hello /hello.c && \
    strip /hello

FROM scratch
COPY --from=builder /hello /
CMD ["/hello"]
gerard@aldebaran:~/docker/test$ 
```

De esta forma, la primera imagen compila nuestro código fuente, y la segunda se limita a copiar el resultado, que al tratarse de un binario estático no necesita librerías adicionales, lo que nos permite partir de la imagen vacía *scratch*.

```bash
gerard@aldebaran:~/docker/test$ docker build -t hello .
Sending build context to Docker daemon  3.072kB
...  
Step 5/7 : FROM scratch
 ---> 
Step 6/7 : COPY --from=builder /hello /
 ---> b01e9f03b69d
Removing intermediate container c177f03a3dab
Step 7/7 : CMD /hello
 ---> Running in e19eb91bc975
 ---> fb52f56a9c03
Removing intermediate container e19eb91bc975
Successfully built fb52f56a9c03
Successfully tagged hello:latest
gerard@aldebaran:~/docker/test$ 
```

Y con esto podemos desechar la imagen de compilación que es muy grande, en favor a una imagen de *runtime* sin compiladores ni librerías, sin necesidad de preocuparnos de limpiar aquello que se haya instalado para compilar nuestro binario.

```bash
gerard@aldebaran:~/docker/test$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
hello               latest              fb52f56a9c03        2 minutes ago       10.1kB
<none>              <none>              fe6a6edae999        2 minutes ago       101MB
alpine              3.5                 4a415e366388        2 months ago        3.99MB
gerard@aldebaran:~/docker/test$ docker run -ti --rm hello
Hello world!
gerard@aldebaran:~/docker/test$ 
```

Es especialmente interesante ver que hacen falta 101mb para construir un binario que solo necesita 10kb para ejecutarse...
