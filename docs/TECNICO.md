# 🔧 Documentación Técnica: BiciClean

> Volver al [README](../README.md)

## 1. Arquitectura del sistema de filtrado y recirculación

```mermaid
graph TD
    TA[Tanque de agua limpia] --> B1[Bomba de enjuague]
    TD[Depósito de desengrasante biodegradable] --> B2[Bomba dosificadora]

    B2 --> EV1{{EV1: Zona Transmisión}}
    B1 --> EV2{{EV2: Zona Cuadro/Ruedas}}

    EV1 --> Z1[Boquillas de transmisión<br/>baja presión]
    EV2 --> Z2[Boquillas en abanico<br/>asperjado suave]

    Z1 --> CAB[Cabina de lavado]
    Z2 --> CAB

    CAB --> COL[Colector / drenaje]
    COL --> TRAMPA[Trampa de decantación<br/>sedimentación]

    TRAMPA -->|Lodos y grasas| RES[Residuos sólidos<br/>retención]
    TRAMPA -->|Agua clarificada ~60 %| TR[Tanque de recirculación]
    TRAMPA -.->|Excedente| DREN[Descarga]

    TR -->|Primer enjuague| B1
```

**Principio de operación:** el agua usada se recoge en un colector y pasa a una trampa de decantación. Los sólidos y la grasa sedimentan o quedan retenidos, y el agua clarificada se almacena para reutilizarse **solo en el primer enjuague**. La etapa final usa agua limpia.

---

## 2. Secuencia del control automatizado (Arduino/PLC)

```mermaid
sequenceDiagram
    autonumber
    actor U as Usuario
    participant S as Sensor/Pulsador de inicio
    participant C as Controlador (Arduino/PLC)
    participant R as Módulo de relés
    participant EV as Electroválvulas
    participant B as Bombas
    participant V as Secado / Ventilación

    U->>S: Fija la bici y pulsa INICIO
    S->>C: Señal de inicio
    C->>C: Verifica interlocks (rampa fijada, nivel de tanque)

    Note over C,B: Etapa 1: Desengrasado (30 s)
    C->>R: Activa relé Bomba dosificadora + EV Transmisión
    R->>B: Bomba de desengrasante ON
    R->>EV: EV1 abierta
    C->>C: Temporizador T1 = 30 s
    C->>R: Desactiva relés (reposo del desengrasante)

    Note over C,B: Etapa 2: Enjuague (2 min)
    C->>R: Activa Bomba de enjuague + EV2 (Cuadro/Ruedas)
    R->>B: Bomba de enjuague ON
    R->>EV: EV2 abierta, cepillos ON
    C->>C: Temporizador T2 = 2 min
    C->>R: Desactiva bomba, EV2 y cepillos

    Note over C,V: Etapa 3: Secado y recirculación (1 min)
    C->>R: Activa secado + bomba de transferencia de la trampa
    R->>V: Secado ON
    C->>C: Temporizador T3 = 1 min
    C->>R: Todos los relés OFF

    C-->>U: Indicador "Ciclo completo"
```

**Notas de implementación**

- Lógica secuencial por temporizadores (`millis()` en Arduino o TON en PLC); evitar `delay()` bloqueante.
- Los relés conmutan cargas de bajo voltaje; el diseño **no usa alta tensión** en la zona húmeda.
- Recomendado: parada de emergencia y bloqueo si la bici no está fijada.

---

## 3. Métricas de rendimiento

| Métrica | Valor | Notas |
|---|---|---|
| Consumo de agua por ciclo | **8 L** | Medido en prototipo |
| Consumo con manguera común | 30–40 L | Referencia de comparación |
| Ahorro de agua | ~75 % | Calculado vs. valor medio de manguera |
| Tasa de recuperación de agua | **~60 %** | Por decantación |
| Tiempo de ciclo | **< 4 min** | 30 s + 2 min + 1 min = 3 min 30 s |
| Presión Zona Transmisión | `[__]` PSI / `[__]` bar | **Completar con medición** |
| Presión Zona Cuadro/Ruedas | `[__]` PSI / `[__]` bar | **Completar con medición** |
| Caudal por zona | `[__]` L/min | **Completar con medición** |
| Fugas observadas | 0 | Según pruebas del prototipo |

### Desglose del ciclo

| Etapa | Duración | Zona | Fluido | Presión |
|---|---|---|---|---|
| 1. Desengrasado | 30 s | Transmisión | Desengrasante biodegradable | Baja |
| 2. Enjuague | 2 min | Cuadro y ruedas | Agua (reciclada en el primer enjuague) | Baja, en abanico |
| 3. Secado/recirculación | 1 min | Toda la bici | Aire / transferencia de agua | n/a |

---

## 4. Diseño bizona y protección de rodamientos

### El reto de ingeniería

| Escenario | Resultado |
|---|---|
| Presión alta | ❌ Dañaba los sellos de la maza y el pedalier |
| Presión baja | ❌ No eliminaba la grasa negra de la transmisión |

### La solución: dos circuitos hidráulicos independientes

**Zona A: Transmisión** (cadena, cassette, platos)
- Desengrasante biodegradable aplicado a **baja presión**.
- **Tiempo de reposo** para que la química disuelva la grasa.
- Enjuague posterior en la zona B.

**Zona B: Cuadro y ruedas**
- Asperjado suave en **abanico** (patrón amplio, baja energía de impacto).
- Cepillos integrados para retirar el barro sin presión excesiva.

### Mitigación de penetración de agua en rodamientos

1. **Presión regulada:** la baja presión evita vencer los labios de los sellos de maza, pedalier y dirección.
2. **Patrón en abanico:** distribuye el flujo y reduce la fuerza puntual sobre juntas y sellos.
3. **Separación de zonas:** cada zona tiene su propia electroválvula y regulación; no se aplica presión de limpieza intensiva donde no se necesita.
4. **Geometría de asperjado:** boquillas orientadas a evitar el impacto directo contra ejes y sellos.
5. **Química + tiempo en lugar de fuerza:** el desengrasante hace el trabajo que antes hacía la presión.

> **Aprendizaje clave:** la química adecuada (desengrasante + tiempo) supera a la fuerza bruta.

---

## 5. Trabajo futuro

- [ ] Registrar y publicar PSI/bar y caudales por zona.
- [ ] Medición de turbidez del agua recirculada.
- [ ] Telemetría de consumo por ciclo.
- [ ] Modo de pago / control de acceso para instalaciones públicas.
