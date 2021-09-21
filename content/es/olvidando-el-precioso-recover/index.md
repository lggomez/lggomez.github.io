{
    "title": "Olvidando el precioso recover",
    "description": "Qué ocurre cuando se olvida agregar un recover a una goroutina creada?",
    "summary": "Qué ocurre cuando se olvida agregar un recover a una goroutina creada?",
    "tags": ["Golang", "Post-mortem"],
    "date": "2021-01-02T02:50:55-03:00",
    "lastmod": "2021-01-02T02:50:55-03:00",
    "categories": ["Programming", "Backend", "Bug"],
    "type": "article",
    "weight": 0,
    "publishdate": null,
    "expirydate": null,
    "aliases": null,
    "slug": "olvidando-el-precioso-recover",
    "url": ""
}

___***El siguiente es un post-mortem publico que estaba pendiente de publicacion en el sistio de ingeniería de MercadoLibre, aunque la lentitud del proceso de edición y mi partida de la compañía previnieron dicha publicación.***___

## El evento 
Era un 28 de diciembre a las 10:10AM (GMT-3), el último viernes del año, cuando se alertó nuestro calculador de envíos en Shipping, el área de Mercado Libre que se encarga de calcular la promesa de entrega junto con las opciones de envío de un artículo o compra. 

Desde nuestro equipo en los monitores de NewRelic vimos que la alerta se debía a tiempos elevados que degradaron fuertemente la API. Era hora de dejar a un lado las sobras de pan dulce y ponerse a diagnosticar.

Revisando nuestras métricas, pudimos detectar que el problema se encontraba sólo en **SLA Coverage** de Brasil.  Seguimos el rastro en profundidad y llegamos a la fuente de la degradación masiva, una API call puntual: **service coverage**.

## El diagnóstico 
En ese momento, nuestros tableros nos mostraban que:
  * En service coverage, tanto el apdex como los tiempos de ejecución y métricas del runtime se degradaron al 100%, de manera casi simultánea, a través de todas las instancias del scope, en sólo un par de minutos. ⏰
  * La degradación está generando una cascada de retries que suman unos 300K RPM. Esto por sí solo no llega a generar afectación en circunstancias normales, pero es el golpe de gracia para las instancias.

En un intento de dar aire al scope, reiniciamos manualmente instancias existentes al mismo tiempo que boosteamos el scope. Adicionalmente, observamos que se comienzan a reemplazar instancias. Una vez identificado el origen del tráfico de prueba y parados dichos tests, las instancias vuelven lentamente, y se recupera el apdex.

Por un momento todo es confusión. Volvemos sobre nuestros pasos y por fin encontramos la fuente del downtime de nuestra API. Y resulta ser algo que habíamos descartado en un principio: los panics.
Cuando revisamos los logs de service coverage, habíamos notado que ocurrieron panics. Pero como se trataban de ocurrencias puntuales y de poca cantidad, en un principio lo ignoramos, creyendo que no era la causa de la degradación. 

## ¿Qué pasó? 
> “...los panics no se propagan entre goroutines, y por lo tanto deben manejarse independientemente.”
> 
Al llegar a aislar los stacktraces de los panics en las instancias afectadas, observamos un patrón:
  1. Solamente incluían el sitio y traza del panic sin ningún middleware ni intermediario de recover por medio, lo cual confirma que el mismo no fue manejado
  2. Pertenecían al código ejecutado desde una goroutina

Al buscar sobre issues de manejo de panics en goroutinas fue que comprendimos qué estaba pasando.

Como se puede ver en este [issue en el repo oficial de Go](https://github.com/golang/go/issues/20161), un detalle de diseño de Go que al principio muchos developers obviamos o desconocemos es que los panics no se propagan entre goroutines, y por lo tanto deben manejarse independientemente.

Cuando ocurre un panic dentro de un proceso de go, y éste no lo captura vía recover, el mismo finaliza de manera anómala (exit != 0).
Sin embargo, la api usa el router default provisto por nuestro toolkit interno de go, el cual incluye un middleware que se recupera ante un eventual panic de la api y devuelve una respuesta de Internal Server Error.

## Implicancias (a.k.a panics for dummies) 
Si ocurre un panic en una goroutine y ésta no la manja vía recover, la goroutine que la lanzó tampoco lo va a manejar, sin importar la cantidad de defer recover que posea a lo largo de sus stack frames.

Es un issue conocido, y por el momento el equipo de Go lo considera una decisión de diseño no sujeta a cambios para Go 1.x (crucemos los dedos para 2.x). 

El siguiente es un ejemplo de goroutines con panic propagado:

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	Go(test)
	time.Sleep(time.Second)
	fmt.Printf("HELLO")
}

func test() {
	panic("PANIC")
}

func Go(f func()) {
	defer func() {
		if r := recover(); r != nil {
			fmt.Printf("RECOVERED ERROR %v", r)
		}
	}()
	go f()
}
```

Comparémoslo ahora con un ejemplo de panic manejado. Notaran que al codigo le esta faltando ser mas componible:

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	Go(test)
	time.Sleep(time.Second)
	fmt.Printf("HELLO")
}

func test() {
	panic("PANIC")
}

func Go(f func()) {
	go func() {
		defer func() {
			if r := recover(); r != nil {
				fmt.Printf("RECOVERED ERROR %v", r)
			}
		}()
		f()
	}()
}
```

En resumen: los panics en goroutines necesitan un recover dentro. Si no se manejan, finaliza el proceso de la app y puede llegar a derribar un pool de instancias completo, y el middleware de recovery de nuestro toolkit interno no funciona en estos casos.

Esta sutil pero importante característica del runtime de Go nos llevó a reevaluar cómo manejamos la concurrencia en nuestras apis.

## Pensando soluciones 
Un par de días después, reuní al equipo con algunas alternativas para dejar a votacion sobre cual era la mejor solución para nuestro caso (revisar y aplicar guardias a un conjunto de N apicalls en varios repos)
Habían 3 candidatos:

### La obvia pero difícil 
No generar panics dentro de goroutines. Estuve a punto de patentarla pero Rob Pike llegó antes.

Esta opción es la ideal, pero en el mundo real, no siempre es posible. Muchas veces tenemos dependencias de terceros que incluyen panics como mecanismo de señalización de errores. Eliminarlos no está en nuestras manos.

### La idiomática pero fea 
Pues, deberíamos entonces agregar un recover a cada goroutine que lancemos entonces?

```go
defer func() {
	if r := recover(); r != nil {
		fmt.Printf("RECOVERED - %v\r\n", r)
	}
}()
```

Va a funcionar, aunque tiene la desventaja de que a pesar de que es la salida idiomática también es un code smell repetir esto por doquier y además este copypaste extra por goroutina es un potencial punto de falla extra.

Podemos mitigarlo al encerrar las funciones dentro de un handler como el siguiente, en tanto sean argumento de otra función que lance la goroutine o seamos nosotros mismos:

```go
func Wrap(f func()) func() {
	return func() {
			defer func() {
				if r := recover(); r != nil {
					fmt.Printf("RECOVERED - %v\r\n", r)
				}
			}()
		f()
	}
}
```

El cual usamos de la siguiente forma:

```go
func main() {
	go Wrap(test)()
	time.Sleep(time.Second)
	fmt.Println("HELLO")
}

func test() {
	panic("PANIC")
}
//…
```

De esta manera, centralizamos el manejo de panics en punto común y dentro de la misma línea en la que disparamos la goroutina.

Y, finalmente, en caso de querer devolver algo desde dicha goroutine, nos comunicamos a través de channels:

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	signal := make(chan bool)
	go WrapWithSignal(test, signal)()
	time.Sleep(time.Second)
	fmt.Println("HELLO")
	select {
	case sig := <-signal:
		if !sig {
			fmt.Println("Could not finish test execution")
		} else {
			fmt.Println("Finished test execution")
		}
	}
}

func test(signal chan bool) {
	panic("PANIC")
	signal <- true
}

func WrapWithSignal(f func(chan bool), signal chan bool) func() {
	return func() {
		defer func() {
			if r := recover(); r != nil {
				fmt.Printf("RECOVERED - %v\r\n", r)
				signal <- false
			}
		}()
		f(signal)
	}
}
```

Vale tener en cuenta que esto es un patrón idiomático pero que puede acumular rápidamente duplicación de código con complejidad adicional, especialmente a la hora de testear. A usarlo con criterio.

### Bonus track: La solución de infraestructura 
Más que solución podemos considerarlo un workaround de infraestructura que aumentará la resiliencia para estos casos (considerado el tiempo reducido de inicialización y warmup en comparación a otras plataformas como Java).

La misma consiste en monitorear el proceso de la aplicación, y ante una finalización anómala, iniciarla nuevamente. No es una solución per se, pero da aire a la instancia para que vuelva a atender requests y mitigar una degradación por tiempos de reemplazo de la misma.

### Bonus track II: Detectando unrecovered panics 
Este era el dump de middlewares de la aplicación (notar que `RecoveryWithWriter` es el middleware de recovery):
 
```shell
vendor/[ommited]/gingonic/mlhandlers.Datadog.func1
vendor/github.com/newrelic/go-agent/_integrations/nrgin/v1.Middleware.func1
vendor/[ommited]/gingonic/mlhandlers.CommonAPiFilter.func1
vendor/[ommited]/gingonic/mlhandlers.RecoveryWithWriter.func1
context.AddContext [ommited]/src/api/context.AccessLog
controllers.(*RouteCoverageController).ValidateGet-fm
controllers.(*RouteCoverageController).Get-fm
```

Y, el stacktrace del panic sin manejar en toda su gloria:
`
```shell
goroutine 141 [running]:
[ommited]/src/api/services.(*CoverageService).GetCoverageServices(0xc4205ec1e0, 0x10e4e20, 0xc420073a70, 0xc420e8a41b, 0x3, 0xc420e8a41b, 0x3, 0x0, 0x0, 0x0, ...)
/services/coverage_service.go:35 +0x39
[ommited]/src/api/services.(*CoverageLocationService).GetRouteCoverage.func1(0xc400000008, 0xca6f90)
/services/coverage_locations.go:39 +0xe3
[ommited]/src/api/vendor/golang.org/x/sync/errgroup.(*Group).Go.func1(0xc42041e2c0, 0xc420426180)
/vendor/golang.org/x/sync/errgroup/errgroup.go:57 +0x57
created by [ommited]/src/api/vendor/golang.org/x/sync/errgroup.(*Group).Go
/vendor/golang.org/x/sync/errgroup/errgroup.go:54 +0x66 ```
```

El comportamiendo se confirma por 2 cosas:
  1. El recover middleware del toolkit no llega a ejecutarse
  2. El panic está ocurriendo dentro de una goroutine disparada desde una una instancia de [errgroup.Group](https://godoc.org/golang.org/x/sync/errgroup)
