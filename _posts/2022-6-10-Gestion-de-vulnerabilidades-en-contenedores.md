---
layout: post
title: Gestion de vulnerabilidades en contenedores
---

Hay mucha información sobre como implementar un escáner de imágenes de contenedores, pero no mucha sobre que hacer cuando estas comienzan a ser reportadas.
Este articulo pretende ofrecer un enfoque de gobierno que permita:
* Disminuir la cantidad de hallazgos en el futuro
* Implementar un sistema de gestion de vulnerabilidades
* Asignar propiedad a un numero de equipos acotado para optimizar la comunicación
* Definir una serie de indicadores que permitan la evolución del proceso.

## Introduccion

Durante el 1er semestre de este año, Gartner publico un documento donde menciona que el 70% de las compañías va estar utilizando alguna tecnología de aplicaciones en contenedores para 2023.
Adicionalmente a esto, durante el año pasado la Cloud-Native Computing Foundation (CNCF) realizo una encuesta llegando a mas de 3800 desarrolladores para conocer que tecnologías utilizaban para el desarrollo de sus backend. Los resultados llegan a la conclusión de que Kubernetes y su consecuente aplicación de aplicaciones en contenedores ya cruzo el umbral de adopción transformándose en una tecnología globalmente conveniente para disponibilidad servicios internos o externos.

## Contexto y desafios

La tecnologia de contenedores permite abstraer el sistema operativo del código, lo que conlleva a desplegar servicios de manera mas rápida y en lenguaje que el desarrollador prefiera. 
Distintos usos de lenguaje generan ademas, distintos usos de librerías para agregar funcionalidad.
En compañías de medio a gran tamaño, la cantidad de microservicios desplegados puede ascender a cientos o miles, generando la aparición de una misma vulnerabilidad en multiples elementos debido a la reutilizaron no controlada de dependencias transitivas.

En una etapa inicial es muy común que el desarrollador seleccione una imagen base que satisfaga el software que precisa para ejecutar su aplicación, ya sea seleccionado un sistema operativo (ubuntu/debian/centos) e instalando sus paquetes necesarios o directamente, utilizar una imagen con el lenguaje importado (ruby / openjdk / golang).

![Image](https://raw.githubusercontent.com/jnahuelperez/jnahuelperez.github.io/master/_posts/images/1/1_docker_simple_layers.png)

En el mejor de los casos, una imagen puede tener elementos que son ajenos a la aplicación pero que deben incluir como pueden ser software base o configuraciones estándar de hardening para agregar un componente de seguridad en las mismas y estandarizar los elementos en ella. 

![Image](https://raw.githubusercontent.com/jnahuelperez/jnahuelperez.github.io/master/_posts/images/1/1_docker_3_layers.png)

Generalmente se deja toda la generación de imagenes en manos de los desarrolladores o equipo DevOps ya que se asume, son ellos los que saben que se debe incluir para que su sistema funcione. 

Cuando los desarrolladores reciben el reporte de vulnerabilidades no tienen idea por donde empezar. Se encuentran con paquetes que ellos no incluyen en su código ni en la configuración de su imagen, pero que vienen heredados de aquellas superiores (OpenJKD > Debian > Scratch > tar.gz) las cuales a su vez, instalan paquetes adicionales generando dependencias transitivas. 

Esto genera nuestro primer problema: Como arreglamos vulnerabilidades fuera de nuestro alcance?

En un ambiente podemos tener cientos o miles de contenedores ejecutando servicios, las cuales pueden utilizar librerías en común, pero que fueron creadas desde imagen distintas. 
Esto da lugar a nuestro segundo inconveniente: cómo podemos disminuir la cantidad de veces que se debe parchear una misma vulnerabilidad?

Otro inconveniente recurrent es la falta de estandarización: como podemos asegurar que nuestros contenedores tienen los elementos necesarios básicos para cumplir con nuestras políticas de seguridad?

Para abordar estas preguntas lo que proponemos es una modelo de gobierno de generación y consumo de imagenes de para contenedores gestionado por al menos 3 capas que aprovecha el uso de la generación multi stage para heredar funcionalidad.

**La imagen baseline**
Esta imagen va contener el sistema operativo BASE sobre el cual se crearan las adicionales. En esta etapa seleccionamos la que satisface nuestro apetito de riesgo. 

Tener cero vulnerabilidad es un camino muy difícil de recorrer al cual se podría llegar utilizando imagenes scratch, el costo de ello es asumir una inmensa gestion de mantenimiento de paquetes que podríamos delegar al implementar una imagen con una cantidad de vulnerabilidades aceptables y tomarla como baseline:
```
➜  cat baseline/Dockerfile 
FROM debian:11.3-slim as baseline
RUN apt-get update && apt-get install -y <your_critical_base_packages>
                                       
➜   docker build -t micompania/baseline baseline/
```
En este caso elegimos `debian:11.3-slim` la cual tiene 3 vulnerabilidades criticas y 4 altas. Esto va a definir nuestro báseline de vulnerabilidades para esta primer capa.

![Image](https://raw.githubusercontent.com/jnahuelperez/jnahuelperez.github.io/master/_posts/images/1/1layer_results.png)

**La imagen middleware**
Esta segunda capa nos va a permitir definir una interfaz que va a contener los paquetes clave que todas las aplicaciones deben tener: una librería para la generación de logs, conexión a bases de datos, cifrado, etc. Estas librerías importadas podrían surgir de los estándares definidos por una puesta en común de los equipos de trabajo.
```
➜  cat middleware/Dockerfile 
FROM micompania/baseline

RUN apt-get update && apt-get install -y \
    apt-transport-https \
    curl \
    gnupg-agent \
    sqlite3 \
    liblog4j2-java    
                                                                                     
➜ docker build -t micompania/middleware middleware/
```
Esta segundo stage acumulara las vulnerabilidades de el primero mas las que están incluidas en esta agregación de paquetes, adicionando 2 vulnerabilidades criticas a la cuenta.

![Image](https://raw.githubusercontent.com/jnahuelperez/jnahuelperez.github.io/master/_posts/images/1/2layer_results.png)

Como agregado, podríamos tener una interfaz middleware dependiendo del lenguaje, una para golfing, una para java, una para python, etc...

La imagen de aplicación
Finalmente en esta 3er capa tenemos la aplicación que los desarrolladores desean publicar en donde adicionalmente, pueden instalar los paquetes necesarios y ausentes en las capas anteriores.

```
 ➜ cat miapp/Dockerfile               
FROM micompania/middleware

EXPOSE 8080

WORKDIR /app
ADD defroster.py .

CMD ["gunicorn", "-w", "4", "app:app"] 
                                                           
➜ docker build -t micompania/miapp miapp/
```

Este modelo de capas y consumo permite generar una suma de vulnerabilidades exponencialmente menor a la que tendríamos por cada aplicaciones desplegada en contenedores que utilize la imagen que mejor le parezca sin tener control sobre los paquetes que las distintas capas puedan agregar.

![Image](https://raw.githubusercontent.com/jnahuelperez/jnahuelperez.github.io/master/_posts/images/1/3layer_results.png)

Adicionalmente permite direccionar la responsabilidad de su resolución de manera mucho mas clara al visualizar que las capas donde están presentes, se corresponden con los equipos que las desarrollan.
En este ejemplo utilizamos la herramienta [dive](https://github.com/wagoodman/dive) para explorar las capas de nuestra imagen:

![Image](https://raw.githubusercontent.com/jnahuelperez/jnahuelperez.github.io/master/_posts/images/1/dive.png)

Finalmente, aplicando este modelo, la aparición de una vulnerabilidad como Log4Shell se parchea una única vez (en este ejemplo en el middleware) generando un efecto en cadena para luego, enviando un mensaje de actualización a sus consumidores (desarrolladores que la importan) ejecutar un nuevo build de sus imágenes, mitigando el riesgo en su aplicación.

![Image](https://raw.githubusercontent.com/jnahuelperez/jnahuelperez.github.io/master/_posts/images/1/patching.png)

#### Fuentes
* https://www.zdnet.com/article/cncf-reports-record-kubernetes-and-container-adoption/
* https://www.cncf.io/wp-content/uploads/2022/02/CNCF-AR_FINAL-edits-15.2.21.pdf
* https://geekflare.com/container-security-scanners/
