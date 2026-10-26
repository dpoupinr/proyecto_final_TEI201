### Información del Equipo
- **Integrantes:**
  - Mateo Candela
  - Claudio Martinez
  - Diego Poupin 

  
- **ODS Seleccionado:**
- El ODS 6, Agua limpia y saneamiento
- **Problema a resolver:**
En las zonas rurales de la Región Metropolitana muchas familias no tienen acceso a una red de agua potable como en la ciudad, por lo que se abastecen desde pozos con bombas de 220 V que llenan estanques de 1.000 o 2.000 litros. Estas bombas llenan un estanque en solo 10 a 15 minutos, por lo que es común que rebalse y se desperdicie agua, un recurso cada vez más escaso por la sequía que afecta a la zona central de Chile. Por esto, nuestro proyecto propone un sistema que automatiza el llenado de los estanques: mediante sensores de nivel, el sistema detecta cuándo el estanque está vacío y envía una señal para encender la bomba, y la apaga automáticamente cuando el estanque está lleno. Así se evitan los rebalses, se reduce el consumo eléctrico y la bomba se desgasta menos. El proyecto se relaciona directamente con el Objetivo de Desarrollo Sostenible 6, “Agua limpia y saneamiento”. Contribuye a la meta 6.1, porque asegura un abastecimiento más confiable de agua para comunidades rurales sin red de agua potable. También aporta a la meta 6.4, al promover el uso eficiente del agua y cuidar los pozos y acuíferos. Por último, apoya la meta 6.b, ya que es una solución simple y de bajo costo que las propias comunidades pueden usar y mantener, lo que fortalece su participación en la gestión de su agua.

### Descripción del Proyecto

Para desarrollar el sistema estamos usando una placa ESP32 con WiFi y relé integrado, que funciona como el cerebro del proyecto, junto con un sensor flotante P35 instalado dentro del estanque. Cuando el agua llega al punto donde está el sensor, este envía una señal a la ESP32, que activa o desactiva la bomba a través del relé. Al mismo tiempo, la placa envía por WiFi una notificación al teléfono del usuario indicando el estado de la bomba (encendida o apagada). Además, desde el teléfono el usuario podrá ver información en tiempo real sobre qué tan lleno está el estanque y revisar estadísticas de uso del agua para analizar su consumo. Por ejemplo, podrá ver qué días de la semana y en qué horarios se usa más agua, cuántos litros se consumen al día, a la semana y al mes, cuántas veces se enciende la bomba por día y cuánto tiempo funciona, y comparar el consumo entre semanas. El sistema también podrá generar alertas ante situaciones anormales: si la bomba funciona más tiempo de lo normal, lo que podría indicar que el pozo está seco o que hay una falla, o si hay consumo durante la noche, lo que podría indicar una fuga. Con esta información, las familias podrán tomar mejores decisiones sobre cómo usan el agua y detectar problemas a tiempo.

### Estado del Proyecto
- **Versión actual:** v3.0
- **Última actualización:** [Fecha]
- **Estado:** Prototipo final

---

##  Estructura del Repositorio
```
├── hardware/          # Circuitos, esquemas y BOM
├── software/          # Código Arduino y librerías
├── diseno_3d/        # Modelos Fusion 360 y renders
├── testing/          # Validación con usuarios
├── documentacion/    # Reportes y presentación
└── iteraciones/      # Historial de versiones
```

---

##  Inicio Rápido

### Requisitos Previos
- Arduino IDE 2.x
- Fusion 360
- [Otras herramientas necesarias]

### Instalación
1. Clonar el repositorio
2. Instalar librerías (ver `software/librerias/`)
3. Cargar código en ESP32/Arduino
4. [Pasos adicionales]

---

##  Checklist de Entrega

### Hardware ✓
- [ ] Esquema del circuito (Fritzing/Wokwi)
- [ ] BOM completo
- [ ] Fotos de alta resolución

### Software ✓
- [ ] Código comentado
- [ ] Librerías documentadas
- [ ] Manual de instalación

### Diseño 3D ✓
- [ ] Archivos Fusion 360 (.f3d) con TODOS los componentes
- [ ] Renders de alta calidad (1+ ángulos)
- [ ] Planos técnicos

### Testing ✓
- [ ] Reporte de testing con usuarios (mín. 5)
- [ ] Evidencias fotográficas/videos
- [ ] Datos cuantitativos

### Documentación ✓
- [ ] Reporte final (máx. 15 páginas)
- [ ] Presentación (10 min)

---


---

##  Licencia
Este proyecto es desarrollado como parte del curso TEI201 - Facultad de Ingenieria y Ciencias - Universidad Adolfo Ibañez
