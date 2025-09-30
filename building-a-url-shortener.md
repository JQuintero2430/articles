# Building a URL Shortener

este diseño esta horientado a brindar una solución dentro del contexto de una entrevista de trabajo, razon por la cual se dejarán de lado algunas consideraciones que puedan afectar el diseño en un entorno de produccion.

Con esto en mente el planteamiento es el siguiente:
"Se te solicita diseñas un acortador de url's que devuelva la url fake-company.io/XXXXXX de modo que podamos brindar un servicio de url cortas a nuestros usuarios y que los mismos puedan brindar dicha url corta si desean compartir la direccion de su pagina web"

Teniendo en cuenta el contexto lo primero que vale la pena preguntar es cuantos caracteres devolverá este acortador para añadirlo al path y cual será la base de hashing para devolver los caracteres, tambien es prudente saber si estas URL acortadas tendran un TTL o si se mantendrán para siempre en nuestros registros y por ultimo es valido preguntar si debemos hacer validaciones previas a devolver la url acortada.

Dado que las validaciones son un diseño en si mismo lo dejaremos fuera de nuestro scope, para este caso usaremos 6 caracteres y usaremos base 36 (a-z,0-9) a modo que simplifiquemos el problema al maximo y nos centremos en el razonamiento detrás del diseño del acortador, imaginando que no debemos eliminar nuestros path acortados nunca.

Con esto claro iniciaremos entendiendo cuantas posibilidades hay dentro de nuestro path acortado que identificará nuestra direccion original.

Para esto vamos a hacer el siguiente calculo, cada uno de los caracteres tiene 36 posibles resultados que a su vez estan anidados a los posibles 36 resultados que poseen sus otros compañeros dentro de nuestro path.

Por lo tanto en este caso debemos encontrar cual es el valor de 36 elevado a la 6, que es el numero de  caracteres a devolver, eso nos daría como resultado 2.176.782.336 posibles resultados. 

Ahora sabemos que cuando almacenemos todas nuestra combinaciones posibles vamos a tener una base de datos con poco mas de dos mil millones de registros. 

Lo que nos lleva de una vez a pensar en nuestra base de datos, para nuestro caso estaría bien una base de datos sql con una columna para el path generado, otra columna con la direccion original y quizas una columna con la fecha de creación, tambien es factible pensar que muchas de las url seran consultadas mas de una vez por lo que sería bueno usar el patron CQRS para poder llevar nuestros registros a una base de datos NoSQL desde donde hacer las consultas y obtener las url originales a donde debemos redirigir a nuestros usuarios.

otra estrategia valida es tener una base de datos clave valor que guarde en disco como RocksDB o utilizar una estrategia intermedia en donde almacenemos nuestros registros unicamente en una base de datos SQL pero al momento de realizar nuestra primera consulta del dia o de la semana almacenemos dicho valor en una base de datos en memoria como Redis, brindando un TTL acorde a nuestro caso de uso.

Con esto resuelto esta el tema de como generar este valor aleatorio que será nuestro path, a pesar de que el primer approach puede ser utilizar una libreria que genere el valor random en base a nuestra direccion origin, el cual desde ahora nombraremos source, este enfoque se queda corto ya que cada path resultante (target), debe ser unico y con una estrategia que use un randomizer encontraremos colisiones muy pronto en nuestro sistema, lo que nos llevaría a hacer un sistema de reintentos que a mediano plazo puede llevarnos a loops casi infinitos, donde nuestro sistema intenta generar un target, verifica que ya existe, vuelve a generar un target alternativo y dicho target ya fue generado previamente, así hasta el infinito.

Con esto en mente es obvio que no podemos simplemente usar un randomizer, lo que nos obliga a pensar en que lo ideal puede ser tener predefinidos todas las posibilidades e irlas asignando alas nuevas URL que desena ser acortadas, pero es ahí donde surge otro inconveniente, como determinaremos en cual posibilidad vamos, ya que si los guardamos en orden natural(000000, 000001, etc) hacemos vulnerable nuestro sistema.

Para abordar esta nueva dificultad una muy buena opcion es tener todos nuestros registros en orden dentro de una estructura de datos lineal, luego desordenarlas a traves de un metodo que actue como shuffle y por ultimo persistirlas dentro de nuestra base de datos. 

Lo que tiene la ventaja de que podríamos hacer dicha labor por lotes y en un unico job, dejando a nuestro sistema con capacidad de computo libre para realzar otras acciones en el momento en que nuestro acortador salga a produccion. 

Con este diseño resolvemos una de las dudas que quedo abierta mas arriba, ya que lo ideal sera usar una base de datos sql que permita hacer lock de los registros y tendremos un mecanismo que funcionará de la siguiente manera:

Preparamos nuestro sistema con todas las posibles conbinaciones en orden aleatorio persistidos en una base de datos sql que permita hacer lock de los registros, en la cual tendremos un campo que servirá de flag para determinar si ese "hash" fue asisgnado previamente y el cual nuestro motor de base de datos usara para pder mantener la secuencia de asignaciones, tambien poseerá un campo donde guardaremos la direccion original de la url a acortar y un marcador de fecha para cuando se asigne el target.
cuando el susuario introduzca la direccion que desee aacortar, haremos las validaciones que consideremos pertinentes y que veremos en una proxima entrada, e iniciaremos el proceso de bloque del target "mas nuevo" sin utilizar.
estando bloqueado para que ningun otro usuario obtenga el mismo target, haremos un update donde guardaremos el source, guardaremos el momento en un timestamp y cambiaremos el flag, para luego liberar el record y devolver el target a nuestro usuario.
Cuando el usuario ingrese a la url fake-company.io/{target} haremos una lectura de nuestra base de datos usando el target como value en nuestro where, devolviendo y redirigiendo al usuario al source o en otras palabras a la direccion original.

Espero que con esta guia puedan afrontar desde una perspectiva simple, una de las preguntas mas frecuentes en entrevistas tecnicas y puedan abrir la mente a las posibilidades dentro del diseño de sistemas.