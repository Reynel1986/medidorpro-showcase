# Detección Automática de Tarjeta con OpenCV (Kotlin)

Este fragmento muestra cómo se detecta una tarjeta de crédito usando OpenCV para calibrar automáticamente la regla de pantalla. La tarjeta de crédito tiene dimensiones estándar internacionales (85.6mm x 54mm), lo que permite usarla como referencia de escala.

> **Nota Técnica:** Se usa la cámara del dispositivo, se convierte el frame a escala de grises, se aplica desenfoque gaussiano, detección de bordes con Canny, y se buscan contornos rectangulares con la proporción correcta de una tarjeta.

```kotlin
// utils/DetectorTarjeta.kt

import org.opencv.android.Utils
import org.opencv.core.*
import org.opencv.imgproc.Imgproc
import android.graphics.Bitmap

object DetectorTarjeta {

    private const val RATIO_TARJETA = 85.6 / 54.0 // Proporción ancho/largo de una tarjeta estándar
    private const val TOLERANCIA_RATIO = 0.15

    /**
     * Detecta una tarjeta en el bitmap y retorna sus esquinas.
     * @return Lista de 4 puntos (esquinas) o null si no se detecta.
     */
    fun detectarTarjeta(bitmap: Bitmap): List<Point>? {
        val mat = Mat()
        Utils.bitmapToMat(bitmap, mat)

        // 1. Convertir a escala de grises
        val gris = Mat()
        Imgproc.cvtColor(mat, gris, Imgproc.COLOR_RGBA2GRAY)

        // 2. Aplicar desenfoque gaussiano para reducir ruido
        val desenfocado = Mat()
        Imgproc.GaussianBlur(gris, desenfocado, Size(5.0, 5.0), 0.0)

        // 3. Detección de bordes con Canny
        val bordes = Mat()
        Imgproc.Canny(desenfocado, bordes, 50.0, 150.0)

        // 4. Dilatar para cerrar huecos en los bordes
        val kernel = Imgproc.getStructuringElement(Imgproc.MORPH_RECT, Size(5.0, 5.0))
        Imgproc.dilate(bordes, bordes, kernel)

        // 5. Buscar contornos
        val contornos = ArrayList<MatOfPoint>()
        val jerarquia = Mat()
        Imgproc.findContours(bordes, contornos, jerarquia, Imgproc.RETR_EXTERNAL, Imgproc.CHAIN_APPROX_SIMPLE)

        // 6. Filtrar contornos por área y proporción
        for (contorno in contornos) {
            val perimetro = Imgproc.arcLength(MatOfPoint2f(*contorno.toArray()), true)
            val aproximado = MatOfPoint2f()
            Imgproc.approxPolyDP(MatOfPoint2f(*contorno.toArray()), aproximado, 0.02 * perimetro, true)

            // Si tiene 4 esquinas, es un candidato
            if (aproximado.total() == 4L) {
                val puntos = aproximado.toList()
                if (esProporcionTarjeta(puntos)) {
                    return puntos
                }
            }
        }
        return null
    }

    /**
     * Verifica si la proporción de los lados coincide con la de una tarjeta.
     */
    private fun esProporcionTarjeta(puntos: List<Point>): Boolean {
        val lados = mutableListOf<Double>()
        for (i in puntos.indices) {
            val p1 = puntos[i]
            val p2 = puntos[(i + 1) % puntos.size]
            lados.add(Math.sqrt(Math.pow(p2.x - p1.x, 2.0) + Math.pow(p2.y - p1.y, 2.0)))
        }
        lados.sortDescending()
        val ratio = lados[0] / lados[1]
        return Math.abs(ratio - RATIO_TARJETA) < TOLERANCIA_RATIO
    }
}
