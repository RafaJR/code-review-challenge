# Revisión de código de clasificador de anuncios

Mis consideraciones personales y propuestas de mejora para la versión actual de la API Restful para puntuación y clasificación de anuncios del portal "idealista.com".

## Índice de contenido
* [Arquitectura](#arquitectura)
* [Controladores](#controladores)
* [Constantes y métodos estáticos](#constantes)

## Arquitectura
Mis consideraciones sobre el modo de estructuración general del proyecto.
### Arquitectura hexagonal
En general, el código no parece estar organizado en una estructura de capas propia de una arquitectura hexagonal. 
Este tipo de arquitectura permitiría, mediante la separación del código en unidades organizativas de responsabilidad única, 
un más fácil entendimiento del código, facilidades en su mantenimiento, más pronta localización de errores y la capacidad de
introducir correcciones o mejoras en ciertos aspectos de la aplicación sin que otros se vean afectados.

Además, conviene considerar la adición de algunas capas para funcionalidades faltantes sobre las que me explayaré más adelante.

Un ejemplo válido de modelo de arquitectura hexagonal podría ser este:

	\_[com.idealista]
		|   Clase principal de arranque y "Listeners" de inicio si procede.
		\_[controller]
		|       Clases para definir los 'endpoints'.
		\_[service]
		|       Interfaces y métodos de la capa de servicios que se inyentará en las clases de 'controller'.
		\_[dao]
        |   |   Interfaces y métodos de la capa de datos que contendrán los métodos encargados de operar sobre la base de datos.
		|	|   Se inyecta en la capa de servicios.
		|	\_[entities]
		|           Clases de mapeo de tablas de la base de datos o entidades.
		\_[model]
		|	|   Clases tipo 'POJO' conocidas como DTOs (Data Transfer Object), empleadas para la transferencia de datos entre los 'endpoints',
		|	|   usualmente siendo transformadas a formato 'JSON'.
		|	\_[input]
		|	|	    DTOs empleados como datos de entrada en el 'endpoint' a través del 'body' de la petición o por parámetro de la 'URL'.
		|	\_[output]
		|		    DTOs empleados como datos de salida de los 'endpoint'.
		\_[validation]
            |   Interfaces y métodos de validación desarrollados conforme al patrón 'javax.validation', para ser aplicados en los
            |   'Controllers' a nivel de parámetro.
		\_[constants]
		|	    Clases continentes de primitivas e instancias estáticas que serán manendrán el mismo valor durante toda la ejecución
		|	    en diferentes lugares de uso, por lo que interesa agruparlas en una clase para evitar duplicidades en el código.
		|	    También puede contener métodos estáticos de uso recurrente, por ejemplo métodos de cálculo que sean necesarios en más de un servicio.
		\_[exceptions]
		|	    Definicion de excepciones customizadas para nuestra lógica de negocio.
		|	    Por ejemplo, podrían diseñarse y lanzarse excepciones para casos de anuncios con puntuaciones por debajo de 0 o por encima de 100
		|	    en el proceso de guaradado de los mismos.
		\_[mappers]
			    Definición de métodos de conversión de DTOs a entidades y viceversa siguiendo el patrón 'org.mapstruct.Mapper'.
### Base de datos embebida en memoria
Con el fin de simular la interacción con una base de datos real, en lugar de cargar unas simples listas en el constructor de la clase del 'dao'
('InMemoryPersistence.java'), se podría configurar una base de datos embebida en memoria de tipo 'H2' y cargarla con datos reales después del evento de
desliegue de la aplicación.
Esta configuración, podría hacerse fácilmente en el archivo 'application.yml' de la siguiente manera:

	server:
	  port: 8080
	spring:
	  datasource:
	    url: jdbc:h2:adclassifierdb
	    driverClassName: org.h2.Driver
	    username: admin
	    password:    
	  h2:
	    console:
	      enabled: true
	  jpa:
	    database-platform: org.hibernate.dialect.H2Dialect
Como se puede ver, esta configuración creará una base de datos embebida en memoria (solo disponible en tiempo de ejecución del servicio, por lo que los datos
se perderán al apagar dicho servicio) denominada 'adclassifierdb', accesible a través del mismo puerto del servicio (8080) y con consola habilitada (http://localhost:8081/h2-console).
Esta base de datos se puede cargar justo después del despliegue añadiendo un "Listener" con un método que se ocupará de dicha carga tras escuchar el evento de despliegue.
Este "Listener" puede estar ubicado en el mismo paquete de la clase principal de carga del servicio ('com.idealista'), podría llamarse 'AdClassifierLoader.java' y su código
aproximado podría ser tal que así:

    @Component
    public class AdClassifierLoader implements ApplicationListener<ApplicationReadyEvent> {
        @Override
        public void onApplicationEvent(ApplicationReadyEvent event) {
    
            loadDB();		
    
        }
    
    }
Cómo se puede ver, se trata de llamar a un método después del proceso de arranque llamado 'loadDB' que cargaría los datos llamando a la capa
de servicios. Tendría un código similar al del constructor de 'AdClassifierLoader', pero cargaría las listas en la base de datos en memoria.
Para ralizar esta función, las clases de entidad deberían estar debidamente mapeadas, tema sobre el cual me explayaré en el apartado sobre
las entidaddes.
## Controladores
Ideas sobre la implementación de los 'endpoints' en el controlador.
### Manejo de excepciones y respuestas fallidas
En el controlador 'AdsController' únicamente se están considerando respuestas 'http' de estado '200', es decir, válidas.
Sería conveniente llevar un control de las excepciones que puedan producirse en el servicio o en el acceso a datos,
capturarlas en el controlador y devolver una respuesta de estado '500' u otro cuando estas se produzcan.
También sería útil diseñar excepciones customizadas para casos específicos de la lógica de negocio que se está aplicando
y manejarlas en el controlador de igual modo. Por ejemplo, podría crearse una excepción que se lanzaría en el servicio
cuando el cálculo de la puntuación de un anuncio sea menor de 0 o mayor de 100.
El código en el método del controlador, por ejemplo para el 'endpoint' cuyo contexto es "/ads/public", podría ser semejante
a este:
	
	@GetMapping("/ads/public")
    public ResponseEntity<?> publicListing() {
    	try {
    		return ResponseEntity.ok(adsService.findPublicAds());
    	}catch(Exception e) {
    		return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("An error happened when trying to find the public ads.");
    	}
    }
## Constantes
Mis consideraciones sobre la forma de declarar las constantes en el proyecto.
### Nombres de constantes poco representativas
Quizá sea más aclaratorio que los nombres de las constantes hicieran referencia a su significado y no al valor de su contenido.
Por ejemplo, se observan constantes empleadas en el cálculo de las puntuaciones tales como:

    public static final int FIVE = 5;
    public static final int TEN = 10;
    public static final int TWENTY = 20;
    public static final int THIRTY = 30;
    public static final int FORTY = 40;
    public static final int FORTY_NINE = 49;
    public static final int FIFTY = 50;
    public static final int ONE_HUNDRED = 100;
Aparte de que estos nombres no aportan ninguna información sobre la función de estas constantes en la aplicación,
si en algún momento de la evolución del proyecto se decidiese cambiar alguno de estos valores habría que cambiar también su nombre y,
por tanto, habría que cambiar todas las referencias a dicha constante en todo el código.
Además, se están usando los mismos valores para conceptos distintos y no hay garantía de que esta condición siga siendo viable en el futuro;
por ejemplo, se está usando 'TWENTY = 20' para fijar el sumando de puntuación para una foto con calidad HD y el mínimo de palabras que debe
tener una descripción para puntuar "+10".
En cambio, si se nombran las contantes en base a su significado, por ejemplo de esta manera:

    public static final int DESCRIPTION_IS_PRESENT_SCORE = 5;
    public static final int NO_PICS_SCORE = -10;
    public static final int HD_PIC_SCORE = 20;
    public static final int SD_PIC_SCORE = 10;
    public static final int LARGE_FLOOR_DESCRIPTION_SCORE = 30;
    public static final int FULL_AD_SCORE = 40;
    public static final int MEDIUM_DESCRIPTION_MIN_LIMIT = 20;
    public static final int MEDIUM_DESCRIPTION_MAX_LIMIT = 49;
    public static final int LARGE_HOUSE_DESCRIPTION_SCORE = 50;
    public static final int MAX_SCORE_LIMIT = 100;
Los nombres resultan mucho más indicativos de lo que representan, se pueden cambiar sus valores sin tener que renombrar las constantes,
y no se generan conflictos por valores que representan más de un concepto. Apreciese también que, al fijar los valores de puntuación que
deben restarse como negativos (NO_PICS_SCORE = -10), se facilita la función de calcular dicha puntuación en el servicio, permitiéndo resolver el
cálculo con una simple suma, sin tener que prestar atención a qué valores han de restarse o sumarse.
## Generalidades
Mis consideraciones sobre aspectos más transversales observables en el código.
### Trazabilidad
No se está aplicando niguna trazabilidad, es decir, 'logs'.


