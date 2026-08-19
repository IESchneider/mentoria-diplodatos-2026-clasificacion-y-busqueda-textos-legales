# Normalización de `materia`

Documento de criterio para la sección 5 de [`tp1.ipynb`](tp1.ipynb). Explica qué se cambió en la
normalización de la variable `materia`, con qué evidencia y con qué fundamento jurídico.

---

## 1. Contexto

`materia` es una de las cuatro variables candidatas a etiqueta del clasificador, junto con
`descriptores`, `tipo-tribunal` y `jurisdiccion`. Proviene del catálogo de sumarios del SAIJ
(Sistema Argentino de Información Jurídica) y expresa la rama del derecho a la que pertenece
cada sumario.

| indicador | valor |
|---|---|
| registros del corpus | 874.845 |
| registros con `materia` | 483.305 (55,24 %) |
| registros sin `materia` | 391.540 (44,76 %) |
| formas distintas en el valor crudo | 2.636 |

Las 2.636 formas no son 2.636 materias. Son unas pocas ramas del derecho escritas de muchas
maneras distintas por muchos catalogadores a lo largo de décadas.

---

## 2. El problema de la versión anterior

La normalización original aplicaba cinco reglas de string:

```python
partes  = re.split(r"[-,/]", sin_acentos(valor).upper())
limpias = {re.sub(r"\s+", " ", p).strip() for p in partes}
return sorted(p for p in limpias if p)
```

Es decir: mayúsculas, sin tildes, corte por `-`, `,` y `/`, colapso de espacios y deduplicación.
El resultado bajaba de 2.636 formas a 156 clases. Funcionaba, pero tenía tres defectos.

### 2.1 Separadores no contemplados

El corte cubría `-`, `,` y `/`, pero los catalogadores también usaron punto, guion bajo, dos
puntos y la conjunción `Y`. Casos que sobrevivían sin partir:

| valor | apariciones | problema |
|---|---|---|
| `CIVIL Y COMERCIAL` | 47 | la conjunción no se cortaba, pero `CIVIL - COMERCIAL` sí |
| `CONSTITUCIONAL. PROCESAL` | 55 | el punto no se cortaba |
| `CONSTITUCIONAL. CIVIL. LABORAL` | 1 | idem |
| `CIVIL_PROCESAL` | 1 | el guion bajo no se cortaba |
| `CIVIL:` | 44 | los dos puntos colgantes quedaban pegados al término |
| `DERECHO TRIBUTARIO` | 1 | el prefijo `DERECHO ` generaba una clase paralela a `TRIBUTARIO` |

Cada uno de estos genera una clase espuria que compite con la clase legítima.

### 2.2 Tratamiento inconsistente del derecho procesal

Este es el defecto grave. La normalización anterior producía dos resultados distintos para el
mismo concepto, según cómo lo hubiera tipeado el catalogador:

| escritura | apariciones | resultado anterior |
|---|---|---|
| `PROCESAL PENAL` (con espacio) | ≈ 2.801 | **una** etiqueta: `PROCESAL PENAL` |
| `PROCESAL-PENAL` | 7.559 | **dos** etiquetas: `PROCESAL` + `PENAL` |
| `PENAL-PROCESAL` | 8.926 | **dos** etiquetas: `PROCESAL` + `PENAL` |
| `PROCESAL - PENAL` | 4.487 | **dos** etiquetas: `PROCESAL` + `PENAL` |

Unos 21.000 registros iban a un conjunto de etiquetas y otros 2.800 a un conjunto distinto,
describiendo lo mismo. Para un clasificador esto es ruido puro en el target: la misma señal de
entrada apunta a dos salidas diferentes.

### 2.3 El criterio se deducía de los separadores, no de la taxonomía

Ninguna regla de string puede resolver el punto 2.2, porque la ambigüedad no es tipográfica sino
taxonómica. La pregunta de fondo no es «¿parto por guion?» sino «¿cuáles son las ramas válidas
del derecho argentino?». Mientras el criterio se dedujera de los separadores, el problema seguía
siendo irresoluble.

---

## 3. Evidencia: el guion es un separador de valores

Antes de rediseñar hay que confirmar el supuesto de base. La evidencia es del propio dataset y es
concluyente.

**El orden se invierte.** Si `CIVIL-COMERCIAL` fuera el nombre de una rama compuesta, el orden
sería fijo. No lo es:

| par | apariciones |
|---|---|
| `CIVIL - COMERCIAL` | 39.082 |
| `CIVIL-COMERCIAL` | 19.792 |
| `COMERCIAL-CIVIL` | 1.115 |
| `PENAL-PROCESAL` | 8.926 |
| `PROCESAL-PENAL` | 7.559 |

La alternancia de orden solo ocurre cuando el campo almacena un **conjunto**, no un nombre.

**Hay cadenas largas.** En la cola aparecen valores como:

```
ADUANERO-ECONOMIA-FINANZAS-PROCESAL-TRIBUTARIO
ADMINISTRATIVO-CIVIL-PROCESAL-SALUD PUBLICA
INTERNACIONAL PUBLICO-DERECHOS HUMANOS-PENAL
```

Ninguna rama del derecho se denomina con cinco términos unidos por guion. Son listas.

**Conclusión:** `materia` es un campo multivaluado serializado con separador, y las ramas del
derecho son los átomos. Partir por guion es correcto.

> **Advertencia para revisar en otros campos.** En la jerga forense argentina el fuero se escribe
> «contencioso-administrativo», con guion como parte del nombre. En esta columna la rama aparece
> como `ADMINISTRATIVO` a secas, de modo que la regla no lo afecta. Si se normalizan otras
> columnas, conviene verificarlo explícitamente antes de aplicar el mismo corte.

---

## 4. El cambio de enfoque: vocabulario controlado

En lugar de deducir la taxonomía a partir de reglas de string, se **declara** el vocabulario de
ramas del derecho argentino y se mapea cada fragmento contra él. Las expresiones regulares dejan
de ser el criterio y pasan a ser un paso mecánico previo.

Esto invierte la carga de la prueba: antes, todo lo que las reglas no contemplaban se convertía
silenciosamente en una clase nueva; ahora, todo lo que no está en el vocabulario cae en `OTRAS` y
queda registrado para auditoría.

### 4.1 Pipeline

| paso | operación | ejemplo |
|---|---|---|
| 1. limpieza | `upper()`, `sin_acentos()`, colapso de espacios, quita el prefijo `DERECHO `, quita puntuación colgante | `Derecho Tributario` → `TRIBUTARIO`; `CIVIL:` → `CIVIL` |
| 2. separación | corta por `-`, `,`, `/`, `.`, `_`, `:`, `;` y la conjunción ` Y ` | `CONSTITUCIONAL. CIVIL. LABORAL` → 3 fragmentos |
| 3. canonización | mapea cada fragmento al vocabulario; lo no reconocido → `OTRAS` | `MEDIO AMBIENTE` → `AMBIENTAL` |
| 4. recomposición | colapsa `PROCESAL` + una sustantiva adjetivable en su subrama | `PENAL-PROCESAL` → `PROCESAL PENAL` |

### 4.2 Qué no se separa

**No se corta por espacio.** Existen ramas legítimas de dos palabras que quedarían destruidas:

```
SEGURIDAD SOCIAL        DERECHOS HUMANOS        RECURSOS NATURALES
SALUD PUBLICA           MEDIO AMBIENTE          INTERNACIONAL PUBLICO
```

Se prefiere arrastrar algo de ruido antes que inventar clases inexistentes.

Detalle de implementación: el prefijo `DERECHO ` se elimina con `^DERECHO\s+`, que exige un
espacio inmediatamente después. `DERECHOS HUMANOS` no se ve afectado porque lleva una `S`
intercalada.

### 4.3 Vocabulario

Ramas sustantivas declaradas (30), tomadas de los valores efectivamente presentes en el corpus:

```
ADMINISTRATIVO   ADUANERO        AERONAUTICO      AGRARIO         AGUAS
AMBIENTAL        BANCARIO        CIVIL            COMERCIAL       CONSTITUCIONAL
CONTRAVENCIONAL  CULTURA         DERECHOS HUMANOS ECONOMIA        EDUCACION
ELECTORAL        FINANZAS        INFORMATICO      INTERNACIONAL   INTERNACIONAL PRIVADO
INTERNACIONAL PUBLICO            LABORAL          MILITAR         NAVEGACION
PENAL            POLITICO        RECURSOS NATURALES               SALUD PUBLICA
SEGURIDAD SOCIAL TRIBUTARIO
```

Más la rama adjetiva `PROCESAL` y sus siete subramas (`PROCESAL PENAL`, `PROCESAL CIVIL`, etc.).

Alias declarados: `MEDIO AMBIENTE` → `AMBIENTAL`.

Nota sobre `INTERNACIONAL`: se conserva como término propio (71 apariciones) además de
`INTERNACIONAL PUBLICO` (941) e `INTERNACIONAL PRIVADO` (720). El derecho internacional público y
el privado son ramas distintas, y el término desnudo no permite decidir cuál de las dos quiso
indicar el catalogador. Asignarlo a una sería inventar información.

---

## 5. Fundamento jurídico de la recomposición procesal

El paso 4 es el que exige conocimiento del derecho argentino, no de pandas.

### 5.1 La distinción sustantivo / adjetivo es constitucional

El artículo 75 inciso 12 de la Constitución Nacional faculta al Congreso a dictar los **códigos de
fondo** —Civil, Comercial, Penal, de Minería, del Trabajo y Seguridad Social— «sin que tales
códigos alteren las jurisdicciones locales». Las provincias, en virtud del artículo 121, conservan
la potestad de dictar sus propios **códigos de procedimiento**.

De ahí que el derecho argentino separe estructuralmente:

- **derecho sustantivo** (o de fondo): define derechos y obligaciones — civil, comercial, penal, laboral;
- **derecho adjetivo** (o procesal): regula cómo se hacen valer ante un tribunal.

Esa división no es una convención de catalogación: es la razón por la cual `PROCESAL` aparece en
el dataset como un eje independiente que se combina con las ramas de fondo.

### 5.2 Las subramas procesales tienen autonomía

Cada subrama procesal se apoya en un cuerpo normativo propio, distinto del código de fondo
correspondiente:

| subrama | cuerpo normativo | distinto de |
|---|---|---|
| `PROCESAL PENAL` | CPPN (Ley 23.984) y CPPF (Ley 27.063) | Código Penal (Ley 11.179) |
| `PROCESAL CIVIL` / `PROCESAL COMERCIAL` | CPCCN (Ley 17.454, t.o. Ley 22.434) | CCyCN (Ley 26.994) |
| `PROCESAL LABORAL` | Ley 18.345, de organización y procedimiento de la justicia nacional del trabajo | LCT (Ley 20.744) |
| `PROCESAL CONSTITUCIONAL` | amparo (Ley 16.986), habeas corpus (Ley 23.098), habeas data (Ley 25.326) | Constitución Nacional |

El derecho procesal penal no es la intersección de «procesal» y «penal»: es una disciplina con
objeto, principios y código propios. Por eso `PROCESAL` junto a **exactamente una** rama
adjetivable colapsa en la subrama correspondiente.

Ramas adjetivables declaradas: `PENAL`, `CIVIL`, `COMERCIAL`, `LABORAL`, `CONSTITUCIONAL`,
`ADMINISTRATIVO`, `TRIBUTARIO`.

### 5.3 La regla declara su propia ignorancia

Cuando hay dos o más ramas adjetivables el valor original es genuinamente ambiguo:

```
CONSTITUCIONAL-PROCESAL-PENAL   (1.016 apariciones)
```

¿Es «procesal penal» + «constitucional», o «procesal constitucional» + «penal»? No hay forma de
saberlo a partir del dato. En ese caso la regla **no recompone**: deja `PROCESAL` como género y
mantiene las sustantivas por separado. Es preferible una etiqueta menos específica pero correcta
antes que una específica elegida al azar.

### 5.4 Efecto de la regla

Los cuatro grupos siguientes convergen ahora a una sola etiqueta cada uno (apariciones aproximadas
en el valor crudo):

| grupo | variantes que absorbe | total aprox. |
|---|---|---|
| `PROCESAL PENAL` | `PROCESAL-PENAL`, `PENAL-PROCESAL`, `PROCESAL - PENAL`, `PROCESAL PENAL` | ≈ 23.400 |
| `PROCESAL CIVIL` | `PROCESAL-CIVIL`, `CIVIL-PROCESAL`, `PROCESAL - CIVIL`, `PROCESAL CIVIL` | ≈ 14.700 |
| `PROCESAL CONSTITUCIONAL` | `CONSTITUCIONAL-PROCESAL`, `CONSTITUCIONAL - PROCESAL`, `PROCESAL-CONSTITUCIONAL`, `PROCESAL - CONSTITUCIONAL` | ≈ 13.900 |
| `PROCESAL LABORAL` | `PROCESAL-LABORAL`, `LABORAL-PROCESAL`, `PROCESAL LABORAL` | ≈ 3.700 |

---

## 6. Decisión: civil y comercial se mantienen separados

Esta decisión **va contra la letra del derecho vigente** y por eso se documenta explícitamente.

Desde la Ley 26.994, en vigor el 1 de agosto de 2015 (fecha adelantada por la Ley 27.077), rige el
**Código Civil y Comercial de la Nación**, que unificó en un solo cuerpo el Código Civil de Vélez
Sarsfield (Ley 340) y el Código de Comercio. En términos de derecho positivo actual, «civil» y
«comercial» son hoy una sola rama.

Aun así se mantienen como clases distintas, por dos razones:

1. **Cobertura temporal del corpus.** El catálogo de sumarios abarca décadas anteriores a la
   unificación, período en el cual ambos códigos eran cuerpos normativos separados, con tribunales
   y doctrina propios. La distinción que hizo el catalogador correspondía a una diferencia real al
   momento de catalogar.

2. **El dataset las distingue con volumen propio.** No es un residuo marginal:

   | valor | apariciones |
   |---|---|
   | `CIVIL` puro | 45.232 |
   | `COMERCIAL` puro | 19.704 |
   | `CIVIL - COMERCIAL` / `CIVIL-COMERCIAL` / `COMERCIAL-CIVIL` | ≈ 59.989 |

**El argumento decisivo es la reversibilidad.** Fusionarlas destruye una distinción que después no
se puede recuperar. Mantenerlas separadas permite unirlas más adelante con una línea de código si
el modelo lo pide. Ante la duda, la normalización debe avanzar siempre hacia el lado reversible.

Consecuencia práctica: `CIVIL Y COMERCIAL` se parte en dos etiquetas, igual que `CIVIL - COMERCIAL`.

---

## 7. Consecuencia para el modelado

La normalización **cambia el tipo de problema**, y esto debe tenerse presente al elegir la variable
objetivo:

- el valor **crudo** es multiclase: una etiqueta por sumario;
- el valor **normalizado** es multietiqueta: al partir las materias compuestas, un sumario pasa a
  tener varias ramas.

Por eso el total de etiquetas resulta mayor que la cantidad de sumarios, y los porcentajes de la
tabla de desbalance se leen **sobre etiquetas, no sobre registros**.

Efectos esperables sobre la distribución respecto de la versión anterior:

- el número de clases baja de 156 a aproximadamente 40;
- `PROCESAL` se reduce de forma marcada (partía de 147.269 etiquetas) porque sus apariciones
  acopladas migran a las subramas;
- desaparecen las clases espurias generadas por separadores no contemplados;
- aparece la clase `OTRAS`, cuyo tamaño es en sí mismo una métrica de calidad del vocabulario.

---

## 8. Auditoría

El paso 3 no descarta nada en silencio. La variable `fuera_de_vocabulario` recoge todos los
términos que no matchearon, con su frecuencia:

```python
fuera_de_vocabulario = (
    materia_cruda.map(lambda v: sorted(desarmar_materia(v) - VOCABULARIO))
    .explode()
    .dropna()
    .value_counts()
)
```

**Cómo leerla.** Un término con volumen significativo no es basura tipográfica: es una rama que
faltó declarar en `RAMAS_SUSTANTIVAS` y debe incorporarse. Términos con una o dos apariciones son,
en general, errores de carga y su lugar natural es `OTRAS`.

Esta es la diferencia práctica entre el enfoque anterior y el actual: antes, lo no contemplado se
convertía en una clase nueva sin que nadie se enterara; ahora queda a la vista y obliga a una
decisión explícita.

---

## 9. Limitaciones conocidas

- **La celda no fue ejecutada al momento de escribir este documento.** Las cifras posteriores a la
  normalización marcadas como aproximadas son estimaciones calculadas a mano sobre los conteos del
  valor crudo. Deben confirmarse corriendo la sección 5.
- **`OTRAS` agrupa causas heterogéneas**: errores de tipeo, ramas no declaradas y valores que no
  son materias. Conviene revisar `fuera_de_vocabulario` antes de decidir si `OTRAS` entra al
  modelo o se descarta.
- **Duplicaciones dentro de un mismo fragmento no se colapsan.** `PROCESAL PROCESAL` (1 aparición)
  no matchea el vocabulario y cae en `OTRAS`. Con ese volumen no se justifica una regla dedicada.
- **`INTERNACIONAL` queda sin resolver** entre público y privado, por decisión deliberada.
- **El vocabulario es específico de este corpus.** Está construido sobre los valores observados en
  `materia`; aplicarlo a otra fuente exige revisar la auditoría de nuevo.

---

## 10. Referencias normativas citadas

| norma | contenido |
|---|---|
| Constitución Nacional, art. 75 inc. 12 y art. 121 | reparto de competencias entre códigos de fondo y legislación procesal local |
| Ley 26.994 | Código Civil y Comercial de la Nación (vigente 1/8/2015) |
| Ley 340 | Código Civil de Vélez Sarsfield (derogado) |
| Ley 11.179 | Código Penal |
| Ley 23.984 | Código Procesal Penal de la Nación |
| Ley 27.063 | Código Procesal Penal Federal |
| Ley 17.454 (t.o. Ley 22.434) | Código Procesal Civil y Comercial de la Nación |
| Ley 18.345 | organización y procedimiento de la justicia nacional del trabajo |
| Ley 20.744 | Ley de Contrato de Trabajo |
| Ley 16.986 | acción de amparo |
| Ley 23.098 | habeas corpus |
| Ley 25.326 | protección de datos personales (habeas data) |
