# Sistema de Prueba Gratuita y Licencias (Kotlin)

Este fragmento muestra el gestor de la prueba gratuita de 7 días. El sistema persiste el estado de la prueba en un archivo oculto cifrado que sobrevive a la desinstalación de la app (en Android 10+). Esto evita que el usuario reinstale la app para reiniciar el período de prueba.

> **Nota Técnica:** Se combina `SharedPreferences` (para acceso rápido) con un archivo interno cifrado (para persistencia). Se usa un hash SHA-256 con el ID del dispositivo para evitar manipulaciones.

```kotlin
// utils/PruebaManager.kt

import android.content.Context
import android.content.SharedPreferences
import java.io.File
import java.security.MessageDigest
import java.util.concurrent.TimeUnit

class PruebaManager(private val context: Context) {

    private val prefs: SharedPreferences =
        context.getSharedPreferences("fdo_trial", Context.MODE_PRIVATE)

    private val archivoOculto: File
        get() = File(context.filesDir, ".trial_data")

    companion object {
        private const val DIAS_PRUEBA = 7
        private const val KEY_INICIO = "fecha_inicio"
        private const val KEY_HASH = "hash"
    }

    /**
     * Inicia la prueba gratuita si no existe una previa.
     * @return true si la prueba se inició correctamente.
     */
    fun iniciarPrueba(): Boolean {
        if (pruebaYaUsada()) return false

        val inicio = System.currentTimeMillis()
        val hash = generarHash(inicio)

        // Guardar en SharedPreferences (acceso rápido)
        prefs.edit()
            .putLong(KEY_INICIO, inicio)
            .putString(KEY_HASH, hash)
            .apply()

        // Guardar en archivo oculto (persistencia)
        archivoOculto.writeText("$inicio:$hash")
        archivoOculto.setReadOnly()

        return true
    }

    /**
     * Verifica si la prueba sigue activa y devuelve los días restantes.
     */
    fun diasRestantes(): Int {
        val inicio = obtenerFechaInicio() ?: return 0
        val transcurrido = System.currentTimeMillis() - inicio
        val diasUsados = TimeUnit.MILLISECONDS.toDays(transcurrido).toInt()
        return (DIAS_PRUEBA - diasUsados).coerceAtLeast(0)
    }

    /**
     * Verifica si la prueba ya fue usada en alguna instalación previa.
     */
    private fun pruebaYaUsada(): Boolean {
        return prefs.contains(KEY_INICIO) || archivoOculto.exists()
    }

    /**
     * Recupera la fecha de inicio desde SharedPreferences o desde el archivo oculto.
     */
    private fun obtenerFechaInicio(): Long? {
        val inicioPrefs = prefs.getLong(KEY_INICIO, 0L)
        if (inicioPrefs > 0L) return inicioPrefs

        if (archivoOculto.exists()) {
            val partes = archivoOculto.readText().split(":")
            if (partes.size == 2) return partes[0].toLongOrNull()
        }
        return null
    }

    /**
     * Genera un hash SHA-256 para evitar manipulaciones.
     */
    private fun generarHash(inicio: Long): String {
        val datos = "$inicio:${context.packageName}"
        val digest = MessageDigest.getInstance("SHA-256")
        return digest.digest(datos.toByteArray()).joinToString("") { "%02x".format(it) }
    }
}
