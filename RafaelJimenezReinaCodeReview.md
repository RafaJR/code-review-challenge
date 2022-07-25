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
* [Dao](#dao)
* [Generalidades](#generalidades)
* [Tests](#tests)
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
En primer lugar, se necesita la librería de 'h2', por lo que la configuramos en nuestro "pom.xml":

    <dependency>
        <groupId>com.h2database</groupId>
		<artifactId>h2</artifactId>
		<scope>runtime</scope>
    </dependency>
Cómo se puede ver, se ha configurado para el ámbito de ejecución, ya que necesitamos que 'hd' genere la base de datos en tiempo de 
ejecución, inmediatemente después de cargar el servicio.
Y finalmente tenemos que configurar la base de datos en el archivo 'application.yml' de la siguiente manera:

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
y con consola habilitada (http://localhost:8080/h2-console).
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
datos pues, si estos anuncios actualizados de cualquier forma que afecte a su puntuación, dicha puntuación permanecerá inconsistente con los
otros datos que la afectan. Es decir, la puntuación no se corresponderá con la descripción, las fotos y demás. Naturalmente, si se consultan
los anuncios antes de llamar a este 'endpoint', la puntuación no será la correcta, por lo que podremos tener listas de anuncios relevantes
con anuncios que realmente son irrelevantes y viceversa.

Esto podría corregirse haciendo que la puntuación sea un valor auto-calculado, lo cual permitiría prescindir del 'endpoint' de cálculo y de 
los servicios a los que llama. La manera en que se podría implementar una solución como esta, con sus ventajas e inconvenientes, 
se explicará con más detalle en el apartado sobre [Mapeo de entidad mediante anotaciones JPA](#mapeo-de-entidad-mediante-anotaciones-JPA).
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
    public static final int LARGE_FLAT_DESCRIPTION_SCORE = 30;
    public static final int MEDIUM_FLAT_DESCRIPTION_SCORE = 10;
    public static final int LARGE_CHALET_DESCRIPTION_SCORE = 20;
    public static final int SCORE_WORD_SCORE = 5;
    public static final int MEDIUM_FLAT_DESCRIPTION_MIN_LIMIT = 20;
    public static final int MEDIUM_FLAT_DESCRIPTION_MAX_LIMIT = 49;
    public static final int LARGE_CHALET_DESCRIPTION_MIN_LIMIT = 50;
    public static final int FULL_AD_SCORE = 40;
    public static final int IRRELEVANT_SCORE_DEAD_LINE = 40;
    public static final int MAX_SCORE_LIMIT = 100;
    public static final int MIN_SCORE_LIMIT = 0;
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
## Dao
En el apartado "[Base de datos embebida en memoria](#base-de-datos-embebida-en-memoria)" se indicó la necesidad de mapear debidamente las
clases de entidad para poder generar los métodos de acceso a las fuentes de datos con 'JPA'. También se comentó en el apartado sobre 
"[Sobre el 'endpoint' para calcular las puntuaciones](#sobre-el-endpoint-para-calcular-las-puntuaciones)" que el método de calculo de las
puntuaciones se podría trasladar a la entidad haciendo de 'score' un campo autocalculado.

A continuación se explicará como se podrían llevar a cabo estos desarrollos.
### Ajuste en el modelo de arquitectura propuesto
El lugar donde crear estos 'Mapper' en la arquitectura propuesta sería aquí:

    \_[com.idealista]
    \_[dao]
    	\_[entities]
### Mapeo de entidad mediante anotaciones JPA
La implementación de la especificación JPA de Hibernate nos permite realizar fácilmente el mapeo de una clase POJO de java a una tabla de
base de datos, es decir, nos ayuda a generar una entidad.
En primer lugar, necesitamos la librería específica para 'springboot' creada para este fin, por lo que configuramos nuestro "pom.xml" para
que maven se ocupe:

    <dependency>
	    <groupId>org.springframework.boot</groupId>
	    <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
Las entidades mínimas que vamos a necesitar para persistir los datos son las correspondientes a las clases 'Ad' y 'Picture', manteniendo
ambas una relación de "muchos a uno", ya que un anuncio puede tener una o más fotografías. La clases 'Typology' podría ser también entidade
con la misma relación "muchos a uno" con 'Ad', pero ya que sus valores posibles son limitados, para simplificar vamos a mapearla como 
enumeradore dentro de la entidad 'Ad'.

Además del mapeo de las entidades, también vamos a emplear las anotaciones de 'lombok' para ahorro de código y para implementar el patrón
'Builder', del mismo modo que ya hicimos con los DTOs.
#### Entidad 'Ad'
En primer lugar, vamos a mapear la clase más importante, la que ha de contener los datos de los anuncios, la entidad 'Ad':
    
    Entity(name = "ad")
    @Table(name = "T_AD")
    
    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    @ToString
    public class Ad {
        
        @Id
	    @GeneratedValue(strategy = GenerationType.SEQUENCE)
        @Column(name = "ID_AD")
        private Integer id;
        @Enumerated(EnumType.STRING)
        @Column(name = "TYPOLOGY")
        private Typology typology;
        @Column(name = "DESCRIPTION")
        private String description;
        @OneToMany(mappedBy = "fk_ad", cascade = CascadeType.ALL, orphanRemoval = true)
        private List<Picture> pictures;
        @Column(name = "HOUSE_SIZE")
        private Integer houseSize;
        @Column(name = "GARDEN_SIZE")
        private Integer gardenSize;
        @Column(name = "SCORE")
        private Integer score;
        @Column(name = "IRRELEVANT_SINCE")
        private LocalDateTime irrelevantSince;

        public boolean isComplete() {
        return (Typology.GARAGE.equals(typology) && !pictures.isEmpty())
                || (Typology.FLAT.equals(typology) && !pictures.isEmpty() && description != null && !description.isEmpty() && houseSize != null)
                || (Typology.CHALET.equals(typology) && !pictures.isEmpty() && description != null && !description.isEmpty() && houseSize != null && gardenSize != null);
       }
        @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (o == null || getClass() != o.getClass()) return false;
            Ad ad = (Ad) o;
            return Objects.equals(id, ad.id) && typology == ad.typology && Objects.equals(description, ad.description) && Objects.equals(pictures, ad.pictures) && Objects.equals(houseSize, ad.houseSize) && Objects.equals(gardenSize, ad.gardenSize) && Objects.equals(score, ad.score) && Objects.equals(irrelevantSince, ad.irrelevantSince);
        }
    
        @Override
        public int hashCode() {
            return Objects.hash(id, typology, description, pictures, houseSize, gardenSize, score, irrelevantSince);
        }
    }
Algunas aclaraciones sobre el mapeo de esta entidad 'Ad':
- Se ha indicado que se ha de crear tabla denominada "T_AD" asociada a la clase, lo cual hace de ella una entidad:


    Entity(name = "Ad")
    @Table(name = "T_AD")
- El campo 'id' se ha hecho autogenerado de forma secuencial, por lo que no habrá que introducir su valor para guardar nuevos anuncios, pero
servirá para métodos que necesiten acceder a ellos de manera idempotente, por ejemplo para actualizarlos.


    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    @Column(name = "ID_AD")
- El campo "tipology", al corresponderse con un enumerador, se ha mapeado como tal. Los datos se transfieren a la base de datos como 
"String", lo que limitará sus posibles valores en base de datos a los mismos establecidos en dicho enumerador.


    @Enumerated(EnumType.STRING)
    @Column(name = "TYPOLOGY")
- Se ha establecido una relación "uno a muchos" con la entidad 'Picture', lo que significa que en la base de datos, ambas tablas estarán
relacionadas través de una "foreing key" en la entidad 'Picture', lo que posilibilita guardar una o más imágenes para el mismo anuncio.
Los parámtros empleados en la anotación indican dos cosas:
  * Que las actualizaciones de datos realizados sobre esta entidad deben propagarse a todas las entidades relacionadas.
  * Que el borrado de un anuncio implica el borrado de todas sus fotos.


    @OneToMany(mappedBy = "fk_ad", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Picture> pictures;
Además, para agilizar las consultas, va a ser conveniente que esta relación sea bidireccional, por lo que se ha de tener en cuenta a la hora
de mapear tambien la entidad 'Picture', lo cual se explica en [siguiente apartado](#entidad-picture)
- El campo 'score' se ha establecido como transitorio, lo que significa que no se persistirá y podemos autocalcularlo en las consultas, 
haciendo que funcione como un campo más a la hora de filtrar. La razón de esto se explicó en el apartado 
[Sobre el 'endpoint' para calcular las puntuaciones](#sobre-el-endpoint-para-calcular-las-puntuaciones).
En el ejemplo de código anterior, no he manifestado cómo se va a realizar el cálculo de este campo en las consultas; este tema se
desarrollará en un apartado posterior sobre el cálculo de puntuaciones.
- El campo 'irrelevantSince', que es de tipo fecha, se ha establecido como de tipo 'LocalDateTime' en lugar de 'Date', que es una clase de
'Java v8 Date Time API' que tiene algunas ventajas sobre representaciones de fecha de versiones anteriores. Algunas de estas ventajas son su
sencillez de manejo, alta precisión de milisegundos (aunque si no se necesita tanta precisión se puede usar 'LocalDate'), ámplia API con 
métodos para comparar o manipular fechas, inmutabilidad para más seguridad en caso de programación multi-thread y otras.


    @Column(name = "IRRELEVANT_SINCE")
    private LocalDateTime irrelevantSince;
#### Entidad 'Picture'
Esta entidad se podría mapear así:

    Entity(name = "picture")
    @Table(name = "T_PICTURE")
    
    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    @ToString
    public class Picture {

        @Id
	    @GeneratedValue(strategy = GenerationType.SEQUENCE)
        @Column(name = "ID_PICTURE")
        private Integer id;
        @Column(name = "URL")
        private String url;
        @Enumerated(EnumType.STRING)
        @Column(name = "QUALITY")
        private Quality quality;
        @ManyToOne(fetch = FetchType.LAZY)
        @JoinColumn(name = "FK_AD", nullable = false)
        private Ad fk_ad;
    }
Como se puede ver, el mapeo es muy similar al de la entidad 'Ad'; se ha establecido una 'id' autogenerada de forma secuencial y se ha
mapeado como un enumerador el campo 'quality' para limitar sus valores posibles. 

Pero además se le ha añadido un nuevo campo 'fk_ad' para
establecer la relación bidireccional con la entidad 'Ad'. Esta debe de ser "de muchos a uno", inversa a la relación establecida desde la
entidad 'Ad'. Además conviene que las consultas de anuncios no se traigan las fotos por defecto, lo cual se establece en el parámetro de
la anotación. También interesa que el anuncio no pueda ser un campo nulo para esta entidad en ningún caso, ya que no tiene sentido una foto
sin un anuncio asociado.

    @JoinColumn(name = "FK_AD", nullable = false)
    private Ad fk_ad;
### Método de cálculo de puntuación como campo autocalculado de la entidad 'Ad'
Cómo ya se ha indicado en el apartado [Sobre el 'endpoint' para calcular las puntuaciones](#sobre-el-endpoint-para-calcular-las-puntuaciones),
el 'endpoint' para calcular la puntuación de los anuncios va a ser removido por los problemas de consistencia que plantea. 
El método de cálculo de la puntuación de un anuncio se va a mover de la capa de servicio para implementarse como un método que resuelve el 
valor del campo autocalculado 'score' de la entidad 'Ad' previamente a cualquier proceso de persistencia de dicha entidad. Esto resolverá
el problema de consistencia de datos, ya que solo se podrá persistir un anuncio después de que el capo 'score' haya sido calculado.

La forma de lograr que 'score' se auto-calcule y se fije su valor antes de la persistencia de la entidad 'Ad' es muy fácil con 'JPA'; solo
hay que emplear las anotaciones '@PrePersist' y '@PreUpdate' en el método de cálculo del campo 'score'. Este método de cálculo será muy 
similar al que teníamos en la capa de servicio (método 'calculateScore' de la case 'AdsServiceImpl') solo que esta vez no hará falta pasar
el anuncio 'Ad' por parámetro, ya que estará dentro de esta misma entidad, y se introducirán unas mejoras sobre el algoritmo de cálculo 
para mejorar su eficiencia y hacerlo más mantenible.

En este ejemplo de código, se emplean las constantes sugeridas en el partado sobre [Nombres de constantes poco representativas](#nombres-de-constantes-poco-representativas)
y se van aplican algunos métodos de cálculo que se explicarán un poco más adelante:

    @PrePersist
    @PreUpdate
    private void calculateStore() {
    
        int score = 
            calculateScoreByPics() +
            calculateScoreByDesc() +
            calculateScoreByCompleteness();
    
        score = score < Constants.MIN_SCORE_LIMIT ? Constants.MIN_SCORE_LIMIT : score;
        score = score > Constants.MAX_SCORE_LIMIT ? Constants.MAX_SCORE_LIMIT : score;
    
        this.setScore(score);
        this.setIrrelevantSince(score < Constants.IRRELEVANT_SCORE_DEAD_LINE ? LocalDateTime.now() : null);
    
    }
El cálculo de la puntuación se hace por separado, en un método privado, para cada propiedad del anuncio que influye sobre la misma, siendo
la puntuación final la suma de los resultados obtenidos para cada una de estas propiedades, a saber:
- Puntuación por fotos, que se realiza en el método 'calculateScoreByPics'.
- Puntuación por descripción, que se realiza en el método 'calculateScoreByDesc'.
- Puntuación por completitud, que se realiza en el método 'calculateScoreByCompleteness'.
De esta forma se facilita la localización de errores si se dán, el mantenimiento y la claridad del código. Además, se corrige un error
existente en el método 'calculateScores' de la clase 'AdsServiceImpl', en la línea 119; al final del cálculo de la puntuación por
completitud, el valor no se suma sino que se sobreescribe sobre en variable 'score', que se emplea como acumulador de la puntuación 
definitiva.

Si la puntuación que así se obtiene no es válida para el intervalo de mínimo y máximo establecido (0 - 100), esta se corrige fijándola al 
mínimo o al máximo según sea demasiado baja o demasiado alta respectivamente.

La fecha de irrelevancia se fija al momento actual si el anuncio es irrelevante y a un valor nulo si no lo es.

Una vez fijado el valor del campo 'score', la entidad 'Ad' ya está lista para ser persistida o actualizada.

A continuación se explica el funcionamiento de los métodos de cálculo para cada propiedad del anuncio.

#### Método 'calculateScoreByPics' para el cálculo de la puntuación por fotos del anuncio
He aquí mi propuesta para el método de cálculo de puntuación por fotos:

    private int calculateScoreByPics() {
    
        return getPictures() == null || getPictures().isEmpty() ? Constants.NO_PICS_SCORE :
            this.getPictures().stream()
                .mapToInt(pic -> pic.getQuality().equals(Quality.HD) ? Constants.HD_PIC_SCORE : Constants.SD_PIC_SCORE)
                .sum();
    }
A diferencia de lo que se hacía en el método 'calculateScore' del servicio, en este ejemplo se comprueba si la lista de fotos es nula o 
está vacía, en cuyo caso se devolverá un valor negativo establecido en las constantes, por lo que no tendremos que preocuparnos en el 
cálculo final si el valor ha de restarse o sumarse. Si el anuncio contiene fotos la lista es convertida, mediante una expresión 'lamba' a un
'stream' de números enteros que varían de valor según la calidad de cada foto, para luego sumar todos los valores aplicando el método 'sum'.
Esta es una forma más eficiente y mantenible de realizar el cálculo que la que se emplea en el método 'calculateScore' del servicio mediante
bucles iterativos 'for'.
#### Método 'calculateScoreByDesc' para el cálculo de puntuación por descripción del anuncio
He aquí mi propuesta para el método de cálculo de puntuación por descripción:

    private int calculateScoreByDesc() {

        int descScore = 0;
    
        if (description != null && !description.replace(" ", "").isEmpty()) {
    
          descScore = Constants.DESCRIPTION_IS_PRESENT_SCORE;
    
          String[] wordsArrayDescription = description.split(" ");
          int wordsCount = wordsArrayDescription.length;
    
          if (Typology.FLAT.equals(tipology)) {
            if (wordsCount >= Constants.MEDIUM_FLAT_DESCRIPTION_MIN_LIMIT && wordsCount <= Constants.MEDIUM_FLAT_DESCRIPTION_MAX_LIMIT) {
              descScore += Constants.MEDIUM_FLAT_DESCRIPTION_SCORE;
            } else if (wordsCount >= Constants.LARGE_CHALET_DESCRIPTION_MIN_LIMIT) {
              descScore += Constants.LARGE_FLAT_DESCRIPTION_SCORE;
            }
          }
          if (Typology.CHALET.equals(tipology)) {
            if (wordsCount >= Constants.LARGE_CHALET_DESCRIPTION_MIN_LIMIT) {
              descScore += Constants.LARGE_CHALET_DESCRIPTION_SCORE;
            }
          }
    
          descScore += Arrays.stream(new String[]{"luminoso", "nuevo", "céntrico", "reformado", "ático"})
              .filter(scoreWord -> Arrays.stream(wordsArrayDescription).anyMatch(word -> word.equalsIgnoreCase(scoreWord)))
              .count() * Constants.SCORE_WORD_SCORE;
        }
    
        return descScore;
    }
Se pueden apreciar varias diferencias importantes respecto a la forma de sumar los puntos de la descripción del método de la capa de 
servicio:
- No se recurre a ningún 'wrapper' para controlar el campo 'description' nulo o vacío. Si de todas formas va a haber que controlar que no
contenga solo espacios en blanco, crear este 'Optional' es un gasto de memoria estéril. Naturalmente, solo se cuentan los puntos si la 
descripción tiene contenido, en caso contrario se devuelve cero.


    if (description != null && !description.replace(" ", "").isEmpty()) {
- Se cuentan las palabras del texto empleando el atributo 'lenght' del array que se obtiene de separar el texto por espacios con el método
'split'. Convertir el texto a "List" y emplear la función 'size' cada vez que se necesita obtener el número de palabras es un consumo
innecesario de recursos del servidor.


    String[] wordsArrayDescription = description.split(" ");
    int wordsCount = wordsArrayDescription.length;
- Para contar los puntos por palabras puntuables, en lugar de una sucesión de sentencias 'if' tan larga como la cantidad de palabras que 
puntúan en la descripción, se recurre a un 'predicate' implementado como una 'lambda' para realizar esta función:
  - Se obtiene la lista de palabras que puntúan como un 'stream'.
  - Se filtra esta lista de palabras que puntúan por aquellas que están contenidas en el texto de la descripción. Para ello, se emplea el 
  método 'anyMatch' sobre al 'array' que contiene todas las palabras de la descripción, lo cual nos permite comparar las palabras con 
  el método 'equalsIgnoreCase', que descarta las diferencias por mayúsculas y minúsculas.
  - Se cuentan las palabras puntables que quedan después de filtrarlas y se multiplican por el valor de puntuación por palabra establecido
  en las constantes, que actualmente es 'Constants.SCORE_WORD_SCORE = 5'.


    descScore += Arrays.stream(new String[]{"luminoso", "nuevo", "céntrico", "reformado", "ático"})
              .filter(scoreWord -> Arrays.stream(wordsArrayDescription).anyMatch(word -> word.equalsIgnoreCase(scoreWord)))
              .count() * Constants.SCORE_WORD_SCORE;

Ejemplo: Si tenemos el texto de descripción como este:
    "Vendo preciosa casa de campo en Alpedrete del Condado, en estado casi **nuevo**, garage recién **reformado** y **luminoso** jardín".
El filtrado de las palabras puntuables arrojaría este resultado:
    {nuevo, reformado, luminoso}
Cuyo conteo arrojaría un resultado de tres palabras puntuables, que al multiplicarse por los cinco puntos establecidos por palabra, da un
resultado de 15 puntos.
#### Método 'calculateScoreByCompleteness' para el cálculo de la puntuación por completitud del anuncio
Para este cálculo, simplemente vamos a aprovechar el método 'isComplete' ya existente en la entidad 'Ad' y vamos a devolver la puntuación
establecida en las constantes o cero según el anuncio sea completo o no.

    private int calculateScoreByCompleteness() {
    
        return isComplete() ? Constants.FULL_AD_SCORE : 0;
    }
## Generalidades
Mis consideraciones sobre aspectos más transversales observables en el código.
### Trazabilidad
No se está aplicando ninguna trazabilidad, es decir, 'logs'.
Se podría implentar fácilmente la trazabilidad con una anotación de la librería 'lombok', como esta:

    @Slf4j
Los lugares en los que sería más interesante aplicar estos 'logs' de trazabilidad será en los controladores, las clases de la capa de 
servicio, la clases de la capa 'dao' y, ya que se ha propuesto aplicar métodos para campos auto-calculados en las entidades, también en
estos métodos.

Sería conveniente introducir 'logs' informativos de aviso de inicio y fin de procesos, como por ejemplo al inicio y fin de una consulta
de anuncios. Estos podrían mostrarse u omitirse dependiendo de la configuración del servidor para ahorrar recursos en entornos de producción 
y disponer de más trazabilidad en entornos de desarrollo. También sería muy útil tener 'logs' de aviso de error en caso de excepciones.

Este es un ejemplo de como se podría aplicar la trazabilidad en el controlador principal, al menos en el 'endpoint' de consulta de anuncios
relevantes:

    @RestController
    @Slf4j
    public class AdsController {
    
        @Autowired
        private AdsService adsService;
    
        @GetMapping("/ads/quality")
        public ResponseEntity<List<QualityAd>> qualityListing() {
            return ResponseEntity.ok(adsService.findQualityAds());
        }
    
        @GetMapping("/ads/public")
        public ResponseEntity<?> publicListing() {
          try {
            log.debug("The relevant ads query is going to be thrown.");
            return ResponseEntity.ok(adsService.findPublicAds());
          }catch(Exception e) {
            log.error("An error happened when trying to find the relevant ads: {}.", e.getMessage());
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("An error happened when trying to find the relevant ads.");
          }
        }
    
        @GetMapping("/ads/score")
        public ResponseEntity<Void> calculateScore() {
            adsService.calculateScores();
            return ResponseEntity.accepted().build();
        }
    }
### Comentarios
En general hay pocos comentarios aclaratorios acerca del propósito de clases y métodos, funcionamiento de partes del código 
o del uso de las variables.

Por ejemplo, en el controlador se podría comentar en la cabecera de cada método el proósito del 'endpoint' que resuelve, y en la capa de 
servicio se podría comentar que se óbtiene de las consultas o la parte en la que se hace el mapeo para el DTO de salida.
## Tests
En general los tests son muy pobres.
Faltaría al menos un test de integración que pudiese cargar todo el contexto de 'Spring' y probar los métodos del controlador.
Una aproximación podría ser así:

    @ExtendWith(SpringExtension.class)
    @SpringBootTest
    @AutoConfigureMockMvc
    @Transactional
    @SqlGroup({
    @Sql(scripts = "/insertAds.sql", executionPhase = ExecutionPhase.BEFORE_TEST_METHOD),
    @Sql(scripts = "/resetAds.sql", executionPhase = ExecutionPhase.AFTER_TEST_METHOD)
    })
    @Slf4j
    public class AdsControllerTests {
    
    @Autowired
    private MockMvc mockMvc;
    
    @BeforeAll
    static void beforeAll() {
    
        log.info("AddsController tests started");
    }
    
    @BeforeEach
    void beforeEach(TestInfo testInfo) {
    
        log.info("AddsController test '{}' started.", testInfo.getDisplayName());
    }
    
    private final String FIND_QUALITY_ADS = "/ads/quality";
    private final String FIND_PUBLIC_ADS = "/ads/public";
    
    @Test
    void qualityListingTest() {
    
        mockMvc.perform(get(FIND_QUALITY_ADS).contentType(CONTENT_TYPE)
            .andExpect(status().is2xxSuccessful());
    
    }
    
    @Test
    void publicListing() {
    
        mockMvc.perform(get(FIND_PUBLIC_ADS).contentType(CONTENT_TYPE)
            .andExpect(status().is2xxSuccessful());
    
    }
    
    }
Las anotaciones iniciales aseguran que todos los test se podrán ejecutar con el contexto de 'Spring' completo, lo que nos permite realizar
tests que realmente sean de integración.

    @ExtendWith(SpringExtension.class)
    @SpringBootTest
    @AutoConfigureMockMvc
Al hacer la clase de tests transacciona, nos aseguramos de que ninguna ejecución de tests va a tener conseguencias sobre los datos reales
que usa la aplicación.

    @Transactional
La notación en la que se establecen unos archivos '.sql' servirá para cargar estos archivos en la base de datos antes y después de cada test,
lo que garantizará que todos los test se ejecuten con el contexto de datos que necesitan y que ninguna ejecución afectará al test siguiente.

    @SqlGroup({
    @Sql(scripts = "/insertAds.sql", executionPhase = ExecutionPhase.BEFORE_TEST_METHOD),
    @Sql(scripts = "/resetAds.sql", executionPhase = ExecutionPhase.AFTER_TEST_METHOD)
    })
Sabemos que la clase de tests se está ejecutando y cual de tus tests se está ejecutando en cada momento gracias a la trazabilidad aplicada
en los métodos anotados de esta manera:

    @BeforeAll    
    @BeforeEach
Estos test son solo una aproximación a lo que deberían ser unos verdaderos tests de integración; no hacen ningún control sobre los datos 
de salida, pero a menos verifican de los estados 'http' de respuesta son los correctos.