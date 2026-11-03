# <Código> · <Título de su problemática>

> **Hackatón IIA · UADE · 3.4.257.** Este README es la plantilla del grupo. En su fork, reemplacen todo lo que está entre `< >` y borren esta nota. Las bases están en `BASES.md` y su ficha en `problematicas/`.

**Integrantes:** <Nikolas Iglesias> · <Ignacio Flores>
**IA declarada:** <chat gpt>
**Pista:** <Neurona / Reglas / Agente>
**Publicación en LinkedIn:** <link>

---

## 1. Formulación y PEAS

**La pregunta.** Para cada caso, ¿el jugador abandona el juego si no se le baja la dificultad? SÍ significa que el jugador frustrado abandona el título; NO significa que continúa jugando.

| PEAS | Su sistema |
|---|---|
| Medida de rendimiento | Minimizar abandonos y preservar el desafío (balance entre Sensibilidad y Especificidad). |
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
*   *Usa:* `Muertes seguidas` (peso $+2$, debido a que indica frustración acumulada), `Vida que le dejó al jefe` (peso $+1$, ya que indica cercanía al triunfo) y `Horas jugadas` (peso $-1$, esto mide el compromiso y tolerancia al desafío del jugador).
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

**Entrenamiento contra prueba.** No hay pérdida de rendimiento (se mantiene 100% de exactitud en ambos conjuntos), lo que demuestra que el perceptrón generaliza correctamente sin sobreajuste (*overfitting*).

## 4. El error más caro

En este dominio, el error más caro es el **Falso Negativo (decir NO cuando era SÍ)**, porque significa no ayudar a un jugador frustrado que termina abandonando el juego para siempre (pérdida directa de usuario y retención). El **Falso Positivo** solo le quita algo de desafío a una pelea, pero retiene al jugador. Para evitar los falsos negativos ajustamos el sesgo a $-10$, priorizando una Sensibilidad del 100%.

## 5. ¿Comparado contra qué?

**Línea base trivial.** Un sistema tonto que responda siempre "SÍ" a todos los jugadores acertaría en 7 de 14 en entrenamiento (50%) y alteraría innecesariamente la dificultad a todos. Nuestro perceptrón supera ampliamente esta línea base con 100% de exactitud.

| Checklist (Clase 6) | Respuesta |
|---|---|
| ¿Qué datos? | Variables de telemetría de sesión (muertes, vida al jefe, horas jugadas). |
| ¿Qué métrica? | Exactitud, Sensibilidad y Precisión. |
| ¿Comparado contra qué? | Clasificador trivial de clase mayoritaria. |
| ¿Quién lo midió? | El sistema evaluador y validación cruzada train/test. |

## 6. Riesgo: el camino completo

Datos de telemetría recolectados del cliente $\rightarrow$ Modelo de perceptrón de dificultad $\rightarrow$ Decisión automatizada de bajar o mantener jefes $\rightarrow$ Consecuencia: Mayor retención de jugadores y evitar el abandono prematuro por picos de frustración injustos.

## 7. Bitácora

**Uso de la IA**

| # | Qué le pedimos | Qué aceptamos | Qué rechazamos y por qué |
|---|---|---|---|
| 1 | Diseñar los pesos del perceptrón lineal para T19 | La combinación de pesos $w_1=2, w_2=1, w_3=-1, b=-10$ | Ninguna, validada matemáticamente con los 20 casos del CSV. |

**Corridas contra el evaluador**

| Corrida | Nota | Agujero que encontró | Qué cambiamos |
|---|---|---|---|
| 1 | 10/10 | Ninguno (todas las métricas cierran perfectamente) | Ninguno. |
