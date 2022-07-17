# Revisión de código de clasificador de anuncios

Mis consideraciones personales y propuestas de mejora para la versión actual de la API Restful para puntuación y clasificación de anuncios
del portal "idealista.com".

## Índice de contenido

* [Arquitectura](#arquitectura)
* [Controladores](#controladores)
* [Constantes](#constantes)
* [Capa de servicio](#capa-de-servicio)
* [Mappers](#mappers)
* [Model](#model)
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
		|	|   Clases tipo 'POJO' conocidas como DTOs (Data Transfer Object), empleadas para la transferencia de datos entre los 
            |   'endpoints', usualmente siendo transformadas a formato 'JSON'.
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
		|	    También puede contener métodos estáticos de uso recurrente, por ejemplo métodos de cálculo que sean necesarios en más de un 
        |       servicio.
		\_[exceptions]
		|	    Definicion de excepciones customizadas para nuestra lógica de negocio.
		|	    Por ejemplo, podrían diseñarse y lanzarse excepciones para casos de anuncios con puntuaciones por debajo de 0 o por encima 
        |       de 100. en el proceso de guaradado de los mismos.
		\_[mappers]
			    Definición de métodos de conversión de DTOs a entidades y viceversa siguiendo el patrón 'org.mapstruct.Mapper'.

El resto de los comentarios se articularán en base a esta arquitectura, indicando de componentes deben de formar parte de cada paquete
cuando estos realmente no se ajustan a este modelo propuesto.
### Base de datos embebida en memoria
Con el fin de simular la interacción con una base de datos real, en lugar de cargar unas simples listas en el constructor de la clase del 
'dao' ('InMemoryPersistence.java'), se podría configurar una base de datos embebida en memoria de tipo 'H2' y cargarla con datos reales 
después del evento de desliegue de la aplicación.
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
Como se puede ver, esta configuración creará una base de datos embebida en memoria (solo disponible en tiempo de ejecución del servicio, por
lo que los datos se perderán al apagar dicho servicio) denominada 'adclassifierdb', accesible a través del mismo puerto del servicio (8080) 
y con consola habilitada (http://localhost:8081/h2-console).
Esta base de datos se puede cargar justo después del despliegue añadiendo un "Listener" con un método que se ocupará de dicha carga tras 
escuchar el evento de despliegue (es decir, que se ejecuta al cargar la aplicacion). Este "Listener" puede estar ubicado en el mismo paquete
de la clase principal de carga del servicio ('com.idealista'), podría llamarse 'AdClassifierLoader.java' y su código aproximado podría ser:

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
En el controlador 'AdsController' únicamente se están considerando respuestas 'http' de estado '200', es decir, válidas. Sería conveniente
llevar un control de las excepciones que puedan producirse en el servicio o en el acceso a datos, capturarlas en el controlador y devolver
una respuesta de estado '500' u otro cuando éstas se produzcan. También sería útil diseñar excepciones customizadas para casos específicos
de la lógica de negocio que se está aplicando y manejarlas en el controlador de igual modo. Por ejemplo, podría crearse una excepción que se
lanzaría en el servicio cuando el cálculo de la puntuación de un anuncio sea menor de 0 o mayor de 100. El código en el método del
controlador, por ejemplo para el 'endpoint' cuyo contexto es "/ads/public", podría ser semejante a este:

	@GetMapping("/ads/public")
    public ResponseEntity<?> publicListing() {
    	try {
    		return ResponseEntity.ok(adsService.findPublicAds());
    	}catch(Exception e) {
    		return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("An error happened when trying to find the public ads.");
    	}
    }
### Sobre el 'endpoint' para calcular las puntuaciones
Las puntuaciones se establecen en la entidad llamando a un 'endpoint' que actualiza sus valores. Esto plantea un problema de consistencia de
datos pues, si estos anuncios actualizados de cualquier forma que afecte a su puntuación, dicha puntuación permancerá inconsistente con los
otros datos que la afectan. Es decir, la puntuación no se corresponderá con la descripción, las fotos y demás. Naturalmente, si se consultan
los anuncios antes de llamar a este 'endpoint', la puntuación no será la correcta, por lo que podremos tener listas de anuncios relevantes
con anuncios que realmente son irrelevantes y viceversa.

Esto podría corregirse haciendo que la puntuación sea un valor transitorio y auto-calculado en los procesos de consulta, lo cual permitiría
prescindir del 'endpoint' de calculo y de los servicios a los que llama. La manera en que se podría implementar una solución como esta, 
con sus ventajas e inconvenientes, se explicará con más detalle en el apartado sobre la capa 'DAO'.
## Constantes
Mis consideraciones sobre la forma de declarar las constantes en el proyecto.
### Ajuste en el modelo de arquitectura propuesto
La única clase de constantes que hay debería ubicarse en este paquete del modelo hexagola propuesto:

        com.idealista.constants
            Constants
### Nombres de constantes poco representativas
Quizá sea más aclaratorio que los nombres de las constantes hicieran referencia a su significado y no al valor de su contenido. Por ejemplo,
se observan constantes empleadas en el cálculo de las puntuaciones tales como:

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
Las ventajas son que nombres resultan mucho más indicativos de lo que representan, se pueden cambiar sus valores sin tener que renombrar las
constantes y no se generan conflictos por valores que representan más de un concepto. Apreciese también que, al fijar los valores de
puntuación que deben restarse como negativos (NO_PICS_SCORE = -10), se facilita la función de calcular dicha puntuación en el servicio,
permitiéndo resolver elcálculo con una simple suma, sin tener que prestar atención a qué valores han de restarse o sumarse.

## Capa de servicio
Mis consideración sobre la implementación de la lógica de negocio en la capa de servicio.
### Ajuste en el modelo de arquitectura propuesto
La interfaz y la implementación del servicio podrían ubicarse dentro del modelo de arquitectura hexagonal propuesto así:

    com.idealista.service
        AdsService
        AdsServiceImpl
### Ordenación de listas en la capa 'dao'.
En método 'findPublicAds' la lista resultante de la consulta a la fuente de datos está siendo ordenada inmediatamene después de la consulta,
sin comprobar si realmente contiene datos. Sería más eficiente realizar la ordenación de la lista en la capa 'dao', en el mismo método de la
consulta y resolverlo todo con una única llamada al 'dao'. La mejor forma sería mediante la implementación de una interfaz de JPA, sobre lo
cual me explayaré en el apartado correspondiente.
### Tratamiento de estructuras de datos con 'streams' y expresiones 'lambda'
En los métodos 'findPublicAds' y 'findQualityAds' del servicio  'AdsserviceImppl' se realiza la conversión de entidades a DTOs de salida
mediante un recorrido iterativo de la propia lista. Se podría compactar el código y mejorar la eficiencia de estos métodos aplicando una
función anónima expresada en modo 'lambda' sobre la lista tratándola como un tipo 'stream'. Por ejemplo, en el método 'findPublicAds' se
podría hacer así:

    @Override
    public List<PublicAd> findPublicAds() {
    List<Ad> ads = adRepository.findRelevantAds();
    ads.sort(Comparator.comparing(Ad::getScore));

        return ads.stream().map(ad -> {
          PublicAd publicAd = new PublicAd();
          publicAd.setDescription(ad.getDescription());
          publicAd.setGardenSize(ad.getGardenSize());
          publicAd.setHouseSize(ad.getHouseSize());
          publicAd.setId(ad.getId());
          publicAd.setPictureUrls(ad.getPictures().stream().map(Picture::getUrl).collect(Collectors.toList()));
          publicAd.setTypology(ad.getTypology().name());
          return publicAd;
        }).collect(Collectors.toList());
    }

Como se puede observar, se está emplendo la función 'map' que permite la conversión de tipo y se está realizando dicho cambio en una
expresión 'lambda'. El 'stream' resultante se reconvierte a tipo 'List' mediante la aplicación de un 'Collector'. La implementación de esta
misma idea en el método 'findQualityAds' sería muy similar. Nótese también que en el ejemplo de código expuesto, se está creando el objeto
de destino de la conversión 'PublicAd' dentro de la misma expresión 'lambda', donde mismo se fijan sus valores. Esto no es muy elegante y
podría corregirse aplicando aquí el patrón de diseño
'Mapper', lo cual se explica a continuación.
### Aplicación de mapeadores para la conversión de tipos en la capa de servicio
La conversión de listas de entidades resultantes de las consultas a las fuentes de datos a DTOs de resultados para la 'response' se está
realizando en los propios métodos de servicio, instanciando ahí mismo los objetos DTOs dentro de bucles iterativos (mala práctica) y fijando
los valores con los metodos 'set' del mismo. Se hace así en los métodos 'findQualityAds' y 'findPublicAds' de la implementación de
servicio 'AdsserviceImppl'. Se evitarían malas prácticas y se haría un código más limpio y mantenible implementando el patrón 'Mapper'. En
primer lugar, se necesita crear un 'Mapper' o servicio de mapeo de clases, lo cual se explica más adelante en su [apartado](#mappers).
Este 'Mapper' podría llamarse 'IPublicAdMapper' y tendría que inyectarse en nuestro servicio para después aplicarlo en el método
'findPublicAds'de esta manera:

    @Autowired
    private IPublicAdMapper publicAdMapper;
    
    @Override
    public List<PublicAd> findPublicAds() {
    List<Ad> ads = adRepository.findRelevantAds();
    ads.sort(Comparator.comparing(Ad::getScore));
    
        return ads.stream().map(ad -> publicAdMapper.map(ad)).collect(Collectors.toList());
    }
# Mappers
La implementción del patrón 'Mapper', complementario al patrón 'Data Transfer Object', permitirá un más fácil conversión de tipos de entre 
entidades y DTOs en una capa propia para esta fin, impidiendo la repetición de código y manteniendo la capa de servicio más limpia.
### Ajuste en el modelo de arquitectura propuesto
El lugar donde crear estos 'Mapper' en la arquitectura propuesta sería aquí:

    \_[com.idealista]
		\_[mappers]
### Ejemplo de implementación de 'Mapper'
Este es un ejemplo de 'Mapper' para la conversión del tipo 'Ad' (entidad) al tipo 'PublicAd' (DTO):

    @Mapper
    public interface IPublicAdMapper {
    
        PublicAd map(PublicAd publicAd);
    
    }
Este 'Mapper' e podría aplicar en el método 'findPublicAds' del servicio 'AdsserviceImppl' como se indica en el apartado anterior sobre la 
aplicación de mapeadores. Dado que los tipos a convertir contienen los mismos campos y se denominan igual, no se hace necesario implementar
la interfaz y sobreescribir el método; el 'framework' se ocupa de crear la implementación.
El ejemplo de 'Mapper' aplicable al método 'findQualityAds'
## Model
Mis consideraciones sobre los DTOs de entrada y salida que ahora se encuentran en el mismo paquete que el controlador.
### Ajuste en el modelo de arquitectura propuesto
El lugar donde ubicar los DTOs de entrada y salida en el modelo de arquitectura propuesto sería aquí:

    \_[com.idealista]
		\_[model]
			\_[input]
			|   \_ -
			\_[output]
                    \_PublicAd
                    \_QualityAd
### Implementación del patron 'Builder' y las anotaciones de 'lombok'
Sería muy útil emplear en estas clases las anotaciones de ahorro de código de la librería 'lombok'. Se permitiria ahorrar el código mónotono
y repetitivo de los métodos 'get' y 'set' y los constructores, se podrían cambiar las primitivas de servicio 'private' de los DTOs sin 
preocuparse por los métodos 'public' que permiten el acceso a los mismos y se podrían crear instancias de empleando el patrón 'Builder',
con lo que no nos tendríamos que preocupar de tener distintos constructores para distintas instancias.
En primer lugar, necesitamos importar la librería con 'maven' en el archivo de configuración 'pom.xml':
    
    <dependency>
		<groupId>org.projectlombok</groupId>
		<artifactId>lombok</artifactId>
		<version>1.18.24</version>
		<scope>provided</scope>
	</dependency>
Así podemos incluir las anotaciones en los DTOs y borrar los constructores y métodos públicos 'get' y 'set'. Este sería el ejemplo de uso
de las anotaciones de 'lombok' en el DTO 'PublicAd'

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    @ToString
    public class PublicAd {

        private Integer id;
        private String typology;
        private String description;
        private List<String> pictureUrls;
        private Integer houseSize;
        private Integer gardenSize;
    }
Cómo se puede ver, también propongo incluir la anotación '@ToString'. Esta anotación sobreescribe el método 'toString' de la superclase
'Object' de modo que, en lugar del código "hash", imprime los nombres de las variables del DTO seguidas de su valor. Esto nos puede ser útil
a la hora de incluir ciertra trazabilidad a los procesos. Sobre esto me explayaré mas en el apartado sobre [generalidades](#generalidades).
## Generalidades
Mis consideraciones sobre aspectos más transversales observables en el código.
### Trazabilidad
No se está aplicando niguna trazabilidad, es decir, 'logs'.