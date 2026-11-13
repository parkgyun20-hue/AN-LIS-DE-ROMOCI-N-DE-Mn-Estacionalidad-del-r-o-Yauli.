// =====================================================================
//  Secuencia temporal del cauce de un río (Sentinel-2) · sep–oct 2026
//  Zona de estudio: Junín (Lago Chinchaycocha – río Yauli / Mantaro)
//  Autora: Nicoll · Tesis 2 (UTEC)
//
//  Qué hace este script:
//   1. Define el tramo de río (corredor = línea del río + buffer).
//   2. Carga Sentinel-2 SR y quita nubes con Cloud Score+.
//   3. Para CADA fecha de sep–oct 2026 calcula el área de agua del cauce
//      (índice de agua > umbral) y el % del tramo libre de nubes.
//   4. Grafica la serie temporal y arma compuestos quincenales.
//   5. Genera un GIF (timelapse) de las escenas despejadas.
//   6. Compara sep y oct de 2019–2026 (¿2026 es normal, seco o húmedo?).
//   7. Grafica la lluvia mensual 2026 vs. el promedio 2016–2025 (CHIRPS)
//      y la lluvia diaria de sep–oct 2026, para explicar la estacionalidad.
//   8. Exporta las tablas a Google Drive (CSV).
// =====================================================================


// ---------------------------------------------------------------------
// 0. PARÁMETROS (lo único que normalmente hay que tocar)
// ---------------------------------------------------------------------
var ANIO        = 2026;
var FECHA_INI   = '2026-09-01';
var FECHA_FIN   = '2026-11-01';   // fecha final EXCLUSIVA (incluye todo octubre)

var INDICE      = 'NDWI';   // 'NDWI'  (B3-B8, 10 m)  → mejor para ríos angostos (< 30 m)
                            // 'MNDWI' (B3-B11, 20 m) → mejor para el lago / ríos anchos
var UMBRAL_AGUA = 0;        // índice > 0 = agua (subir a 0.1 si aparecen sombras como agua)
var BUFFER_M    = 60;       // ancho a cada lado de la línea del río (m)
var UMBRAL_CS   = 0.60;     // Cloud Score+: 0.60 = estándar; 0.50 = más permisivo
var MIN_CLARO   = 70;       // % mínimo del tramo sin nubes para usar una fecha
var ANIOS_COMP  = [2019, 2020, 2021, 2022, 2023, 2024, 2025, 2026];
var FUENTE_LLUVIA = 'CHIRPS';   // 'CHIRPS' (mejor en los Andes) o 'ERA5' (llega antes)


// ---------------------------------------------------------------------
// 1. ZONA DE ESTUDIO Y TRAMO DE RÍO
// ---------------------------------------------------------------------
// Caja del Lago Chinchaycocha (tu script original)
var caja       = ee.Geometry.Rectangle([-76.30, -11.25, -75.90, -10.80]);
var centroLago = ee.Geometry.Point([-76.10, -11.02]);

// Línea del río POR DEFECTO: río Yauli, desde Yauli hasta su confluencia
// con el Mantaro en La Oroya (trazado APROXIMADO).
// >>> Para usar otro río (p. ej. el Mantaro a la salida del lago, o el
// >>> San Juan), dibuja una LÍNEA con la herramienta de dibujo del mapa
// >>> y nómbrala  rio  en "Imports". El script la usará automáticamente.
var rioPorDefecto = ee.Geometry.LineString([
  [-76.090, -11.668],   // pueblo de Yauli
  [-76.060, -11.635],
  [-76.030, -11.600],
  [-75.990, -11.570],
  [-75.955, -11.548],
  [-75.920, -11.530],
  [-75.905, -11.522]    // La Oroya (confluencia con el Mantaro)
]);

var rioLinea = (typeof rio !== 'undefined') ? rio : rioPorDefecto;
var tramo    = ee.FeatureCollection(rioLinea).geometry().buffer(BUFFER_M);
var zonaLluvia = tramo.buffer(15000);   // cuenca aproximada para la lluvia

var areaTramoHa = ee.Number(tramo.area(1)).divide(10000);
print('Área del corredor analizado (ha):', areaTramoHa);

Map.centerObject(tramo, 12);
Map.setOptions('SATELLITE');


// ---------------------------------------------------------------------
// 2. SENTINEL-2 + CLOUD SCORE+
// ---------------------------------------------------------------------
var S2  = ee.ImageCollection('COPERNICUS/S2_SR_HARMONIZED');
var CSP = ee.ImageCollection('GOOGLE/CLOUD_SCORE_PLUS/V1/S2_HARMONIZED');

function prepararS2(img) {
  var esc   = img.select(['B2', 'B3', 'B4', 'B8', 'B11']).multiply(0.0001);
  var claro = img.select('cs_cdf').gte(UMBRAL_CS);
  var ndwi  = esc.normalizedDifference(['B3', 'B8']).rename('NDWI');
  var mndwi = esc.normalizedDifference(['B3', 'B11']).rename('MNDWI');
  var rgb   = esc.select(['B4', 'B3', 'B2'], ['R', 'G', 'B']);   // sin máscara (para el GIF)

  return esc.addBands([ndwi, mndwi])
            .updateMask(claro)
            .addBands(rgb)
            .addBands(claro.rename('claro'))
            .copyProperties(img, ['system:time_start']);
}

function cargarS2(ini, fin, region) {
  return S2.filterBounds(region)
           .filterDate(ini, fin)
           .linkCollection(CSP, ['cs_cdf'])
           .map(function (img) { return ee.Image(prepararS2(img)); });
}

// Une las teselas (MGRS) de un mismo día en un solo mosaico
function mosaicoDiario(col) {
  var fechas = ee.List(col.aggregate_array('system:time_start'))
    .map(function (t) { return ee.Date(t).format('YYYY-MM-dd'); })
    .distinct();
  return ee.ImageCollection.fromImages(fechas.map(function (f) {
    var d = ee.Date.parse('YYYY-MM-dd', f);
    return col.filterDate(d, d.advance(1, 'day')).mosaic()
              .set('system:time_start', d.millis())
              .set('fecha', f);
  }));
}


// ---------------------------------------------------------------------
// 3. AGUA EN EL TRAMO PARA CADA FECHA
// ---------------------------------------------------------------------
function medirAgua(img) {
  var area  = ee.Image.pixelArea();
  var agua  = img.select(INDICE).gt(UMBRAL_AGUA);   // enmascarada donde hay nube

  var sumas = ee.Image.cat([
    agua.multiply(area).rename('agua_m2'),
    img.select('claro').multiply(area).rename('claro_m2')
  ]).reduceRegion({
    reducer: ee.Reducer.sum(), geometry: tramo, scale: 10, maxPixels: 1e9
  });

  var medio = img.select(INDICE).reduceRegion({
    reducer: ee.Reducer.mean(), geometry: tramo, scale: 10, maxPixels: 1e9
  }).get(INDICE);

  var aguaHa  = ee.Number(sumas.get('agua_m2')).divide(10000);
  var pctClaro = ee.Number(sumas.get('claro_m2')).divide(10000)
                   .divide(areaTramoHa).multiply(100);

  return img.set({
    agua_ha: aguaHa,
    pct_claro: pctClaro,
    indice_medio: medio
  });
}

var diario = mosaicoDiario(cargarS2(FECHA_INI, FECHA_FIN, tramo))
               .map(medirAgua)
               .sort('system:time_start');

var diarioOK = diario.filter(ee.Filter.gte('pct_claro', MIN_CLARO));

print('Fechas Sentinel-2 en el periodo:', diario.size());
print('Fechas con ≥ ' + MIN_CLARO + ' % del tramo despejado:', diarioOK.size());
print('Fechas útiles:', diarioOK.aggregate_array('fecha'));

// Tabla (FeatureCollection) para gráficos y exportación
function aTabla(col) {
  return ee.FeatureCollection(col.map(function (img) {
    return ee.Feature(null, {
      'system:time_start': img.get('system:time_start'),
      fecha: img.get('fecha'),
      agua_ha: img.get('agua_ha'),
      pct_claro: img.get('pct_claro'),
      indice_medio: img.get('indice_medio')
    });
  }));
}
var tablaTodo = aTabla(diario);
var tablaOK   = aTabla(diarioOK);


// ---------------------------------------------------------------------
// 4. GRÁFICOS DE LA SERIE SEP–OCT 2026
// ---------------------------------------------------------------------
print(ui.Chart.feature.byFeature(tablaOK, 'system:time_start', ['agua_ha'])
  .setChartType('LineChart')
  .setOptions({
    title: 'Área de agua en el cauce · sep–oct ' + ANIO + ' (' + INDICE + ' > ' + UMBRAL_AGUA + ')',
    hAxis: {title: 'Fecha', format: 'dd MMM'},
    vAxis: {title: 'Área de agua (ha)'},
    pointSize: 6, lineWidth: 2, colors: ['1f77b4'], legend: {position: 'none'}
  }));

print(ui.Chart.feature.byFeature(tablaOK, 'system:time_start', ['indice_medio'])
  .setChartType('LineChart')
  .setOptions({
    title: INDICE + ' medio del corredor · sep–oct ' + ANIO,
    hAxis: {title: 'Fecha', format: 'dd MMM'},
    vAxis: {title: INDICE + ' medio'},
    pointSize: 6, lineWidth: 2, colors: ['17becf'], legend: {position: 'none'}
  }));

print(ui.Chart.feature.byFeature(tablaTodo, 'system:time_start', ['pct_claro'])
  .setChartType('ColumnChart')
  .setOptions({
    title: '% del tramo sin nubes por fecha (control de calidad)',
    hAxis: {title: 'Fecha', format: 'dd MMM'},
    vAxis: {title: '% despejado', viewWindow: {min: 0, max: 100}},
    colors: ['999999'], legend: {position: 'none'}
  }));


// ---------------------------------------------------------------------
// 5. COMPUESTOS QUINCENALES EN EL MAPA
// ---------------------------------------------------------------------
var visRGB = {bands: ['B4', 'B3', 'B2'], min: 0, max: 0.3};
var periodos = [
  ['2026-09-01', '2026-09-16', '1–15 sep', '00ffff'],
  ['2026-09-16', '2026-10-01', '16–30 sep', '0080ff'],
  ['2026-10-01', '2026-10-16', '1–15 oct', 'ff8000'],
  ['2026-10-16', '2026-11-01', '16–31 oct', 'ff0000']
];

var col2026 = cargarS2(FECHA_INI, FECHA_FIN, tramo);
periodos.forEach(function (p) {
  var comp = col2026.filterDate(p[0], p[1]).median().clip(tramo.buffer(2000));
  Map.addLayer(comp, visRGB, 'RGB ' + p[2], false);
  Map.addLayer(comp.select(INDICE).gt(UMBRAL_AGUA).selfMask().clip(tramo),
               {palette: [p[3]]}, 'Agua ' + p[2], p[2] === '1–15 sep');
});

// Referencia: frecuencia histórica de agua (JRC 1984–2021)
var jrc = ee.Image('JRC/GSW1_4/GlobalSurfaceWater').select('occurrence');
Map.addLayer(jrc.clip(tramo.buffer(2000)),
             {min: 0, max: 100, palette: ['ffffff', '0000ff']},
             'JRC frecuencia de agua (%)', false);
Map.addLayer(caja, {color: 'yellow'}, 'Caja Chinchaycocha', false);
Map.addLayer(ee.FeatureCollection(rioLinea).style({color: 'white', width: 1}), {}, 'Línea del río');
Map.addLayer(ee.FeatureCollection(tramo).style({color: 'yellow', fillColor: '00000000', width: 2}),
             {}, 'Tramo analizado');


// ---------------------------------------------------------------------
// 6. GIF (TIMELAPSE) DE LAS ESCENAS DESPEJADAS
// ---------------------------------------------------------------------
var regionGif = tramo.bounds().buffer(1500).bounds();
var cuadros = diarioOK.map(function (img) {
  var fondo = img.select(['R', 'G', 'B']).visualize({min: 0, max: 0.3});
  var agua  = img.select(INDICE).gt(UMBRAL_AGUA).selfMask().clip(tramo)
                 .visualize({palette: ['00ffff']});
  var borde = ee.Image().paint(ee.FeatureCollection(tramo), 0, 1)
                 .visualize({palette: ['ffff00']});
  return ee.ImageCollection([fondo, agua, borde]).mosaic();
});
var paramsGif = {dimensions: 700, region: regionGif, framesPerSecond: 1.5, crs: 'EPSG:3857'};
print('GIF sep–oct ' + ANIO + ' (orden de cuadros = lista de "Fechas útiles"):');
print(ui.Thumbnail(cuadros, paramsGif));
print('Enlace para descargar el GIF:', cuadros.getVideoThumbURL(paramsGif));


// ---------------------------------------------------------------------
// 7. ESTACIONALIDAD: SEP Y OCT DE 2019–2026
// ---------------------------------------------------------------------
var filasAnuales = [];
ANIOS_COMP.forEach(function (a) {
  [9, 10].forEach(function (m) {
    var ini  = ee.Date.fromYMD(a, m, 1);
    var comp = cargarS2(ini, ini.advance(1, 'month'), tramo).median();
    var area = ee.Image.pixelArea();
    var s = ee.Image.cat([
      comp.select(INDICE).gt(UMBRAL_AGUA).multiply(area).rename('agua'),
      comp.select(INDICE).mask().multiply(area).rename('cubierto')
    ]).reduceRegion({reducer: ee.Reducer.sum(), geometry: tramo, scale: 10, maxPixels: 1e9});
    filasAnuales.push(ee.Feature(null, {
      anio: String(a),
      mes: (m === 9 ? 'Septiembre' : 'Octubre'),
      agua_ha: ee.Number(s.get('agua')).divide(10000),
      pct_cubierto: ee.Number(s.get('cubierto')).divide(10000).divide(areaTramoHa).multiply(100)
    }));
  });
});
var tablaAnual = ee.FeatureCollection(filasAnuales);

print(ui.Chart.feature.groups(tablaAnual, 'anio', 'agua_ha', 'mes')
  .setChartType('ColumnChart')
  .setOptions({
    title: 'Área de agua del cauce: septiembre vs octubre (' + ANIOS_COMP[0] + '–' + ANIO + ')',
    hAxis: {title: 'Año'}, vAxis: {title: 'Área de agua (ha)'},
    colors: ['e6a23c', '1f77b4']
  }));


// ---------------------------------------------------------------------
// 8. LLUVIA: 2026 vs PROMEDIO 2016–2025, Y LLUVIA DIARIA SEP–OCT 2026
// ---------------------------------------------------------------------
function coleccionLluvia() {
  if (FUENTE_LLUVIA === 'ERA5') {
    return ee.ImageCollection('ECMWF/ERA5_LAND/DAILY_AGGR')
      .map(function (i) {
        return ee.Image(i.select('total_precipitation_sum').multiply(1000).rename('pp')
                .copyProperties(i, ['system:time_start']));
      });
  }
  return ee.ImageCollection('UCSB-CHG/CHIRPS/DAILY').select(['precipitation'], ['pp']);
}
var LLUVIA = coleccionLluvia();
var escalaLluvia = (FUENTE_LLUVIA === 'ERA5') ? 11132 : 5566;

function lluviaMes(anio, mes) {
  var ini = ee.Date.fromYMD(anio, mes, 1);
  var col = LLUVIA.filterDate(ini, ini.advance(1, 'month'));
  var valor = col.sum().reduceRegion({
    reducer: ee.Reducer.mean(), geometry: zonaLluvia, scale: escalaLluvia
  }).get('pp');
  // Solo se acepta si el mes está completo (≥ 28 días cargados)
  return ee.Algorithms.If(col.size().gte(28), valor, null);
}

var meses = ee.List.sequence(1, 12);
var nombresMes = ['Ene', 'Feb', 'Mar', 'Abr', 'May', 'Jun', 'Jul', 'Ago', 'Sep', 'Oct', 'Nov', 'Dic'];
var tablaLluvia = ee.FeatureCollection(meses.map(function (m) {
  m = ee.Number(m);
  var clim = ee.List.sequence(2016, 2025).map(function (a) {
    var ini = ee.Date.fromYMD(a, m, 1);
    return LLUVIA.filterDate(ini, ini.advance(1, 'month')).sum().reduceRegion({
      reducer: ee.Reducer.mean(), geometry: zonaLluvia, scale: escalaLluvia
    }).get('pp');
  });
  return ee.Feature(null, {
    mes: m,
    nombre: ee.List(nombresMes).get(m.subtract(1)),
    promedio_2016_2025_mm: ee.List(clim).reduce(ee.Reducer.mean()),
    anio_2026_mm: lluviaMes(ANIO, m)
  });
}));

print(ui.Chart.feature.byFeature(tablaLluvia, 'nombre', ['promedio_2016_2025_mm', 'anio_2026_mm'])
  .setChartType('ColumnChart')
  .setOptions({
    title: 'Lluvia mensual (' + FUENTE_LLUVIA + '): ' + ANIO + ' vs promedio 2016–2025',
    hAxis: {title: 'Mes'}, vAxis: {title: 'Precipitación (mm/mes)'},
    colors: ['bbbbbb', '1f77b4']
  }));

var lluviaDiaria = LLUVIA.filterDate(FECHA_INI, FECHA_FIN);
print('Días de lluvia disponibles en sep–oct ' + ANIO + ' (' + FUENTE_LLUVIA + '):', lluviaDiaria.size());
print(ui.Chart.image.series({
  imageCollection: lluviaDiaria, region: zonaLluvia,
  reducer: ee.Reducer.mean(), scale: escalaLluvia, xProperty: 'system:time_start'
}).setChartType('ColumnChart').setOptions({
  title: 'Lluvia diaria sep–oct ' + ANIO + ' (' + FUENTE_LLUVIA + ')',
  hAxis: {title: 'Fecha', format: 'dd MMM'}, vAxis: {title: 'mm/día'},
  colors: ['1f77b4'], legend: {position: 'none'}
}));


// ---------------------------------------------------------------------
// 9. EXPORTAR TABLAS A GOOGLE DRIVE (pestaña "Tasks" → RUN)
// ---------------------------------------------------------------------
Export.table.toDrive({
  collection: tablaTodo, description: 'serie_agua_rio_sep_oct_' + ANIO,
  folder: 'GEE_tesis2', fileFormat: 'CSV',
  selectors: ['fecha', 'agua_ha', 'pct_claro', 'indice_medio']
});
Export.table.toDrive({
  collection: tablaAnual, description: 'agua_rio_sep_oct_' + ANIOS_COMP[0] + '_' + ANIO,
  folder: 'GEE_tesis2', fileFormat: 'CSV',
  selectors: ['anio', 'mes', 'agua_ha', 'pct_cubierto']
});
Export.table.toDrive({
  collection: tablaLluvia, description: 'lluvia_mensual_' + FUENTE_LLUVIA + '_' + ANIO,
  folder: 'GEE_tesis2', fileFormat: 'CSV',
  selectors: ['nombre', 'promedio_2016_2025_mm', 'anio_2026_mm']
});
