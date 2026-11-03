# T19 · ¿Bajamos la dificultad de este jefe?

> **Hackatón IIA · UADE · 3.4.257.**

**Integrantes:** <Nikolas Iglesias> · <Ignacio Flores>
**IA declarada:** <chat gpt>
**Pista:** <Neurona>
**Publicación en LinkedIn:** <link>

---

## 1. Formulación y PEAS

**La pregunta.** Para cada caso, ¿el jugador abandona el juego si no se le baja la dificultad? SÍ significa que el jugador frustrado abandona el título; NO significa que continúa jugando.

| PEAS | Su sistema |
|---|---|
| Medida de rendimiento | Maximizar la Sensibilidad (Recall) y la Precisión para retener jugadores sin alterar el desafío injustamente. |
| Entorno | Sistema de dificultad adaptativa en tiempo real analizando la sesión del jugador contra el jefe final. |
| Actuadores | Disparar o no el evento de reducción dinámica de la dificultad del jefe en el próximo reintento. |
| Sensores | Muertes seguidas, porcentaje de vida dejado al jefe en el mejor intento y horas totales jugadas. |

**Objetivo y métrica.** El objetivo real del estudio es retener al usuario evitando que abandone por frustración sin arruinarle el desafío a quien está por superarlo (mantener el flujo de juego). No es lo mismo que la métrica ciega del estudio (porcentaje global de victorias), ya que si optimizáramos ciegamente el éxito global haciendo trampa bajándole la vida a todos, destruiríamos el valor del juego. Medimos el éxito con Sensibilidad y Precisión para evitar falsos negativos (jugadores perdidos).

## 2. El sistema

**Cómo decide.** Neurona (Perceptrón con pesos y sesgo):
$$z = 2 \cdot (\text{Muertes seguidas}) + 1 \cdot (\text{Vida restante \%}) - 1 \cdot (\text{Horas jugadas}) - 10$$
*   Si $z \ge 0 \rightarrow$ **SÍ** (Bajar la dificultad).
*   Si $z < 0 \rightarrow$ **NO** (Mantener la dificultad).

**Variables que usa y por qué.**
*   *Usa:* `Muertes seguidas` (peso $+2$, indica frustración acumulada), `Vida que le dejó al jefe` (peso $+1$, indica cercanía al triunfo) y `Horas jugadas` (peso $-1$, mide el compromiso y tolerancia al desafío del jugador).
*   *Descarta:* `Jugadores que derrotan al jefe (% global)` porque es un indicador estadístico macro posterior que no forma parte de la decisión en tiempo real del usuario (evita *data leakage*).

**Cómo se corre en vivo.** 
Se evalúa la fórmula lineal haciendo la suma ponderada de las tres variables del caso provisto y comparándola contra el umbral del sesgo.

**Dos casos de ejemplo.**

| Caso inventado | Valores | Salida del sistema |
|---|---|---|
| A (Caso 3) | Muertes=11, Vida=38%, Horas=42 | SÍ ($z = 8 \ge 0$) |
| B (Caso 1) | Muertes=2, Vida=69%, Horas=107 | NO ($z = -44 < 0$) |

## 3. Medición

**Entrenamiento (casos 1–14)**

| | Predice SÍ | Predice NO |
|---|---|---|
| **Real SÍ** | VP = 7 | FN = 0 |
| **Real NO** | FP = 0 | VN = 7 |

**Prueba (casos 15–20)**

| | Predice SÍ | Predice NO |
|---|---|---|
| **Real SÍ** | VP = 4 | FN = 0 |
| **Real NO** | FP = 0 | VN = 2 |

| Métrica | Entrenamiento | Prueba |
|---|---|---|
| Exactitud | 100% (14/14) | 100% (6/6) |
| Precisión | 100% (7/7) | 100% (4/4) |
| Sensibilidad | 100% (7/7) | 100% (4/4) |
| Especificidad | 100% (7/7) | 100% (2/2) |

**Entrenamiento contra prueba (Independencia del modelo).** Los pesos y el sesgo ($w_1=2, w_2=1, w_3=-1, b=-10$) se sintonizaron **únicamente** utilizando los 14 casos de entrenamiento. El conjunto de prueba (casos 15 a 20) permaneció completamente oculto durante el diseño, por lo que el 100% obtenido en prueba es una medición independiente y válida de generalización.

## 4. El error más caro

En este dominio, el error más caro es el **Falso Negativo (decir NO cuando era SÍ)**, porque significa no ayudar a un jugador frustrado que termina abandonando el juego para siempre (pérdida directa de usuario y retención). El **Falso Positivo** solo le quita algo de desafío a una pelea, pero retiene al jugador. Para evitar los falsos negativos ajustamos el sesgo a $-10$, priorizando una Sensibilidad del 100%.

## 5. ¿Comparado contra qué?

**Línea base trivial.** 
* En **entrenamiento**, un sistema tonto que responde siempre SÍ acierta en 7 de 14 (50%).
* En **prueba**, un sistema que responde siempre SÍ acierta en 4 de 6 casos (66.7% de exactitud), ya que hay 4 verdaderos positivos reales (casos 15, 18, 19, 20) y 2 negativos (16, 17). Nuestro perceptrón supera esta línea base alcanzando el 100% de exactitud tanto en entrenamiento como en prueba.

| Checklist (Clase 6) | Respuesta |
|---|---|
| ¿Qué datos? | Variables de telemetría de sesión (muertes, vida al jefe, horas jugadas). |
| ¿Qué métrica? | Exactitud, Sensibilidad y Precisión. |
| ¿Comparado contra qué? | Clasificador trivial de clase mayoritaria (responder siempre SÍ). |
| ¿Quién lo midió? | El equipo de desarrollo mediante evaluación separada train/test. |

## 6. Riesgo: el camino completo

**Datos** (telemetría de muertes y tiempo) $\rightarrow$ **Modelo** (perceptrón lineal de dificultad adaptativa) $\rightarrow$ **Decisión** (activar o no la reducción de vida del jefe) $\rightarrow$ **Consecuencia**. 
* **Riesgo concreto:** Un falso positivo en jugadores hiper-competitivos que disfrutan la máxima dificultad (por ejemplo, alterar el jefe a alguien que quería pasarlo por mérito propio) puede generar frustración inversa, haciendo que el usuario sienta que el juego lo infantiloide o subestima su habilidad.

## 7. Bitácora

**Uso de la IA**

| # | Qué le pedimos | Qué aceptamos | Qué rechazamos y por qué |
|---|---|---|---|
| 1 | 
Le pedimos: revision del PEAS para buscar mejoras. Proponer los pesos iniciales del perceptrón. Armar el sistema y ayudarnos con las mediciones | 
Aceptamos: La estructura base de las variables de telemetría | 
Rechazamos usar la columna de éxito global sugerida inicialmente por la IA para evitar *data leakage* (variable posterior al evento). Ademas de rechazar las opciones que indico como posibles en lo que respecta a el uso de reglas y agentes |

**Corridas contra el evaluador**

| Corrida | Nota | Agujero que encontró | Qué cambiamos |
|---|---|---|---|
| 1 | 7/10 | Desalineación de métricas (PEAS), falta de prueba en línea base de test, definición de independencia en prueba y ausencia de riesgo concreto. | Unificamos las métricas en PEAS, aclaramos que el test estuvo oculto al entrenar, calculamos la línea base en prueba (66.7%), incorporamos el riesgo de frustración inversa y registramos los rechazos de IA en la bitácora. |
