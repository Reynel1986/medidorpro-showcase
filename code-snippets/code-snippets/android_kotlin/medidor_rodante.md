# Medidor Rodante - Motor de Medición con Giroscopio (Kotlin)

Este fragmento muestra el motor del medidor rodante, que calcula la distancia recorrida usando el giroscopio del dispositivo. A diferencia de otros medidores que requieren calibración por rodado, este utiliza un modelo teórico basado en las dimensiones físicas del celular y la compensación de biseles de pantalla.

> **Nota Técnica:** Se usa `Sensor.TYPE_GYROSCOPE` y se integra la velocidad angular en el tiempo para obtener el ángulo de rotación. Ese ángulo se multiplica por el avance por vuelta (ancho o largo del celular) para obtener la distancia.

```kotlin
// utils/MedidorRodante.kt

class MedidorRodante(
    private val anchoCm: Float,
    private val largoCm: Float,
    private val biselArriba: Float,
    private val biselAbajo: Float,
    private val biselIzquierda: Float,
    private val biselDerecha: Float,
    private val modoRodado: ModoRodado // CORTO o LARGO
) : SensorEventListener {

    private var distanciaTotal = 0f
    private var ultimaMarcaTiempo = 0L
    private var anguloAcumulado = 0f

    /**
     * Avance por vuelta según el modo de rodado.
     * Se compensan los biseles de pantalla para mayor precisión.
     */
    private val avancePorVuelta: Float
        get() = when (modoRodado) {
            ModoRodado.CORTO -> anchoCm - biselIzquierda - biselDerecha
            ModoRodado.LARGO -> largoCm - biselArriba - biselAbajo
        }

    override fun onSensorChanged(event: SensorEvent) {
        if (event.sensor.type != Sensor.TYPE_GYROSCOPE) return

        val tiempoActual = System.nanoTime()
        if (ultimaMarcaTiempo == 0L) {
            ultimaMarcaTiempo = tiempoActual
            return
        }

        val deltaTiempo = (tiempoActual - ultimaMarcaTiempo) / 1_000_000_000f
        ultimaMarcaTiempo = tiempoActual

        // Velocidad angular en rad/s -> grados
        val velocidadAngular = Math.toDegrees(event.values[0].toDouble()).toFloat()

        // Integrar ángulo
        anguloAcumulado += velocidadAngular * deltaTiempo

        // Calcular distancia: (ángulo / 360) * avance por vuelta
        val distanciaParcial = (anguloAcumulado / 360f) * avancePorVuelta
        distanciaTotal += distanciaParcial

        // Reiniciar el ángulo acumulado para el siguiente cálculo
        anguloAcumulado = 0f

        // Notificar al observador
        onDistanciaActualizada?.invoke(distanciaTotal)
    }

    override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {}

    var onDistanciaActualizada: ((Float) -> Unit)? = null
}

enum class ModoRodado { CORTO, LARGO }
