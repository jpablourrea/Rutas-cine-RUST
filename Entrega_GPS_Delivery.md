# Humanos vs LLM: creatividad aplicada a la simulación de datos seriales con micro:bit

**Equipo simulado:** GPS de delivery (NMEA 0183) — ruta de 10 cines de Tijuana, B.C.

**Alumnos:**

- Bernal Lopez Donaldo Manrique
- Lizarraga Figueroa Samanta
- Rodriguez Camarena Joshua Nezib
- Urrea Ramirez Juan Pablo

**Grupo:** SC6B  
**Docente:** René Solis  
**Fecha:** 29 de septiembre de 2026

# 1. Equipo elegido y objetivo

Elegimos el equipo GPS de delivery. El micro:bit V2 simula el receptor GPS de un repartidor que recorre 10 cines de Tijuana y envía su posición por el puerto serial (USB, 115200 baudios, 8N1) como sentencias NMEA 0183 de tipo `$GPGGA`. Una interfaz en Python recibe cada trama, verifica el checksum, la interpreta y la clasifica como NORMAL o FALLA.

En lugar de los 10 puntos de ejemplo de la Zona Río, usé 10 cines reales de la ciudad, ordenados de oeste a este para que formen un recorrido razonable.

# 2. Cómo usé el LLM

El LLM actuó como generador: propuso el formato de las tramas, escribió el firmware del micro:bit y el programa receptor, y ayudó a ubicar los cines en un mapa. En equipo nuestro trabajo fue de ingeniero validador: elegí el equipo, cambié el contexto de la ruta, revisé los resultados y contrasté los datos con fuentes.

| **Paso** | **Qué se pidió** | **Qué entregó el LLM** | **Qué verifiqué** |
| --- | --- | --- | --- |
| Elegir equipo | Adaptar un proyecto previo (un theremin) a la actividad. | Advirtió que el theremin no estaba entre los equipos permitidos y que no usaba un estándar con tramas normal/falla. | Decidimos cambiar a GPS de delivery, que sí está en la lista. |
| Estándar y tramas | Un programa que envíe tramas NMEA normales y con falla. | Tramas `$GPGGA` normales y tres tipos de falla: sin fix, señal pobre y checksum incorrecto. | Comparé formato y checksum con fuentes externas (sección 6). |
| Ruta | Cambiar los puntos a cines de Tijuana. | Buscó 10 cines en Google Maps y entregó sus coordenadas. | Las coordenadas se redondearon a 4 decimales, como en el ejemplo de la actividad. |
| Código | Firmware en Rust y receptor en Python. | Primera versión del firmware con 10 errores de compilación. | Los errores mostraron pines inexistentes y traits sin importar; se corrigieron con la documentación del crate microbit-common [8]. |

# 3. Ruta de delivery: 10 cines de Tijuana

Las coordenadas decimales se convierten al formato NMEA (grados y minutos decimales). Ejemplo para el punto 5: 32.5142° = 32° + 0.5142 × 60′ = 32° 30.852′, que se escribe 3230.852.

| **#** | **Cine** | **Latitud** | **Longitud** | **Lat. NMEA (N)** | **Lon. NMEA (W)** |
| --- | --- | --- | --- | --- | --- |
| 1 | Cinépolis Playas | 32.5311 | -117.1135 | 3231.866 | 11706.810 |
| 2 | Cinemex Paseo de los Héroes | 32.5306 | -117.0263 | 3231.836 | 11701.578 |
| 3 | Cinépolis Plaza Río | 32.5270 | -117.0184 | 3231.620 | 11701.104 |
| 4 | Cinemex Market | 32.5163 | -117.0151 | 3230.978 | 11700.906 |
| 5 | Cinépolis Neidhart | 32.5142 | -117.0123 | 3230.852 | 11700.738 |
| 6 | Cinépolis VIP Hipódromo | 32.5083 | -116.9929 | 3230.498 | 11659.574 |
| 7 | Cinemex El Rosario | 32.4748 | -116.9938 | 3228.488 | 11659.628 |
| 8 | Cinépolis Las Brisas | 32.4981 | -116.9591 | 3229.886 | 11657.546 |
| 9 | Cinemex Alamar | 32.5143 | -116.9667 | 3230.858 | 11658.002 |
| 10 | Cinemex Alameda Otay | 32.5311 | -116.9509 | 3231.866 | 11657.054 |

# 4. Estructura de la trama `$GPGGA`

Ejemplo con el punto 5 de la ruta (Cinépolis Neidhart):

```text
$GPGGA,183000.00,3230.852,N,11700.738,W,1,08,0.9,25.0,M,-34.0,M,,*6E
```

| **Campo** | **Valor en el ejemplo** | **Significado** |
| --- | --- | --- |
| Tipo | GPGGA | GP = receptor GPS; GGA = datos de fijación (Global Positioning System Fix Data). |
| Hora UTC | 183000.00 | 18:30:00.00 (hhmmss.ss). |
| Latitud | 3230.852,N | 32° 30.852′ Norte (ddmm.mmm). |
| Longitud | 11700.738,W | 117° 00.738′ Oeste (dddmm.mmm). |
| Calidad del fix | 1 | 0 = sin fix, 1 = fix GPS. |
| Satélites | 08 | Número de satélites usados. |
| HDOP | 0.9 | Dilución horizontal de la precisión (menor es mejor). |
| Altitud | 25.0,M | Altitud sobre el nivel medio del mar, en metros. |
| Separación geoidal | -34.0,M | Diferencia entre el geoide y el elipsoide, en metros. |
| Edad / estación DGPS | (vacíos) | Solo se usan con correcciones diferenciales. |
| Checksum | `*6E` | XOR de todos los caracteres entre `$` y `*`, en dos dígitos hexadecimales. |

Checksum: se hace XOR carácter por carácter de `GPGGA,183000.00,...,M,,` (sin el `$` ni el `*`). Para esta trama el resultado es `0x6E`. Para el punto 5 de los ejemplos de la actividad (`3231.596,N,11701.200,W`), el checksum que se pedía calcular es `0x65`.

# 5. Tramas normal y con falla

| **Caso** | **Trama enviada** | **Clasificación** |
| --- | --- | --- |
| Normal | `$GPGGA,183000.00,3230.852,N,11700.738,W,1,08,0.9,25.0,M,-34.0,M,,*6E` | NORMAL |
| Sin fix GPS | `$GPGGA,183100.00,,,,,0,00,99.99,,,,,,*6D` | FALLA: sin posición |
| Señal pobre (3 satélites, HDOP 25.0) | `$GPGGA,183110.00,3231.866,N,11657.054,W,1,03,25.0,25.0,M,-34.0,M,,*53` | FALLA: señal pobre |
| Checksum incorrecto | `$GPGGA,183120.00,3231.866,N,11657.054,W,1,08,0.9,25.0,M,-34.0,M,,*3F` | FALLA: trama corrupta (el correcto sería 65) |

Todas las tramas miden entre 42 y 71 caracteres contando el retorno de carro y el salto de línea.

# 6. Validación contra fuentes

La actividad pide comprobar si los bytes, unidades y rangos que entrega el LLM son reales. Comparé cada dato contra fuentes externas. Las fuentes de la sección 10 son secundarias; la norma oficial es de pago y no pude consultarla directamente.

| **Dato** | **Lo que dijo el LLM** | **Lo que dicen las fuentes** | **Resultado** |
| --- | --- | --- | --- |
| Tipo de sentencia | `$GPGGA (GP = receptor GPS).` | GP identifica GPS y GN multiconstelación; GGA es la sentencia de datos de posición [2][1]. | Correcto |
| Orden de campos | Hora, latitud, N/S, longitud, E/W, calidad, satélites, HDOP, altitud, M, separación geoidal, M, edad, estación. | El mismo orden aparece en [2] y en la definición de [4]. | Correcto |
| Formato de coordenadas | ddmm.mmm para latitud y dddmm.mmm para longitud. | Las coordenadas NMEA son grados más minutos decimales, no grados decimales [1]. Lo comprobé convirtiendo 32.5142° a 3230.852. | Correcto |
| Checksum | XOR de los caracteres entre `$` y `*`, en 2 dígitos hex. | XOR de todos los bytes entre `$` y `*` [1][3]. | Correcto; calculado en Rust y en Python. |
| Calidad del fix | 0 = sin fix, 1 = fix GPS. | 0 = inválido, 1 = SPS autónomo, 2 = DGPS, y otros valores hasta 9 [1][4]. | Correcto |
| Trama sin fix | Posición vacía, calidad 0, 00 satélites, HDOP 99.99. | Un ejemplo de [5] usa posición vacía, calidad 0 y satélites 0, con el HDOP vacío. | Parcial: no confirmé el HDOP 99.99. |
| Altitud y separación geoidal | 25.0 m y −34.0 m. | La altitud es sobre el nivel medio del mar y la separación geoidal va en un campo aparte [2]. Las fuentes no dan valores para Tijuana. | Formato correcto; valores supuestos, pendientes de verificar con un modelo de geoide. |
| Umbrales de falla | Falla si hay menos de 4 satélites o HDOP mayor a 5. | Las fuentes consultadas no fijan estos umbrales. | Criterio propio de ingeniería, no del estándar. |
| Longitud de la trama | No lo mencionó. | La norma fija una longitud máxima para las sentencias; la cifra que suele citarse es 82 caracteres. | Mis tramas miden hasta 71; falta confirmar la cifra en la norma. |

## Hallazgos al validar

- Un ejemplo publicado tenía un checksum distinto al calculado. La página de Mapit GIS [3] muestra una trama GGA que termina en `*47`, pero al recalcular el XOR obtuve 44. En cambio, la trama clásica (`123519`, `4807.038,N`, `01131.000,E`) que [6] documenta con checksum `0x47` sí da 47. Por eso recalculé todos los checksums en lugar de copiarlos.

- El código del LLM no compiló a la primera. Usó nombres de pines que no existen en esta versión del crate (p0_03, p0_04, p0_20) y olvidó importar traits de embedded-hal 1.0. La documentación de microbit-common [8] muestra que esos pines están en board.edge y que p0_20 se usa para el micrófono.

- Nuestro primera idea era hacer un theremin en Rust, sin embargo, era difícil cumplir con los estándares de la actividad. Fue decisión humana cambiar a un equipo de la lista.

- Los valores de altitud, separación geoidal y los umbrales de falla fueron supuestos del LLM. Quedan señalados como no verificados en la tabla.

# 7. Implementación

## Firmware del micro:bit (Rust)

- Placa: micro:bit V2 (nRF52833). Las tramas salen por el puerto serial que comparte con el USB, a 115200 baudios.

- Librerías: microbit-v2 (placa, UARTE y botones), cortex-m-rt (arranque), panic-halt (manejo de errores) y embedded-hal (traits de temporizador y entradas).

- Ciclo de 13 tramas, una por segundo: los 10 cines como tramas normales, y después sin fix, señal pobre y checksum incorrecto. La hora avanza 10 segundos por trama.

- Botones: mantener A fuerza tramas sin fix; mantener B fuerza tramas con checksum incorrecto.

## Interfaz receptora (Python, receiver.py)

El programa lee las líneas del puerto serial con pyserial, o usa tramas de ejemplo con la opción --demo. Clasifica cada trama en este orden:

| **Condición** | **Clasificación** | **Motivo mostrado** |
| --- | --- | --- |
| El checksum calculado no coincide | FALLA | checksum incorrecto (trama corrupta) |
| No es una sentencia GPGGA válida | FALLA | trama no es GPGGA válida |
| Calidad de fix 0 o posición vacía | FALLA | sin fix GPS (sin posición) |
| Menos de 4 satélites o HDOP mayor a 5.0 | FALLA | señal pobre |
| Ninguna de las anteriores | NORMAL | muestra el cine más cercano, latitud, longitud, satélites y HDOP |

# 8. Resultados

**Figura 1.** Salida del receptor en modo demostración: tres tramas normales (con el cine identificado) y tres fallas.

![Figura 1. Salida del receptor en modo demostración](img/figura1.png)

**Figura 2.**

> Pendiente: captura del receptor con el micro:bit real conectado por USB.

# 9. Reflexión: humanos vs LLM

En esta práctica, el LLM trabajó sobre todo en las dos primeras: recombinó patrones que ya conocía (el formato GGA, el cálculo de checksum, código para microcontroladores) y exploró las reglas del dominio para generar tramas normales y fallas en segundos.

Nosotros como equipo nos vimos en la obligación de conocer el sistema de embebidos y que un microbitv2 puede hacer cualquier cosa que se proponga ya que es un dispositivo de desarrollo como cualquier otro. Elegimos el equipo correcto cuando el primer proyecto no encajaba, cambiamos el contexto de la ruta a cines de Tijuana, y validamos los datos contra fuentes en lugar de aceptarlos. La revisión nos hizo cambiar datos que no eran verificados.

La simulación respeta las reglas del estándar NMEA 0183 y no las transforma; la parte que más pesó fue hacer las preguntas correctas y conectar el hardware con el estándar real. El código se desarrolló con ayuda de un LLM, y por eso lo revisé contra la documentación y lo probé en lugar de confiar en él. 

# 10. Fuentes

[1] GNSS Metrics. NMEA 0183. <https://gnssmetrics.com/learn/protocols/nmea-0183/> (consultado el 29 de septiembre de 2026).

[2] Open Survey. NMEA 0183 for Surveyors: GGA, GST, and What Fix Quality Really Means. <https://opensurvey3d.com/articles/nmea-0183-gga-gst-explained/> (consultado el 29 de septiembre de 2026).

[3] Mapit GIS. NMEA Protocol. <https://mapitgis.com/docs/external-gnss/nmea-protocol/> (consultado el 29 de septiembre de 2026).

[4] nmea-kit 0.5.6, definición de la sentencia GGA. <https://docs.rs/crate/nmea-kit/latest/source/src/nmea/sentences/gga.rs> (consultado el 29 de septiembre de 2026).

[5] nmeasis 26.4.1, decodificador de la sentencia GGA. <https://docs.rs/crate/nmeasis/latest/source/src/message/gga.rs> (consultado el 29 de septiembre de 2026).

[6] marlin-nmea-0183, decodificadores de sentencias NMEA 0183. <https://docs.rs/marlin-nmea-0183> (consultado el 29 de septiembre de 2026).

[7] NMEA. NMEA 0183 Standard, versión 4.11. <https://www.nmea.org> (norma oficial de pago; no consultada directamente).

[8] microbit-common 0.16.0, código de la placa micro:bit V2 (board.rs). <https://docs.rs/crate/microbit-common/latest/source/src/v2/board.rs> (consultado el 29 de septiembre de 2026).

# Anexo A. Código del micro:bit (`src/main.rs`)

```rust
//! GPS de delivery (ruta de 10 cines, Tijuana) - micro:bit V2
//!
//! Sends NMEA 0183 $GPGGA sentences over the USB serial port (115200 8N1).
//!
//! Sequence (repeats forever, 1 frame per second):
//!   steps 0-9 : NORMAL      - the 10 cinemas of the route, fix=1, 8 satellites, HDOP 0.9
//!   step  10  : FAULT       - no fix (fix=0, 0 satellites, empty position, HDOP 99.99)
//!   step  11  : FAULT       - poor signal (3 satellites, HDOP 25.0)
//!   step  12  : FAULT       - corrupted frame (wrong checksum)
//!
//! Buttons: hold A = send "no fix" frames, hold B = send bad-checksum frames.
//!
//! Format of $GPGGA (NMEA 0183):
//!   $GPGGA,hhmmss.ss,ddmm.mmm,N,dddmm.mmm,W,fix,sats,hdop,alt,M,geoid,M,age,ref*CS
//!   CS = XOR of every character between '$' and '*', in 2 hex digits.

#![no_std]
#![no_main]

use core::fmt::Write;
use cortex_m_rt::entry;
use embedded_hal::{delay::DelayNs, digital::InputPin};
use microbit::{
    hal::{
        uarte::{self, Baudrate, Parity},
        Timer,
    },
    Board,
};
use panic_halt as _;

/// Delivery route through 10 cinemas in Tijuana: (latitude N, longitude W) in degrees x 10000.
///  1 Cinepolis Playas            2 Cinemex Paseo de los Heroes   3 Cinepolis Plaza Rio
///  4 Cinemex Market              5 Cinepolis Neidhart            6 Cinepolis VIP Hipodromo
///  7 Cinemex El Rosario          8 Cinepolis Las Brisas          9 Cinemex Alamar
/// 10 Cinemex Alameda Otay
const ROUTE: [(u32, u32); 10] = [
    (325311, 1171135),
    (325306, 1170263),
    (325270, 1170184),
    (325163, 1170151),
    (325142, 1170123),
    (325083, 1169929),
    (324748, 1169938),
    (324981, 1169591),
    (325143, 1169667),
    (325311, 1169509),
];

#[derive(Clone, Copy, PartialEq)]
enum Kind {
    Normal,
    NoFix,
    PoorSignal,
    BadChecksum,
}

/// Small fixed buffer so we can use write!() without an allocator.
struct Line {
    buf: [u8; 128],
    len: usize,
}

impl Line {
    fn new() -> Self {
        Line { buf: [0; 128], len: 0 }
    }
    fn bytes(&self) -> &[u8] {
        &self.buf[..self.len]
    }
}

impl Write for Line {
    fn write_str(&mut self, s: &str) -> core::fmt::Result {
        let b = s.as_bytes();
        if self.len + b.len() > self.buf.len() {
            return Err(core::fmt::Error);
        }
        self.buf[self.len..self.len + b.len()].copy_from_slice(b);
        self.len += b.len();
        Ok(())
    }
}

/// NMEA checksum: XOR of all bytes between '$' and '*'.
fn checksum(body: &[u8]) -> u8 {
    body.iter().fold(0u8, |acc, b| acc ^ b)
}

/// Build the sentence body (no '$', no '*CS').
fn body(out: &mut Line, kind: Kind, idx: usize, t: u32) {
    let (h, m, s) = (t / 3600, (t / 60) % 60, t % 60);

    if kind == Kind::NoFix {
        let _ = write!(out, "GPGGA,{:02}{:02}{:02}.00,,,,,0,00,99.99,,,,,,", h, m, s);
        return;
    }

    let (lat, lon) = ROUTE[idx];
    let (lat_deg, lat_frac) = (lat / 10_000, lat % 10_000);
    let (lon_deg, lon_frac) = (lon / 10_000, lon % 10_000);
    // minutes x 1000 = fraction(4 decimals) x 60 / 10 = fraction x 6
    let lat_min = lat_frac * 6;
    let lon_min = lon_frac * 6;

    let (sats, hdop_x10) = if kind == Kind::PoorSignal { (3, 250) } else { (8, 9) };

    let _ = write!(
        out,
        "GPGGA,{:02}{:02}{:02}.00,{:02}{:02}.{:03},N,{:03}{:02}.{:03},W,1,{:02},{}.{},25.0,M,-34.0,M,,",
        h,
        m,
        s,
        lat_deg,
        lat_min / 1000,
        lat_min % 1000,
        lon_deg,
        lon_min / 1000,
        lon_min % 1000,
        sats,
        hdop_x10 / 10,
        hdop_x10 % 10
    );
}

#[entry]
fn main() -> ! {
    let board = Board::take().unwrap();

    let mut timer = Timer::new(board.TIMER0);
    let mut serial = uarte::Uarte::new(
        board.UARTE0,
        board.uart.into(),
        Parity::EXCLUDED,
        Baudrate::BAUD115200,
    );

    let mut btn_a = board.buttons.button_a;
    let mut btn_b = board.buttons.button_b;

    let mut step: usize = 0;
    let mut t: u32 = 18 * 3600 + 29 * 60 + 20; // 18:29:20, +10 s per frame

    loop {
        let kind = if btn_a.is_low().unwrap_or(false) {
            Kind::NoFix
        } else if btn_b.is_low().unwrap_or(false) {
            Kind::BadChecksum
        } else {
            match step {
                0..=9 => Kind::Normal,
                10 => Kind::NoFix,
                11 => Kind::PoorSignal,
                _ => Kind::BadChecksum,
            }
        };
        let idx = step.min(9);

        let mut b = Line::new();
        body(&mut b, kind, idx, t);

        let mut cs = checksum(b.bytes());
        if kind == Kind::BadChecksum {
            cs ^= 0x5A; // deliberately wrong
        }

        let mut frame = Line::new();
        let _ = frame.write_str("$");
        let _ = frame.write_str(core::str::from_utf8(b.bytes()).unwrap_or(""));
        let _ = write!(frame, "*{:02X}\r\n", cs);

        let _ = serial.write(frame.bytes());

        step = (step + 1) % 13;
        t = (t + 10) % 86_400;
        timer.delay_ms(1000);
    }
}
```

# Anexo B. Interfaz receptora (`receiver.py`)

```python
"""Receiver / classifier for the micro:bit GPS delivery simulator.

Usage:
    pip install pyserial
    python receiver.py COM5          # real micro:bit (see Device Manager for the COM number)
    python receiver.py --demo        # no board needed, uses sample frames
"""
import sys

DEMO = """\
$GPGGA,182920.00,3231.866,N,11706.810,W,1,08,0.9,25.0,M,-34.0,M,,*61
$GPGGA,183010.00,3230.498,N,11659.574,W,1,08,0.9,25.0,M,-34.0,M,,*62
$GPGGA,183050.00,3231.866,N,11657.054,W,1,08,0.9,25.0,M,-34.0,M,,*63
$GPGGA,183100.00,,,,,0,00,99.99,,,,,,*6D
$GPGGA,183110.00,3231.866,N,11657.054,W,1,03,25.0,25.0,M,-34.0,M,,*53
$GPGGA,183120.00,3231.866,N,11657.054,W,1,08,0.9,25.0,M,-34.0,M,,*3F""".splitlines()


CINES = [
    ("Cinepolis Playas", 32.5311, -117.1135),
    ("Cinemex Paseo de los Heroes", 32.5306, -117.0263),
    ("Cinepolis Plaza Rio", 32.5270, -117.0184),
    ("Cinemex Market", 32.5163, -117.0151),
    ("Cinepolis Neidhart", 32.5142, -117.0123),
    ("Cinepolis VIP Hipodromo", 32.5083, -116.9929),
    ("Cinemex El Rosario", 32.4748, -116.9938),
    ("Cinepolis Las Brisas", 32.4981, -116.9591),
    ("Cinemex Alamar", 32.5143, -116.9667),
    ("Cinemex Alameda Otay", 32.5311, -116.9509),
]


def nearest_cine(lat, lon):
    return min(CINES, key=lambda c: (c[1] - lat) ** 2 + (c[2] - lon) ** 2)[0]


def checksum_ok(line):
    if not line.startswith("$") or "*" not in line:
        return False
    body, cs = line[1:].split("*", 1)
    calc = 0
    for ch in body:
        calc ^= ord(ch)
    try:
        return calc == int(cs.strip()[:2], 16)
    except ValueError:
        return False


def to_degrees(value, hemi, deg_digits):
    deg = int(value[:deg_digits])
    minutes = float(value[deg_digits:])
    d = deg + minutes / 60
    return -d if hemi in ("S", "W") else d


def classify(line):
    """Return (status, reason, info)."""
    if not checksum_ok(line):
        return "FALLA", "checksum incorrecto (trama corrupta)", ""
    f = line[1:].split("*")[0].split(",")
    if f[0] != "GPGGA" or len(f) < 9:
        return "FALLA", "trama no es GPGGA valida", ""
    fix, sats, hdop = int(f[6] or 0), int(f[7] or 0), float(f[8] or 99.99)
    if fix == 0 or not f[2]:
        return "FALLA", "sin fix GPS (sin posicion)", ""
    lat = to_degrees(f[2], f[3], 2)
    lon = to_degrees(f[4], f[5], 3)
    info = f"{nearest_cine(lat, lon)} lat={lat:.4f} lon={lon:.4f} sats={sats} hdop={hdop}"
    if sats < 4 or hdop > 5.0:
        return "FALLA", "senal pobre (pocos satelites / HDOP alto)", info
    return "NORMAL", "ok", info


def main():
    if len(sys.argv) < 2:
        print(__doc__)
        return
    if sys.argv[1] == "--demo":
        lines = iter(DEMO)
    else:
        import serial  # pyserial

        port = serial.Serial(sys.argv[1], 115200, timeout=3)
        lines = (port.readline().decode(errors="ignore") for _ in iter(int, 1))
    for raw in lines:
        line = raw.strip()
        if not line:
            continue
        status, reason, info = classify(line)
        print(f"[{status:6}] {reason:45} {info}\n         {line}")


if __name__ == "__main__":
    main()
```
