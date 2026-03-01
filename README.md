# Descripción General

📘 Este proyecto basado en FastAPI y su documentación representan una interpretación práctica de los principios de
Clean Architecture y Command Query Responsibility Segregation (CQRS) con elementos de Domain-Driven Design (DDD).
Aunque no pretende servir como referencia exhaustiva ni como aplicación estricta de estas metodologías, el proyecto
demuestra cómo sus ideas fundamentales pueden ponerse en práctica de manera efectiva en Python.
Si son nuevas para ti, consulta la sección [Recursos Útiles](#recursos-útiles).

# Tabla de Contenidos

1. [Descripción General](#descripción-general)
2. [Principios de Arquitectura](#principios-de-arquitectura)
    1. [Introducción](#introducción)
    2. [Enfoque por Capas](#enfoque-por-capas)
    3. [Regla de Dependencias](#regla-de-dependencias)
        1. [Nota sobre Adaptadores](#nota-sobre-adaptadores)
    4. [Enfoque por Capas (Continuación)](#enfoque-por-capas-continuación)
    5. [Inversión de Dependencias](#inversión-de-dependencias)
    6. [Inyección de Dependencias](#inyección-de-dependencias)
    7. [CQRS](#cqrs)
3. [Proyecto](#proyecto)
    1. [Grafos de Dependencias](#grafos-de-dependencias)
    2. [Estructura](#estructura)
    3. [Stack Tecnológico](#stack-tecnológico)
    4. [API](#api)
        1. [General](#general)
        2. [Cuenta](#cuenta-apiv1account)
        3. [Usuarios](#usuarios-apiv1users)
    5. [Configuración](#configuración)
        1. [Archivos](#archivos)
        2. [Flujo](#flujo)
        3. [Entorno Local](#entorno-local)
        4. [Otros Entornos](#otros-entornos-devprod)
        5. [Añadir Nuevos Entornos](#añadir-nuevos-entornos)
4. [Recursos Útiles](#recursos-útiles)
5. [Apoya el Proyecto](#-apoya-el-proyecto)
6. [Agradecimientos](#agradecimientos)

# Principios de Arquitectura

## Introducción

Este repositorio puede resultar útil para quienes buscan una implementación backend en Python que sea tanto
independiente del framework como independiente del almacenamiento (a diferencia de Django).
Dicha flexibilidad se puede lograr utilizando un framework web que no imponga un diseño de software estricto
(como FastAPI) y aplicando una arquitectura por capas basada en la propuesta por Robert Martin, que exploraremos
a continuación.

La explicación original de los conceptos de Clean Architecture se puede
encontrar [aquí](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html).
Si aún te preguntas por qué importa Clean Architecture, lee el artículo — solo toma unos 5 minutos.
En esencia, se trata de hacer tu aplicación independiente de sistemas externos y altamente testeable.

<p align="center">
  <img src="docs/Robert_Martin_CA.png" alt="Diagrama de Clean Architecture" />
  <br><em>Figura 1: Diagrama de Clean Architecture de <b>Robert Martin</b></em>
</p>

> "Un programa de computadora es una descripción detallada de la **política** mediante la cual las entradas se
> transforman en salidas."
>
> — Robert Martin

Las políticas más abstractas definen las reglas de negocio fundamentales, mientras que las menos abstractas manejan
las operaciones de E/S.
Al estar más cerca de los detalles de implementación, las políticas menos abstractas tienen más probabilidades de cambiar.
Una **capa** representa un conjunto de componentes que expresan políticas al mismo nivel de abstracción.

Los círculos concéntricos representan los límites entre diferentes capas.
El significado de las flechas en el diagrama se discutirá [más adelante](#regla-de-dependencias).
Por ahora, nos centraremos en el propósito de las capas.

## Enfoque por Capas

![#gold](https://placehold.co/15x15/gold/gold.svg) **Capa de Dominio**

- El **modelo de dominio** es un conjunto de conceptos, reglas y comportamientos que definen qué es el negocio
  (contexto) y cómo opera.
  Se expresa en **lenguaje ubicuo** — terminología consistente compartida entre desarrolladores y expertos del dominio.
  La capa de dominio implementa el modelo de dominio en código; esta implementación a menudo se llama modelo de dominio.
- Las reglas de dominio más estrictas son los **invariantes** — condiciones que siempre deben cumplirse para el modelo.
  Hacer cumplir los invariantes significa mantener la consistencia de datos en el modelo.
  Esto se puede lograr mediante **encapsulación**, que oculta el estado interno y acopla datos con comportamiento.
- Los bloques de construcción del modelo de dominio son (sin limitarse a estos):
    - **objetos de valor** — tipos de negocio inteligentes (sin identidad, inmutables, iguales por valor).
    - **entidades** — objetos de negocio (tienen identidad y ciclo de vida, iguales por identidad).
    - **servicios de dominio** — contenedores para comportamiento que no tiene lugar en los componentes anteriores.
- Otros bloques de construcción del modelo de dominio, no utilizados en este proyecto pero importantes para DDD avanzado:
    - **agregados** — grupos de entidades (1+) que deben cambiar juntas como una sola unidad,
      gestionados exclusivamente a través de su raíz, definiendo los límites de la consistencia transaccional.
    - **repositorios** — abstracciones que emulan colecciones de raíces de agregados.
- El modelo de dominio se sitúa en un espectro que va de anémico a rico.
    - **anémico** — tipos de datos simples, las entidades son solo contenedores de datos, las reglas y comportamientos
      viven fuera.
    - **rico** — los objetos de valor y las entidades encapsulan datos y reglas;
      los invariantes se aplican internamente, de modo que el propio modelo previene estados inválidos.
      Para los componentes: anémico significa sin comportamiento interno, rico — lo contrario.
- Los servicios de dominio originalmente representan operaciones que no pertenecen naturalmente a una entidad o un
  objeto de valor específico.
  Pero en proyectos con entidades anémicas, también pueden contener lógica que de otro modo estaría en esas entidades.
- En las primeras etapas de desarrollo, cuando el modelo de dominio aún no está claramente definido,
  recomiendo mantener las entidades planas y anémicas, aunque esto último debilite la encapsulación.
  Una vez que la lógica de dominio esté bien establecida, algunas entidades pueden, como raíces de agregados, volverse
  no planas y ricas.
  Esto es lo que mejor hace cumplir los invariantes, pero puede ser difícil de diseñar de una vez por todas.
- Prefiere objetos de valor ricos desde el principio, liberando a las entidades y servicios de una carga excesiva
  de reglas locales.
- Considera la capa de dominio como la parte más importante, estable e independiente de un sistema.

![#red](https://placehold.co/15x15/red/red.svg) **Capa de Aplicación**

- El negocio define un **caso de uso** como una especificación de comportamiento observable que entrega valor al
  alcanzar un objetivo.
- Dentro de un caso de uso, el comportamiento es ejecutado por un **actor** — posiblemente un cliente del sistema
  de software.
- El actor ejecuta el caso de uso en pasos, algunos de los cuales requieren interacción con el sistema.
  Estas interacciones paso a paso con el sistema son manejadas en la capa de aplicación por **interactores**.
  En otras palabras, cada interactor maneja una sola operación de negocio que corresponde a un paso dentro del caso
  de uso.
- Los interactores son sin estado y no pueden llamarse entre sí, a diferencia de los casos de uso.
  Cada uno se invoca de forma independiente — típicamente por controladores externos como controladores HTTP,
  consumidores de mensajes o tareas programadas.
- El interactor orquesta la lógica de dominio y las llamadas externas necesarias para realizar la operación.
  Sus responsabilidades principales pueden incluir la verificación de permisos y la gestión de transacciones.
  Para acceder a sistemas externos, los interactores se apoyan en **interfaces (puertos)** que abstraen los detalles
  de infraestructura.
- El interactor usa **DTOs (Data Transfer Objects)** para intercambiar datos serializables con capas externas.
  Estos son transportadores simples, sin comportamiento — el transporte entre capas para contratos externos.
- Si la lógica se reutiliza entre interactores: extrae un servicio de aplicación cuando cae bajo las responsabilidades
  típicas de un interactor; de lo contrario, considera evolucionar el modelo de dominio para incluirla.
  Dicha evolución es una parte normal del enriquecimiento del modelo.
- Juntas, las capas de dominio y aplicación forman el **núcleo** del sistema.

![#green](https://placehold.co/15x15/green/green.svg) **Capa de Infraestructura**

- Esta capa es responsable de adaptar el núcleo a sistemas externos.
- Consiste en **adaptadores**: conductores y conducidos.
  Los adaptadores conductores invocan al núcleo, traduciendo solicitudes externas en llamadas a interactores.
  Los adaptadores conducidos (implementaciones de puertos) son invocados por el núcleo a través de puertos, permitiendo
  al núcleo interactuar con sistemas externos (bases de datos, APIs, sistemas de archivos, etc.) manteniendo la lógica
  de negocio desacoplada.
- La lógica de adaptadores relacionada puede agruparse en un **servicio de infraestructura**.

> [!IMPORTANT]
> - Clean Architecture no prescribe ningún número particular de capas.
    Lo clave es seguir la Regla de Dependencias, que se explica en la siguiente sección.

## Regla de Dependencias

Una dependencia ocurre cuando un componente de software depende de otro para funcionar.
Si dividieras todos los bloques de código en módulos separados, las dependencias se manifestarían como importaciones
entre esos módulos.
Típicamente, las dependencias se representan gráficamente en estilo UML de tal manera que

> [!IMPORTANT]
> - `A -> B` (**A apunta a B**) significa **A depende de B**.

El principio clave de Clean Architecture es la **Regla de Dependencias**.
Esta regla establece que **los componentes de software más abstractos no deben depender de los más concretos.**
En otras palabras, las dependencias nunca deben apuntar hacia afuera.

> [!IMPORTANT]
> - Las capas de dominio y aplicación pueden importar herramientas y bibliotecas externas en la medida necesaria para
    describir la lógica de negocio — aquellas que extienden las capacidades del lenguaje de programación (utilidades
    matemáticas/numéricas, conversión de zonas horarias, modelado de objetos, etc.). Esto sacrifica algo de estabilidad
    del núcleo a cambio de claridad y expresividad. Lo que no es aceptable son dependencias que vinculen la lógica de
    negocio con detalles de implementación (incluidos frameworks) o con sistemas fuera de proceso (bases de datos,
    brokers, sistemas de archivos, SDKs en la nube, etc.).
>
> - Los componentes dentro de la misma capa **pueden depender entre sí.** Por ejemplo, los componentes en la capa de
    Infraestructura pueden interactuar entre sí sin cruzar a otras capas.
>
> - Los componentes en cualquier capa externa pueden depender de componentes en cualquier capa interna, no
    necesariamente la más cercana a ellos. Por ejemplo, los componentes en la capa de Presentación pueden depender
    directamente de la capa de Dominio, saltándose las capas de Aplicación e Infraestructura.
>
> - Evita que la lógica de negocio se filtre a detalles periféricos, como lanzar excepciones específicas del negocio
    en la capa de Infraestructura sin relanzarlas en la lógica de negocio o declarar reglas de dominio fuera de la
    capa de Dominio.
>
> - En casos específicos donde las restricciones de la base de datos aplican reglas de negocio, la capa de
    Infraestructura puede lanzar excepciones específicas del dominio, como `UsernameAlreadyExistsError` para una
    violación de `UNIQUE CONSTRAINT`.
    Manejar estas excepciones en la capa de Aplicación asegura que cualquier lógica de negocio implementada en
    adaptadores permanezca bajo control.
>
> - Evita introducir elementos en capas internas que existan específicamente para soportar capas externas.
    Por ejemplo, podrías estar tentado a colocar algo en la capa de Aplicación que existe únicamente para soportar
    una pieza específica de infraestructura.
    A primera vista, basándose en las importaciones, podría parecer que la Regla de Dependencias no se viola. Sin
    embargo, en realidad, has roto la idea central de la regla al integrar preocupaciones de infraestructura (más
    concretas) en la lógica de negocio (más abstracta).

### Nota sobre Adaptadores

La **capa de Infraestructura** en Clean Architecture actúa como la capa de adaptadores — conectando la aplicación con
sistemas externos.
En este proyecto, tratamos tanto la **Infraestructura** como la **Presentación** como adaptadores, ya que ambas adaptan
la aplicación al mundo exterior.
En cuanto a la dirección de las dependencias, el diagrama de R. Martin en la Figura 1 puede, sin pérdida significativa,
ser reemplazado por uno más conciso y pragmático — donde la capa de adaptadores sirve como un "puente", dependiendo
tanto de las capas internas de la aplicación como de los componentes externos.
Este ajuste implica **invertir** la flecha de la capa azul a la capa verde en el diagrama de R. Martin.

La solución propuesta es un **compromiso**.
No sigue estrictamente el concepto original de R. Martin, pero evita introducir abstracciones excesivas con
implementaciones fuera de los límites de la aplicación.
Perseguir la pureza en la capa más externa es más probable que resulte en sobreingeniería que en beneficios prácticos.

Mi enfoque conserva casi todas las ventajas de Clean Architecture mientras simplifica el desarrollo en el mundo real.
Cuando sea necesario, los adaptadores pueden eliminarse junto con los componentes externos para los que fueron escritos,
lo cual no supone un problema significativo.

Acordemos, para este proyecto, revisar el principio:

Original:
> "Las dependencias nunca deben apuntar hacia afuera."

Revisado:
> "Las dependencias nunca deben apuntar hacia afuera **dentro del núcleo**."

<div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(400px, 1fr)); gap: 10px; justify-items: center;">
  <img src="docs/onion_1.svg" alt="Interpretación Revisada de CA-D" style="width: 400px; height: auto;" />
  <img src="docs/onion_2.svg" alt="Interpretación Revisada de CA-D, alternativa" style="width: 400px; height: auto;" />
</div>
<p align="center" style="font-size: 14px;">
  <em>Figura 2: <b>Interpretación Revisada</b> de Clean Architecture<br>
  (diagramada — representación original y alternativa)
  </em>
</p>

## Enfoque por Capas (Continuación)

![#blue](https://placehold.co/15x15/blue/blue.svg) **Capa de Presentación**

> [!NOTE]
> En el diagrama original, la capa de Presentación no está explícitamente diferenciada y en su lugar se incluye dentro
> de la capa de Adaptadores de Interfaz. Elegí introducirla como una capa separada, marcada en azul, ya que la veo
> como aún más externa en comparación con los adaptadores típicos.

- Esta capa maneja las solicitudes externas e incluye **controladores** que validan las entradas y las pasan a los
  interactores en la capa de Aplicación. Las capas más abstractas del programa asumen que los datos de la solicitud
  ya están validados, permitiéndoles centrarse únicamente en su lógica central.
- Los controladores deben ser lo más delgados posible, sin contener más lógica que la validación básica de entrada
  y el enrutamiento. Su rol es actuar como intermediarios entre la aplicación y los sistemas externos (ej., FastAPI).

> [!IMPORTANT]
> - La validación **_básica_**, como verificar si la estructura de la solicitud entrante coincide con la estructura del
    modelo de solicitud definido (ej., seguridad de tipos y campos obligatorios) debe ser realizada por los
    controladores en esta capa, mientras que la validación de **_reglas de negocio_** (ej., asegurar que el dominio
    del email está permitido, verificar la unicidad del nombre de usuario, o comprobar si un usuario cumple la edad
    requerida) pertenece a la capa de Dominio o Aplicación.
> - La validación de reglas de negocio a menudo involucra relaciones entre campos, como asegurar que un descuento
    se aplique solo dentro de un rango de fechas específico o que un código de promoción sea válido para pedidos
    por encima de cierto total.
> - **Considera cuidadosamente** el uso de Pydantic para la validación de reglas de negocio. Aunque es conveniente,
    los modelos de Pydantic son más lentos que los dataclasses regulares y reducen la estabilidad del núcleo de la
    aplicación al acoplar la lógica de negocio con una biblioteca externa.
> - Si eliges Pydantic (o una herramienta similar incluida con el framework web) para las definiciones de modelos
    de negocio, asegúrate de que un modelo Pydantic en las capas de negocio sea un modelo separado del de la capa
    de Presentación, incluso si su estructura parece idéntica. Mezclar lógica de presentación de datos con lógica
    de negocio es un error común cometido al inicio del desarrollo para ahorrar esfuerzo en la creación de modelos
    separados y el mapeo de campos, a menudo por no entender que las similitudes estructurales son temporales.

![#gray](https://placehold.co/15x15/gray/gray.svg) **Capa Externa**

> [!NOTE]
> En el diagrama original, los componentes externos se incluyen en la capa azul (Frameworks y Drivers).
> Los he marcado en gris para distinguirlos claramente de las capas dentro de los límites de la aplicación.

- Esta capa representa componentes completamente externos como frameworks web (ej., el propio FastAPI), bases de datos,
  APIs de terceros y otros servicios.
- Estos componentes operan fuera de la lógica central de la aplicación y pueden ser fácilmente reemplazados o
  modificados sin afectar las reglas de negocio, ya que interactúan con la aplicación solo a través de las capas de
  Presentación e Infraestructura.

<p align="center">
  <img src="docs/dep_graph_basic.svg" alt="Grafo Básico de Dependencias" />
  <br><em>Figura 3: Grafo Básico de Dependencias</em>
</p>

## Inversión de Dependencias

La técnica de **inversión de dependencias** permite invertir dependencias **introduciendo una interfaz** entre
componentes, permitiendo que una capa interna se comunique con una capa externa mientras se adhiere a la Regla de
Dependencias.

<p align="center">
  <img src="docs/dep_graph_inv_corrupted.svg" alt="Dependencia Corrupta" />
  <br><em>Figura 4: Dependencia <b>Corrupta</b></em>
</p>

En este ejemplo, el componente de Aplicación depende directamente del componente de Infraestructura, violando la
Regla de Dependencias.
Esto crea dependencias "corruptas", donde los cambios en la capa de Infraestructura pueden propagarse y afectar
involuntariamente a la capa de Aplicación.

<p align="center">
  <img src="docs/dep_graph_inv_correct.svg" alt="Dependencia Correcta" />
  <br><em>Figura 5: Dependencia <b>Correcta</b></em>
</p>

En el diseño correcto, el componente de la capa de Aplicación depende de una **abstracción (puerto)**, y el componente
de la capa de Infraestructura **implementa** la interfaz correspondiente.
Esto convierte al componente de Infraestructura en un adaptador para el puerto, convirtiéndolo efectivamente en un
plugin para la capa de Aplicación.
Tal diseño se adhiere al **Principio de Inversión de Dependencias (DIP)**, minimizando el impacto de los cambios de
infraestructura en la lógica de negocio central.

## Inyección de Dependencias

La idea detrás de la **Inyección de Dependencias** es que un componente no debería crear las dependencias que necesita,
sino recibirlas.
De esta definición, queda claro que una forma común de implementar DI es pasando las dependencias como argumentos al
método `__init__` o a funciones.

¿Pero cómo exactamente deben inicializarse (y finalizarse) estas dependencias?

Los **frameworks de DI** ofrecen una solución elegante al crear automáticamente los objetos necesarios (mientras
gestionan su **ciclo de vida**) e inyectarlos donde se necesiten.
Esto hace que el proceso de inyección de dependencias sea mucho más limpio y fácil de gestionar.

<p align="center">
  <img src="docs/dep_graph_inv_correct_di.svg" alt="Dependencia Correcta con DI" />
  <br><em>Figura 6: Dependencia <b>Correcta</b> <b>con DI</b></em>
</p>

FastAPI proporciona un **mecanismo de DI** integrado llamado [Depends](https://fastapi.tiangolo.com/tutorial/dependencies/),
que tiende a filtrarse a diferentes capas de la aplicación. Esto crea un acoplamiento fuerte con FastAPI, violando los
principios de Clean Architecture, donde el framework web pertenece a la capa más externa y debería ser fácilmente
reemplazable.

Refactorizar el código para eliminar `Depends` al cambiar de framework puede ser innecesariamente costoso. También tiene
[otras limitaciones](https://dishka.readthedocs.io/en/stable/alternatives.html#why-not-fastapi) que están fuera del
alcance de este README. Personalmente, prefiero [**Dishka**](https://dishka.readthedocs.io/en/stable/index.html) — una
solución que evita estos problemas y se mantiene agnóstica al framework.

## CQRS

El proyecto implementa Command Query Responsibility Segregation (**CQRS**) — un patrón que separa las operaciones de
lectura y escritura en caminos distintos.

- Los **Comandos** (a través de interactores) manejan las operaciones de escritura y las lecturas críticas para el
  negocio usando gateways de comandos que trabajan con entidades y objetos de valor.
- Las **Consultas** se implementan a través de servicios de consulta (contrato similar a los interactores) que usan
  gateways de consulta para obtener datos optimizados para la presentación como modelos de consulta.

Esta separación permite:

- Operaciones de lectura eficientes a través de gateways de consulta especializados, evitando cargar modelos completos
  de entidades.
- Optimización del rendimiento adaptando la recuperación de datos a los requisitos específicos de cada vista.
- Flexibilidad para combinar datos de múltiples modelos en operaciones de lectura con selección mínima de campos.

# Proyecto

## Grafos de Dependencias

<details>
  <summary>Controlador de Aplicación - Interactor</summary>

  <p align="center">
  <img src="docs/application_controller_interactor.svg" alt="Controlador de Aplicación - Interactor" />
  <br><em>Figura 7: Controlador de Aplicación - Interactor</em>
  </p>

En la capa de presentación, un modelo Pydantic aparece cuando se trabaja con FastAPI y se necesita mostrar información
detallada en la documentación de OpenAPI.
También puede resultarte conveniente validar ciertos campos usando Pydantic;
sin embargo, ten cuidado de no filtrar reglas de negocio en la capa de presentación.

Para los datos de solicitud, un `dataclass` simple suele ser suficiente.
A diferencia de alternativas más ligeras, proporciona acceso por atributos, lo cual es más conveniente para trabajar
en la capa de aplicación.
Sin embargo, dicho acceso es innecesario para los datos devueltos al cliente, donde un `TypedDict` es suficiente (es
aproximadamente el doble de rápido de crear que un dataclass con slots, con tiempos de acceso comparables).

</details>

<details>
  <summary>Interactor de Aplicación</summary>

  <p align="center">
  <img src="docs/application_interactor.svg" alt="Interactor de Aplicación" />
  <br><em>Figura 8: Interactor de Aplicación</em>
  </p>

</details>

<details>
  <summary>Interactor de Aplicación - Adaptador</summary>

  <p align="center">
  <img src="docs/application_interactor_adapter.svg" alt="Interactor de Aplicación - Adaptador" />
  <br><em>Figura 9: Interactor de Aplicación - Adaptador</em>
  </p>

</details>

<details>
  <summary>Dominio - Adaptador</summary>

  <p align="center">
  <img src="docs/domain_adapter.svg" alt="Dominio - Adaptador" />
  <br><em>Figura 10: Dominio - Adaptador</em>
  </p>

</details>

<details>
  <summary>Controlador de Infraestructura - Handler</summary>
  <p align="center">
  <img src="docs/infrastructure_controller_handler.svg" alt="Controlador de Infraestructura - Handler" />
  <br><em>Figura 11: Controlador de Infraestructura - Handler</em>
  </p>

Un handler de infraestructura puede ser necesario como solución temporal en casos donde existe un contexto separado pero
no está físicamente separado en un dominio distinto (ej., no implementado como un módulo independiente dentro de una
aplicación monolítica).
En tales casos, el handler opera como un interactor a nivel de aplicación pero reside en la capa de infraestructura.

Inicialmente, llamé a estos handlers interactores, pero la comunidad reaccionó muy negativamente a la idea de
interactores en la capa de infraestructura, negándose a reconocer que estos esencialmente pertenecen a otro contexto.

En esta aplicación, dichos handlers incluyen los que gestionan cuentas de usuario, como registro, inicio de sesión y
cierre de sesión.

</details>

<details>
  <summary>Handler de Infraestructura</summary>
  <p align="center">
  <img src="docs/infrastructure_handler.svg" alt="Handler de Infraestructura" />
  <br><em>Figura 12: Handler de Infraestructura</em>
  </p>

Los puertos en infraestructura no son comunes — típicamente, solo están presentes las implementaciones concretas.
Sin embargo, en este proyecto, dado que tenemos una capa separada de adaptadores (presentación) ubicada fuera de la
infraestructura, los puertos son necesarios para cumplir con la regla de dependencias.

</details>

<details>

El **Identity Provider (IdP)** abstrae los detalles de autenticación, vinculando el contexto de negocio principal con el
contexto de autenticación. En este ejemplo, el contexto de autenticación no está físicamente separado, convirtiéndolo en
un detalle de infraestructura. Sin embargo, puede potencialmente evolucionar a un dominio separado.

  <summary>Identity Provider</summary>
  <p align="center">
  <img src="docs/identity_provider.svg" alt="Identity Provider" />
  <br><em>Figura 13: Identity Provider</em>
  </p>

Normalmente, se espera que el IdP proporcione toda la información sobre el usuario actual.
Sin embargo, en este proyecto, dado que los roles no se almacenan en sesiones o tokens, recuperarlos en el contexto
principal era más natural.

</details>

## Estructura

```
.
├── config/...                                   # archivos de configuración y scripts, incluye Docker
├── Makefile                                     # atajos para configuración y tareas comunes
├── scripts/...                                  # scripts auxiliares
├── pyproject.toml                               # configuración de herramientas y entorno (uv)
├── ...
└── src/
    └── app/
        ├── domain/                              # capa de dominio
        │   ├── services/...                     # servicios de la capa de dominio
        │   ├── entities/...                     # entidades (tienen identidad)
        │   │   ├── base.py                      # declaraciones base
        │   │   └── ...                          # entidades concretas
        │   ├── value_objects/...                # objetos de valor (sin identidad)
        │   │   ├── base.py                      # declaraciones base
        │   │   └── ...                          # objetos de valor concretos
        │   └── ...                              # puertos, enums, excepciones, etc.
        │
        ├── application/...                      # capa de aplicación
        │   ├── commands/                        # operaciones de escritura, lecturas críticas para el negocio
        │   │   ├── create_user.py               # interactor
        │   │   └── ...                          # otros interactores
        │   ├── queries/                         # operaciones de lectura optimizadas
        │   │   ├── list_users.py                # servicio de consulta
        │   │   └── ...                          # otros servicios de consulta
        │   └── common/                          # objetos comunes de la capa
        │       ├── services/...                 # autorización, etc.
        │       └── ...                          # puertos, excepciones, etc.
        │
        ├── infrastructure/...                   # capa de infraestructura
        │   ├── adapters/...                     # adaptadores de puertos
        │   ├── auth/...                         # contexto de autenticación (basado en sesiones)
        │   └── ...                              # persistencia, excepciones, etc.
        │
        ├── presentation/...                     # capa de presentación
        │   └── http/                            # interfaz http
        │       ├── auth/...                     # lógica de autenticación web
        │       ├── controllers/...              # controladores y routers
        │       └── errors/...                   # utilidades de manejo de errores
        │
        ├── setup/
        │   ├── ioc/...                          # configuración de inyección de dependencias
        │   ├── config/...                       # ajustes de la aplicación
        │   └── app_factory.py                   # constructor de la aplicación
        │
        └── run.py                               # punto de entrada de la aplicación
```

## Stack Tecnológico

- **Python**: `3.13`
- **Core**: `alembic`, `alembic-postgresql-enum`, `bcrypt`, `dishka`, `fastapi-error-map`, `fastapi`, `orjson`,
  `psycopg3[binary]`, `pyjwt[crypto]`, `sqlalchemy[mypy]`, `uuid-utils`, `uvicorn`, `uvloop`
- **Desarrollo**: `deptry`, `import-linter`, `mypy`, `pre-commit`, `ruff`, `slotscheck`
- **Testing**: `coverage`, `line-profiler`, `pytest`, `pytest-asyncio`

## API

<p align="center">
  <img src="docs/handlers.png" alt="Handlers" />
  <br><em>Figura 14: Handlers</em>
</p>

### General

- `/` (GET): Abierto a **todos**.
    - Redirige a la documentación de Swagger.
- `/api/v1/health` (GET): Abierto a **todos**.
    - Devuelve `200 OK` si la API está activa.

### Cuenta (`/api/v1/account`)

- `/signup` (POST): Abierto a **todos**.
    - Registra un nuevo usuario con validación y verificaciones de unicidad.
    - Las contraseñas se procesan con pepper, salt y se almacenan como hashes.
    - Un usuario con sesión iniciada no puede registrarse hasta que la sesión expire o sea terminada.
- `/login` (POST): Abierto a **todos**.
    - Autentica al usuario registrado, establece un token de acceso JWT con un ID de sesión en cookies y crea una sesión.
    - Un usuario con sesión iniciada no puede iniciar sesión nuevamente hasta que la sesión expire o sea terminada.
    - La autenticación se renueva automáticamente al acceder a rutas protegidas antes de la expiración.
    - Si el JWT es inválido, ha expirado o la sesión ha sido terminada, el usuario pierde la autenticación. [^1]
- `/password` (PUT): Abierto a **usuarios autenticados**.
    - El usuario actual puede cambiar su contraseña.
    - La nueva contraseña debe ser diferente de la contraseña actual.
- `/logout` (DELETE): Abierto a **usuarios autenticados**.
    - Cierra la sesión del usuario eliminando el token de acceso JWT de las cookies y borrando la sesión de la base
      de datos.

### Usuarios (`/api/v1/users`)

- `/` (POST): Abierto a **administradores**.
    - Crea un nuevo usuario, incluyendo administradores, si el nombre de usuario es único.
    - Solo los super administradores pueden crear nuevos administradores.
- `/` (GET): Abierto a **administradores**.
    - Recupera una lista paginada de usuarios existentes con información relevante.
- `/{user_id}/password` (PUT): Abierto a **administradores**.
    - Los administradores pueden establecer contraseñas de usuarios subordinados.
- `/{user_id}/roles/admin` (PUT): Abierto a **super administradores**.
    - Otorga derechos de administrador a un usuario especificado.
    - Los derechos de super administrador no pueden ser modificados.
- `/{user_id}/roles/admin` (DELETE): Abierto a **super administradores**.
    - Revoca derechos de administrador de un usuario especificado.
    - Los derechos de super administrador no pueden ser modificados.
- `/{user_id}/activation` (PUT): Abierto a **administradores**.
    - Restaura un usuario previamente eliminado de forma suave.
    - Solo los super administradores pueden activar a otros administradores.
- `/{user_id}/activation` (DELETE): Abierto a **administradores**.
    - Realiza una eliminación suave de un usuario existente, haciéndolo inactivo.
    - También elimina las sesiones del usuario.
    - Solo los super administradores pueden desactivar a otros administradores.
    - Los super administradores no pueden ser eliminados de forma suave.

> [!NOTE]
> - Los privilegios de super administrador deben otorgarse inicialmente de forma manual (ej., directamente en la base
    de datos), aunque la cuenta de usuario en sí puede crearse a través de la API.

## Configuración

> [!WARNING]
> - Esta parte de la documentación **no** está relacionada con el enfoque de arquitectura.
> - Usa cualquier método de configuración que prefieras.

### Archivos

- **config.toml**: Ajustes principales de la aplicación organizados en secciones
- **export.toml**: Lista los campos a exportar a .env (`export.fields = ["postgres.USER", "postgres.PASSWORD", ...]`)
- **.secrets.toml**: Datos sensibles opcionales (mismo formato que config.toml, se fusiona con la configuración principal)

> [!IMPORTANT]
> - Este proyecto incluye archivos secretos solo con fines de demostración. En un proyecto real, **debes** asegurarte
    de que `.secrets.toml` y todos los archivos `.env` no sean rastreados por el sistema de control de versiones para
    evitar exponer información sensible. Consulta el `.gitignore` de este proyecto para ver un ejemplo de cómo excluir
    correctamente estos archivos sensibles de Git.

### Flujo

En este proyecto utilizo mi propio sistema de configuración basado en archivos TOML como fuente única de verdad.
El sistema genera archivos `.env` para Docker y componentes de infraestructura, mientras que la aplicación lee los
ajustes directamente desde los archivos TOML estructurados. Más detalles disponibles en
https://github.com/ivan-borovets/toml-config-manager

<p align="center">
  <img src="docs/toml_config_manager.svg" alt="Flujo de configuración" />
  <br><em>Figura 15: Flujo de configuración</em>
  <br><small>Aquí, las flechas representan el flujo de uso, <b>no dependencias.</b></small>
</p>

### Entorno Local

1. Configurar el entorno local

* En este proyecto, la configuración local ya está preparada en `config/local/`.
  No necesitas crear nada — ajusta los archivos solo si deseas cambiar los valores por defecto.
* Si deseas ajustar los parámetros, edita los archivos TOML existentes en `config/local/` directamente.
  `.env.local` se generará automáticamente — **no** lo crees ni edites manualmente.
* Docker Compose en este proyecto ya está configurado con `APP_ENV`.
  Solo ten en cuenta esta variable si cambias la configuración:

```yaml
services:
  app:
    # ...
    environment:
      APP_ENV: ${APP_ENV}
```

2. Establecer variable de entorno

```shell
export APP_ENV=local
# export APP_ENV=dev
# export APP_ENV=prod
```

3. Verificar y generar `.env`

```shell
# Probablemente necesitarás Python 3.13 instalado en tu sistema para ejecutar estos comandos.
# La siguiente sección de código proporciona comandos para su instalación rápida.
make env  # debería imprimir APP_ENV=local
make dotenv  # debería indicarte dónde se generó .env.local
```

4. Instalar `uv`

```shell
# sudo apt update
# sudo apt install pipx
# pipx ensurepath
# pipx install uv
# https://docs.astral.sh/uv/getting-started/installation/#shell-autocompletion
# uv python install 3.13  # Para instalar Python
```

5. Configurar el entorno virtual

```shell
uv sync --group dev
source .venv/bin/activate

# Alternativamente,
# uv v
# source .venv/bin/activate  # en Unix
# .venv\Scripts\activate  # en Windows
# uv pip install -e . --group dev
```

No olvides indicarle a tu IDE dónde se encuentra el intérprete.

Instalar hooks de pre-commit:

```shell
# https://pre-commit.com/
pre-commit install
```

6. Lanzamiento

- Para ejecutar solo la base de datos en Docker y usar la aplicación localmente, usa el siguiente comando:

    ```shell
    make up.db
    # make up.db-echo
    ```

- Luego, aplica las migraciones:
    ```shell
    alembic upgrade head
    ```

- Después de aplicar las migraciones, la base de datos está lista, y puedes lanzar la aplicación localmente (ej., a
  través de tu IDE). Recuerda establecer la variable de entorno `APP_ENV` en la configuración de ejecución de tu IDE.

- Para ejecutar vía Docker Compose:

    ```shell
    make up
    # make up.echo
    ```

  En este caso, las migraciones se aplicarán automáticamente al inicio.

7. Apagado

- Para detener los contenedores, usa:
    ```shell
    make down
    ```

### Otros Entornos (dev/prod)

1. Usa las instrucciones sobre el [entorno local](#entorno-local) de arriba

* Pero asegúrate de haber creado una estructura similar en `config/dev` o `config/prod` con los [archivos](#archivos):
    * `config.toml`
    * `.secrets.toml`
    * `export.toml`
    * `docker-compose.yaml` si es necesario
* `.env.dev` o `.env.prod` se generarán después — **no** los crees manualmente

### Añadir Nuevos Entornos

1. Agrega un nuevo valor al enum `ValidEnvs` en `config/toml_config_manager.py` (y posiblemente en los ajustes de tu aplicación)
2. Actualiza el mapeo `ENV_TO_DIR_PATHS` en el mismo archivo (y posiblemente en los ajustes de tu aplicación)
3. Crea el directorio correspondiente en la carpeta `config/`
4. Agrega los [archivos](#archivos) de configuración necesarios

Los directorios de entorno también pueden contener otros archivos específicos del entorno como `docker-compose.yaml`,
que serán utilizados por los comandos del Makefile.

# Recursos Útiles

## Arquitectura por Capas

- [Robert C. Martin. Clean Architecture: A Craftsman's Guide to Software Structure and Design. 2017](https://www.amazon.com/Clean-Architecture-Craftsmans-Software-Structure/dp/0134494164)

- [Alistair Cockburn. Hexagonal Architecture Explained. 2024](https://www.amazon.com/Hexagonal-Architecture-Explained-Alistair-Cockburn-ebook/dp/B0D4JQJ8KD)
  (introducida en 2005)

## Domain-Driven Design

- [Vlad Khononov. Learning Domain-Driven Design: Aligning Software Architecture and Business Strategy. 2021](https://www.amazon.com/Learning-Domain-Driven-Design-Aligning-Architecture/dp/1098100131)

- [Vaughn Vernon. Implementing Domain-Driven Design. 2013](https://www.amazon.com/Implementing-Domain-Driven-Design-Vaughn-Vernon/dp/0321834577)

- [Eric Evans. Domain-Driven Design: Tackling Complexity in the Heart of Software. 2003](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215)

- [Martin Fowler. Patterns of Enterprise Application Architecture. 2002](https://www.amazon.com/Patterns-Enterprise-Application-Architecture-Martin/dp/0321127420)

## Relacionados

- [Vladimir Khorikov. Unit Testing Principles. 2020](https://www.amazon.com/Unit-Testing-Principles-Practices-Patterns/dp/1617296279)

# ⭐ Apoya el Proyecto

Si encuentras este proyecto útil, ¡dale una estrella o compártelo!
Tu apoyo significa mucho.

👉 Échale un vistazo al increíble [fastapi-error-map](https://github.com/ivan-borovets/fastapi-error-map), utilizado
aquí para habilitar el manejo contextual de errores por ruta con generación automática del esquema OpenAPI.

💬 No dudes en abrir issues, hacer preguntas o enviar pull requests.

# Agradecimientos

Me gustaría expresar mi sincera gratitud a las siguientes personas por sus valiosas ideas y apoyo en satisfacer mi
curiosidad a lo largo del desarrollo de este proyecto:
[igoryuha](https://github.com/igoryuha),
[tishka17](https://github.com/tishka17),
[chessenjoyer17](https://github.com/chessenjoyer17),
[PlzTrustMe](https://github.com/PlzTrustMe),
[Krak3nDev](https://github.com/Krak3nDev),
[Ivankirpichnikov](https://github.com/Ivankirpichnikov),
[SamWarden](https://github.com/SamWarden),
[nkhitrov](https://github.com/nkhitrov),
[ApostolFet](https://github.com/ApostolFet),
Lancetnik, Sehat1137, Maclovi.

También aprecio enormemente las valiosas aportaciones compartidas por los participantes del chat de Telegram de la
Comunidad ASGI, a pesar de los frecuentes y animados desafíos de comunicación, así como el chat de Telegram de
⚗️ Reagento (adaptix/dishka) [Telegram](https://t.me/reagento_ru) por sus reflexivas discusiones y generoso
intercambio de conocimientos.

# Pendientes

- [x] configurar CI
- [x] simplificar ajustes
- [x] simplificar anotaciones
- [ ] agregar tests de integración
- [ ] explicar decisiones de diseño

[^1]: La sesión y el token comparten el mismo tiempo de expiración, evitando lecturas a la base de datos si el token ha
expirado. Este esquema de uso de JWT **no está** relacionado con OAuth 2.0 y es una micro-optimización personalizada.
