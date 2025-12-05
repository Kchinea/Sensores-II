# Sensores-II-Kyliam-ChineaSalcedo

En esta práctica realizamos dos ejercicios para comprender el funcionamiento de los sensores en unity, es imporatnte saber los tipos de sensores, y que no todos estan disponibles en todos los dispositivos, cosa que se puede comprobar con el primer ejercicio de la práctica.

## Ejercicio 1

### Enunciado 
Crear una aplicación en Unity que muestre en la UI los valores de todos los sensores disponibles en tu móvil. Incluir en el Readme una medida de los valores en el laboratorio y otra en el jardin de la ESIT.

### Codigo

```csharp

using UnityEngine;
using UnityEngine.UI;
using UnityEngine.InputSystem; 
using System.Collections;

public class ShowSensors : MonoBehaviour
{
    public Text textUI;

    IEnumerator Start()
    {
        if (Accelerometer.current != null) 
            InputSystem.EnableDevice(Accelerometer.current);

        if (Gyroscope.current != null) 
            InputSystem.EnableDevice(Gyroscope.current);

        Input.compass.enabled = true;

        if (!Input.location.isEnabledByUser)
        {
            Debug.Log("GPS desactivado por el usuario");
        }
        else 
        {
            Input.location.Start(1f, 1f);
            
            int tiempoEspera = 20;
            while (Input.location.status == LocationServiceStatus.Initializing && tiempoEspera > 0)
            {
                yield return new WaitForSeconds(1f);
                tiempoEspera--;
            }

            if (tiempoEspera < 1 || Input.location.status == LocationServiceStatus.Failed)
            {
                Debug.Log("No se pudo iniciar el GPS");
            }
        }

        while (true)
        {
            UpdateInfo();
            yield return new WaitForSeconds(0.2f);
        }
    }

    void UpdateInfo()
    {
        string info = "";

        if (Accelerometer.current != null)
        {
            info += "Acelerómetro (InputSystem): " + Accelerometer.current.acceleration.ReadValue().ToString() + "\n";
        }
        else
        {
            info += "Acelerómetro (Legacy): " + Input.acceleration.ToString() + "\n";
        }

        if (Gyroscope.current != null)
        {
            info += "Giroscopio (InputSystem): " + Gyroscope.current.angularVelocity.ReadValue().ToString() + "\n";
        }
        else
        {
            info += "Giroscopio: No disponible\n";
        }

        if (Input.compass.enabled)
        {
            info += "Brújula (Heading): " + Input.compass.trueHeading.ToString("F2") + "\n";
        }

        if (Input.location.status == LocationServiceStatus.Running)
        {
            var d = Input.location.lastData;
            info += "GPS Lat: " + d.latitude.ToString("F6") + "\n";
            info += "GPS Lon: " + d.longitude.ToString("F6") + "\n";
            info += "GPS Alt: " + d.altitude.ToString("F2") + "\n";
        }
        else
        {
            info += "GPS Estado: " + Input.location.status.ToString() + "\n";
        }

        textUI.text = info;
    }

    void OnDisable()
    {
        Input.location.Stop();
    }
}
```

### Explicacion

Este script tiene como objetivo principal la depuración y visualización de datos de sensores en tiempo real.
Inicialización: Se utiliza una corrutina para gestionar la activación asíncrona del servicio de localización (GPS), estableciendo un timeout de 20 segundos para evitar bloqueos si el GPS no responde. También se activan el acelerómetro y el giroscopio mediante el Input System y la brújula mediante el sistema Legacy.
Lectura: Se concatenan los valores de los sensores en una cadena de texto. Se ha implementado compatibilidad dual: intenta leer del nuevo Input System y, si no detecta el sensor, recurre al sistema clásico (Legacy) o indica que no está disponible.

### Prueba
**Actualmente debido a que no tengo un telefono android no he podido comprobar en su totalidad que la implementacion funciona, epero poder hacerlo en la hora de practicas**
## Ejercicio 2

### Enunciado 
Crear una apk que oriente alguno de los guerreros de la práctica mirando siempre hacia el norte, avance con una aceleración proporcional a la del dispositivo y lo pare cuando el dispositivo esté fuera de un rango de latitud, longitud dado. El acelerómetro nos dará la velocidad del movimiento. A lo largo del eje z (hacia adelante y hacia atrás), se produce el movimiento inclinando el dispositivo hacia adelante y hacia atrás. Sin embargo, necesitamos invertir el valor z porque la orientación del sistema de coordenadas corresponde con el punto de vista del dispositivo. Queremos que la rotación final coincida con la orientación cuando mantenemos el dispositivo en la posición Horizontal Izquierda. Esto ocurre cuando la izquierda en la posición vertical ahora es la parte inferior. Aplicar las rotaciones con interpolación  Slerp en un quaternion.


### Codigo

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

public class WarriorController : MonoBehaviour
{
    public Transform warrior;
    public Rigidbody warriorRb;
    public float movementFactor = 10f;
    public float rotationSpeed = 2f;

    private float startLat;
    private float startLon;
    private bool gpsReady = false;

    void Start()
    {
        Input.compass.enabled = true;
        Input.location.Start(1f, 0.1f);
        
        if (Accelerometer.current != null)
            InputSystem.EnableDevice(Accelerometer.current);
    }

    void FixedUpdate()
    {
        if (warrior == null || warriorRb == null) return;

        if (Input.location.status == LocationServiceStatus.Running)
        {
            float lat = Input.location.lastData.latitude;
            float lon = Input.location.lastData.longitude;

            if (!gpsReady)
            {
                startLat = lat;
                startLon = lon;
                gpsReady = true;
            }

            float rango = 0.00009f;

            bool fueraDeRango = 
                lat < (startLat - rango) || lat > (startLat + rango) ||
                lon < (startLon - rango) || lon > (startLon + rango);

            if (fueraDeRango)
            {
                warriorRb.velocity = Vector3.zero;
                return;
            }
        }

        float norte = Input.compass.trueHeading;
        Quaternion objetivoRot = Quaternion.Euler(0, norte, 0);
        warrior.rotation = Quaternion.Slerp(warrior.rotation, objetivoRot, Time.fixedDeltaTime * rotationSpeed);

        if (Accelerometer.current != null)
        {
            float zInclinacion = Accelerometer.current.acceleration.ReadValue().z;
            float aceleracion = -zInclinacion; 

            if (Mathf.Abs(aceleracion) > 0.1f)
            {
                Vector3 desplazamiento = warrior.forward * aceleracion * movementFactor * Time.fixedDeltaTime;
                warriorRb.MovePosition(warriorRb.position + desplazamiento);
            }
        }
    }
    
    void OnDisable()
    {
        Input.location.Stop();
    }
}
```
### Explicacion
Este script implementa un controlador de un guerrero que llevamos usando en las practicas anteiores basado completamente en sensores físicos:
1. Geolocalización : Al iniciarse el GPS y recibir la primera lectura válida, se guarda la posición inicial (variables startLat y startLon). En cada frame físico, se comprueba si la posición actual difiere más de 0.00009 grados (lo que seria aproximadamente 10 metros) de la inicial. Si se sale del rango, la velocidad del Rigidbody se anula, impidiendo el movimiento.
2. Orientación: Se utiliza la brújula (un Input.compass.trueHeading) para obtener el Norte magnético real. Se aplica esta rotación al eje Y del personaje usando Quaternion.Slerp para que el giro sea suave y no brusco.
3. Movimiento Proporcional: Se lee el eje Z del acelerómetro. Como indica el enunciado, este valor se invierte (-zInclinacion) para que al inclinar el móvil hacia adelante, el personaje avance. Se multiplica por un factor de movimiento, logrando que a mayor inclinación, mayor sea la velocidad de desplazamiento.

### Prueba
**Actualmente debido a que no tengo un telefono android no he podido comprobar en su totalidad que la implementacion funciona, epero poder hacerlo en la hora de practicas**
