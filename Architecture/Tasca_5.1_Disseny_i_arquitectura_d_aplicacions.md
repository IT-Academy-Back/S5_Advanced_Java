# Tasca S5.01 - Disseny i arquitectura d'aplicacions

## Introducció

Fins ara has treballat sobretot amb sistemes relativament simples, sovint propers a un **CRUD**, amb poca lògica de negoci real i amb un pes important de la persistència i dels endpoints. Aquest tipus d'aplicació es útil en molts contextos, com ara panells d'administració, gestors interns, catàlegs senzills o mòduls on gairebé tot consisteix a crear, consultar, editar i esborrar dades. Però no és l'única manera de construir un backend.

En aquesta tasca construiràs un backend del joc del **Blackjack** per començar a treballar una altra manera d'entendre el disseny d'una aplicació: menys centrada en l'estructura típica de controladors, serveis i repositoris, i molt més centrada en el domini i en les regles del negoci.

Quan parlem de disseny del domini, hi ha dues maneres molt diferents d'abordar-lo:

En un **[anemic model](https://martinfowler.com/bliki/AnemicDomainModel.html)**, les classes del domini acostumen a ser poc més que contenidors de dades, i la lògica acaba dispersa normalment en serveis, o si tot va malament als controladors.

En un **[rich model](https://enterprisecraftsmanship.com/posts/what-is-domain-logic/)**, en canvi, les entitats i els objectes del domini concentren comportament, validacions i regles del negoci.

https://www.baeldung.com/java-anemic-vs-rich-domain-objects

En aquesta pràctica ens interessa avançar cap a aquest segon enfocament treballant sobretot amb tècniques de **disseny orientat a objectes** aplicades al backend. Busquem començar a modelar un sistema on els objectes del domini tinguin responsabilitat i on les decisions importants no quedin escampades fora del model.

El context és aquest: fins ara has pogut resoldre problemes on la persistència, els endpoints i el flux general del CRUD tenien més pes que el domini. Aquí farem un pas diferent. El centre ja no és només guardar i recuperar dades, sinó representar bé el comportament del joc i decidir quina estructura ajuda més a mantenir-lo, provar-lo i fer-lo evolucionar.

També és habitual començar un projecte amb una lògica **database-first**: primer es pensa la base de dades, després les taules, i finalment el codi s'adapta a aquesta estructura. Aquest enfocament pot ser una bona opció quan el problema és sobretot administratiu, les regles són simples, el sistema és bàsicament un CRUD i el pes principal és guardar i consultar dades amb rapidesa. En dominis amb regles de negoci més vives, en canvi, acostuma a tenir costos clars: el model acaba massa condicionat per la persistència, les regles es reparteixen entre diverses capes i els tests del comportament real es tornen més difícils de construir.

Un enfocament més proper al **rich model** té altres trade-offs. Et demana pensar millor el domini abans d'implementar, desacoblar dependències i justificar més les decisions d'arquitectura. Acostuma a compensar quan hi ha regles de negoci que importen de debò, fluxos que canvien segons l'estat, necessitat d'auditar què ha passat o quan la testabilitat i l'evolució del codi són part important del problema. A curt termini pot semblar més lent, però a canvi guanyes un model més expressiu, més provable, més coherent amb el negoci i més resistent a canvis quan la lògica deixa de ser trivial.

El que volem és que comencis a dissenyar un backend amb més criteri: un model de domini amb comportament, una arquitectura amb responsabilitats separades i una persistència escollida segons el que necessita cada part del sistema.

Per fer-ho, construiràs un petit joc de Blackjack per a un únic jugador i el dealer, sense frontend obligatori i sense focus en apostes. El centre de la tasca és el backend.

Treballaràs amb dues persistències, però no per duplicar dades, sinó perquè cada una resol una necessitat diferent del sistema:

- **[MongoDB](https://www.mongodb.com/docs/)** guardarà l'estat real de la partida: l'ordre de la baralla, les cartes ja repartides, l'estat de la mà del jugador i del dealer, i qualsevol altra informació necessària perquè el joc pugui continuar exactament des del punt on havia quedat.
- **[MySQL](https://dev.mysql.com/doc/)** guardarà informació derivada de les partides acabades, en un format més fàcil de consultar després, com ara un rànquing, estadístiques acumulades o un resum final de cada partida.

Per connectar aquestes dues peces amb criteri, hauràs d'introduir com a mínim un **[event](https://martinfowler.com/eaaDev/DomainEvent.html)** senzill de domini o d'integració. El cas base recomanat és **`GameFinished`**: quan una partida es tanca, aquest event pot actualitzar a **MySQL** el resum, les estadístiques o el rànquing corresponents, sense barrejar aquesta lògica directament dins de l'aggregate principal.

## Objectius

Amb aquesta tasca hauràs de ser capaç de:

- modelar un domini senzill però real amb un enfocament de **[rich model](https://enterprisecraftsmanship.com/posts/what-is-domain-logic/)**,
- separar clarament **domini**, **aplicació** i **infraestructura**,
- construir una API REST amb **Spring Boot**,
- justificar per què l'estat real de la partida viu a **MongoDB** i la informació derivada a **MySQL**,
- aplicar com a mínim un **[event](https://martinfowler.com/eaaDev/DomainEvent.html)** bàsic per actualitzar o registrar informació derivada,
- documentar i provar el backend amb criteri tècnic,
- dissenyar el codi perquè els **tests unitaris** siguin possibles de veritat: amb dependències desacoblades, **mocks** quan calgui i control dels elements aleatoris del joc,
- i explicar les decisions d'arquitectura d'una manera entenedora.

## Descripció

Hauràs de desenvolupar una API REST en **Java** amb **[Spring Boot](https://docs.spring.io/spring-boot/index.html)** per gestionar un joc de **Blackjack** per a un únic jugador.

La tasca ha d'estar orientada al disseny del backend, no a la capa visual. No s'espera un frontend en aquesta pràctica.

El joc ha d'incloure com a mínim:

- creació d'una partida,
- consulta de l'estat actual de la partida,
- conservació real de l'estat de la partida entre crides,
- acció de demanar carta,
- acció de plantar-se,
- resolució del resultat final,
- i consulta d'algun resum o rànquing derivat de les partides acabades.

També hauràs de tenir en compte una restricció important de negoci: el jugador no pot veure informació que no li tocaria veure. Per tant, l'API no hauria d'exposar al client l'ordre complet de la baralla, les cartes encara no repartides ni informació oculta del dealer mentre la partida continua oberta.

Per evitar interpretacions massa diferents, en aquesta pràctica també s'espera que implementis unes regles mínimes compartides:

- el jugador i el dealer han de començar amb una mà inicial,
- el jugador pot demanar carta mentre la partida continuï oberta,
- si el jugador supera 21, la partida acaba immediatament,
- quan el jugador es planta, el dealer ha de resoldre la seva mà seguint una regla fixa i explícita que has de documentar,
- has de definir com es resolen com a mínim la victòria, la derrota i l'empat,
- i has de deixar clar què passa si hi ha un **blackjack** inicial o si decideixes no contemplar aquest cas.

L'aplicació ha d'estar estructurada amb una arquitectura orientada al domini. Pots implementar-la amb un enfocament proper a **[arquitectura hexagonal](https://alistair.cockburn.us/hexagonal-architecture)**, **[Clean Architecture](https://blog.cleancoder.com/uncle-bob/2011/11/22/Clean-Architecture.html)** o una variant equivalent, sempre que es compleixin aquestes idees:

- el domini no ha de dependre de Spring,
- la persistència no ha de condicionar el model del domini,
- i els casos d'ús han de tenir responsabilitats clares.

## Què s'espera del domini

Aquest exercici té sentit si el model del joc està ben pensat. No busquem un model anèmic amb tota la lògica dins d'un service gegant.

S'espera que identifiquis i modelis amb criteri peces com ara:

- `Game` com a **[agregat](https://martinfowler.com/bliki/DDD_Aggregate.html)** principal,
- `Player`, `Dealer`, `Hand`, `Card` o estructures equivalents,
- estats de partida,
- regles del joc,
- i, si escau, algun **[value object](https://martinfowler.com/bliki/ValueObject.html)** o algun tipus de suport amb significat propi.

Les regles del joc haurien de viure dins del domini. Per exemple:

- quan una partida pot començar,
- quin ordre de cartes té la baralla o l'estat del joc en cada moment,
- quan es pot demanar carta,
- quan el jugador o el dealer s'han de plantar,
- quina informació és visible per al jugador i quina s'ha de mantenir oculta fins al moment adequat,
- quan la partida acaba,
- com es calcula el resultat final,
- i quines dades cal conservar de la partida per poder-la reconstruir i auditar després.

## Pistes de disseny

No cal que utilitzis tots aquests conceptes de manera formal ni que forcis el projecte perquè hi encaixin tots. Però, per resoldre bé aquesta tasca, és probable que t'ajudi entendre i aplicar alguns d'aquests elements:

- **[Agregat](https://martinfowler.com/bliki/DDD_Aggregate.html)**: quina és la unitat principal de coherència del joc i quina informació ha de canviar junta.
- **Entitats de domini i [value objects](https://martinfowler.com/bliki/ValueObject.html)**: quines peces tenen identitat pròpia i quines tenen sentit sobretot pel seu valor dins del domini.
- **Casos d'ús o application services**: on coordines el flux de l'aplicació sense col·locar-hi la lògica del joc.
- **[Domain services](https://martinfowler.com/bliki/EvansClassification.html)**: només si tens una lògica rellevant que no encaixa bé dins d'una entitat o de l'agregat principal.
- **Ports i adapters**: com desacobles el domini de MongoDB, MySQL o de qualsevol altra dependència tècnica.
- **[Esdeveniments](https://martinfowler.com/eaaDev/DomainEvent.html)**: com comuniques que ha passat alguna cosa important, per exemple quan una partida comença, avança o acaba.
- **Projeccions o models de consulta**: quina informació derivada té sentit guardar a MySQL per consultar-la després amb més facilitat.

L'important no és acumular conceptes d'arquitectura, sinó entendre quines et serveixen per donar forma a una solució més clara i coherent.

## Requisits mínims obligatoris

Independentment del nivell al qual vulguis aspirar, la tasca ha de complir com a mínim aquests requisits:

- **Spring Boot** com a base de l'aplicació.
- **API REST** funcional per al flux principal del joc.
- **Model de domini amb comportament**, no només DTOs i serveis CRUD.
- **Separació clara** entre domini, aplicació i infraestructura.
- **MongoDB** per persistir l'agregat principal de la partida.
- **MySQL** per persistir una projecció, resum o rànquing derivat del joc.
- Les dues persistències han de tenir **responsabilitats diferents i justificades**.
- La partida ha de ser **auditable i persistent**: si la baralla es mescla o si ja s'han repartit cartes, l'estat ha de continuar sent coherent a la següent crida i no es pot recalcular de manera arbitrària.
- L'API no pot exposar informació que permeti fer trampes o anticipar el resultat de la partida, com ara cartes ocultes del dealer, l'ordre de la baralla o dades internes que el jugador encara no hauria de conèixer.
- Com a mínim un **[esdeveniment de domini o d'integració](https://martinfowler.com/eaaDev/DomainEvent.html)** senzill, amb utilitat real dins del flux de l'aplicació.
- Gestió consistent d'errors i respostes HTTP coherents.
- **Documentació de l'API** amb **[OpenAPI](https://swagger.io/specification/)** i una interfície com **[Swagger UI](https://swagger.io/tools/swagger-ui/)**.
- **Tests obligatoris** al backend.
- Com a mínim, hauràs d'incloure:
  - **tests unitaris** del domini o dels casos d'ús,
  - **tests d'acceptació o integració** dels endpoints principals.
- No cal que tot el projecte sigui sofisticat, però sí que ha de quedar clar quina part és el mínim funcional i quines decisions has pres perquè el disseny sigui mantenible, testable i compatible amb proves deterministes del joc.
- El backend ha d'assolir una cobertura mínima de proves del **60%**.
- Aquesta mètrica no substitueix la qualitat dels tests: la cobertura ha de reflectir proves útils sobre el flux principal, els casos d'error rellevants i els punts crítics del model.
- `README.md` complet amb instruccions d'execució i explicació de l'arquitectura.

## Abans de començar

Abans d'escriure codi, dedica una estona a entendre bé el domini i a dissenyar-lo.

Si et va bé per ordenar idees, pots escriure abans entre **3 i 5 User Stories** o escenaris d'ús curts. No han de convertir-se en un backlog complet, però sí que et poden ajudar a aclarir el flux principal del joc, el comportament del dealer i la informació derivada que després voldràs consultar.

### 1. Entén les regles del Blackjack

Has d'entendre bé les regles bàsiques del joc abans de modelar res. En aquesta tasca no treballarem apostes ni variants complexes: centra't en una versió clara i controlada del joc per a un únic jugador contra el dealer.

### 2. Decideix quin és l'agregat principal

Abans de pensar en controladors o repositoris, respon aquesta pregunta: quina peça del sistema canvia com una unitat coherent?

En aquesta pràctica, el més natural és que sigui la **partida**.

### 3. Dissenya el model de domini

Abans d'implementar, defineix com a mínim:

- quines entitats o objectes formen el domini,
- quina informació necessita la partida,
- quin estat mínim has de persistir perquè la partida continuï sent la mateixa entre tirades,
- quines operacions formen part del comportament del joc,
- quines regles s'han de validar dins del domini,
- quines dependències caldrà desacoblar per poder provar els casos d'ús amb **mocks** o dobles de prova,
- com controlaràs en tests els elements aleatoris del joc perquè les proves siguin repetibles,
- i quin estat s'ha de persistir a cada base de dades.

### 4. Decideix què va a MongoDB i què va a MySQL

Abans de començar, deixa clar:

- quina informació guarda **MongoDB** com a font principal de la partida,
- quina informació guarda **MySQL** com a projecció o model de consulta,
- i com es connecten aquestes dues parts.

### 5. Defineix l'esdeveniment mínim

Pensa quin esdeveniment pot tenir sentit dins del flux del joc.

Per exemple:

- `GameStarted`
- `CardDrawn`
- `GameFinished`

No cal complicar-ho. L'important és que aquest esdeveniment tingui una utilitat clara, com ara actualitzar una estadística, registrar un resum o sincronitzar una projecció.

## Endpoints orientatius

No és imprescindible que la teva API sigui exactament igual a aquesta llista, però com a mínim ha de cobrir un flux equivalent.

- `POST /games` per crear una partida.
- `GET /games/{id}` per consultar l'estat d'una partida.
- `POST /games/{id}/hit` per demanar carta.
- `POST /games/{id}/stand` per plantar-se i tancar el flux principal.
- `GET /ranking` o un endpoint equivalent per consultar dades derivades guardades a **MySQL**.

Si necessites algun endpoint addicional per donar coherència al teu model, el pots afegir.

## Criteris d'avaluació

Els nivells següents representen tres graus diferents de qualitat, profunditat i maduresa sobre una mateixa pràctica.

## Nivell 1 - Joc funcional i arquitectura base

En aquest nivell es valorarà especialment:

- que el joc de Blackjack funcioni de punta a punta,
- que el flux principal de la partida estigui ben resolt,
- que la partida conservi un estat real i auditable entre crides,
- que l'API només exposi la informació que el jugador hauria de veure en cada moment,
- que hi hagi una separació mínima real entre domini, aplicació i infraestructura,
- que l'estat real de la partida visqui a **MongoDB** i la informació derivada a **MySQL** amb una justificació clara,
- que el model del joc no sigui purament anèmic,
- l'ús de **[Docker](https://docs.docker.com/)** per facilitar l'entorn d'execució,
- que l'API tingui documentació **OpenAPI/Swagger** disponible,
- que existeixin proves automatitzades mínimes útils,
- que la implementació permeti escriure **tests unitaris** reals sense dependre sempre de la infraestructura,
- i que es puguin provar escenaris concrets del joc controlant l'ordre de les cartes o la lògica aleatòria quan calgui,
- i que el projecte es pugui executar seguint les instruccions del `README.md`.

## Nivell 2 - Model ric i criteri arquitectònic

En aquest nivell es valorarà especialment:

- la qualitat real del **model ric**,
- que les regles del joc visquin al domini i no disperses entre controladors i serveis,
- que el model permeti entendre i reconstruir què ha passat a la partida sense reinterpretar-la des de zero a cada petició,
- que el disseny protegeixi correctament la informació oculta del joc i eviti respostes que permetin fer trampes,
- la claredat dels casos d'ús o capa d'aplicació,
- la qualitat de la separació entre ports i adapters, si escau,
- que les dues persistències estiguin ben justificades i no siguin un duplicat arbitrari,
- que l'esdeveniment de domini o integració tingui utilitat real dins del sistema, especialment si serveix per actualitzar la informació derivada de **MySQL** quan una partida es tanca,
- la qualitat dels tests del domini i dels endpoints principals,
- que les decisions d'arquitectura facilitin l'ús de **mocks** o dobles de prova quan calgui aïllar un cas d'ús,
- que el disseny permeti construir proves deterministes del joc sense dependre de resultats aleatoris difícils de reproduir,
- que la cobertura de tests sigui coherent amb els riscos reals del projecte i no es limiti a inflar una mètrica,
- la qualitat de la documentació tècnica i de l'explicació de les decisions d'arquitectura,
- i que el codi sigui prou clar perquè una altra persona pugui entendre què passa i per què.

## Nivell 3 - Maduresa tècnica i qualitat global

En aquest nivell es valorarà especialment:

- una millor qualitat general del disseny i de l'execució,
- una explicació més madura de les decisions arquitectòniques,
- una estratègia de proves més sòlida,
- una millor observabilitat o traçabilitat bàsica,
- i qualsevol millora addicional que aporti valor real a la tasca sense desviar-la del seu objectiu principal.

No es tracta d'afegir tecnologies a l'atzar. Es tracta de demostrar més criteri.

## Lliurament

Per al lliurament final hauràs d'aportar com a mínim:

- l'enllaç al repositori del projecte,
- un `README.md` complet,
- les instruccions per executar l'aplicació,
- les instruccions per executar els tests,
- la ruta o URL de la documentació **Swagger/OpenAPI**,
- una explicació breu de l'arquitectura aplicada,
- una explicació breu de què guarda **MongoDB** i què guarda **MySQL**,
- i una explicació breu de l'esdeveniment de domini o d'integració implementat.

## README mínim recomanat

El `README.md` hauria d'incloure com a mínim:

- objectiu de la pràctica,
- regles del Blackjack que has implementat,
- stack tecnològic,
- estructura general del projecte,
- explicació del model de domini,
- justificació de l'ús de **MongoDB** i **MySQL**,
- explicació de l'esdeveniment implementat,
- instruccions d'execució local,
- configuració necessària,
- com accedir a **Swagger/OpenAPI**,
- com executar les proves,
- i qualsevol limitació o simplificació que hagis decidit aplicar.

## Recursos recomanats

Si necessites suport, pots consultar aquests recursos:

- **Spring Boot**: [Spring Boot Documentation](https://docs.spring.io/spring-boot/index.html)
- **Spring Data MongoDB**: [Spring Data MongoDB Reference](https://docs.spring.io/spring-data/mongodb/reference/)
- **Spring Data JPA**: [Spring Data JPA Reference](https://docs.spring.io/spring-data/jpa/reference/)
- **OpenAPI Specification**: [OpenAPI Documentation](https://swagger.io/specification/)
- **Swagger UI**: [Swagger UI Documentation](https://swagger.io/tools/swagger-ui/)
- **springdoc-openapi**: [springdoc.org](https://springdoc.org/)
- **MongoDB**: [MongoDB Documentation](https://www.mongodb.com/docs/)
- **MySQL**: [MySQL Documentation](https://dev.mysql.com/doc/)
- **Docker**: [Docker Documentation](https://docs.docker.com/)
- **Testing amb Spring Boot**: [Spring Boot Testing Reference](https://docs.spring.io/spring-boot/reference/testing/index.html)
- **Domain Model**: [Martin Fowler - Domain Model](https://martinfowler.com/eaaCatalog/domainModel.html)
- **Domain Logic**: [Enterprise Craftsmanship - What is domain logic?](https://enterprisecraftsmanship.com/posts/what-is-domain-logic/)
- **Always-Valid Domain Model**: [Enterprise Craftsmanship - Always-Valid Domain Model](https://enterprisecraftsmanship.com/posts/always-valid-domain-model)
- **Domain Model Isolation**: [Enterprise Craftsmanship - Domain model isolation](https://enterprisecraftsmanship.com/posts/domain-model-isolation/)
- **Domain-Driven Design**: [Martin Fowler - Domain Driven Design](https://martinfowler.com/bliki/DomainDrivenDesign.html)
- **Hexagonal Architecture**: [Alistair Cockburn - Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture)